# My Notes — Pintos Lab 1, Task 3: Priority Donation

These are my own notes on how I implemented priority donation, written so
future-me remembers *why* each piece exists, not just *what* it does. This
builds directly on Task 2 (basic priority scheduling), which already gave me:

- `thread_higher_priority()` — a comparator that sorts any thread list from
  highest to lowest priority.
- `thread_preempt()` — yields the CPU whenever a higher-priority thread
  becomes ready.
- Sorted `ready_list` and sorted semaphore/condvar waiter lists.

Task 3 is about fixing **priority inversion**: a low-priority thread holding
a lock that a high-priority thread needs can get stuck behind medium-priority
threads that have nothing to do with the lock. The fix is **donation**: the
high-priority thread temporarily lends its priority to whoever is blocking
it, so that thread can finish faster and hand the lock back.

---

## 1. What I added to the data structures

### `struct thread` (in `thread.h`)

```c
int base_priority;              /* Priority without any donations. */
struct list locks_held;         /* Locks this thread currently holds. */
struct lock *wait_on_lock;      /* Lock this thread is blocked on, or NULL. */
```

**Why I needed these:**

- `priority` (already existed from Task 2) had to become the *effective*
  priority — the one the scheduler actually looks at. But I still need to
  remember the thread's *real* priority (what it was set to, ignoring
  donations), so I can go back to it once a lock is released. That's
  `base_priority`.
- `wait_on_lock` is how I walk *up* a donation chain. If H waits on a lock
  held by M, and M waits on a lock held by L, I need to be able to hop
  from H → M → L by following `wait_on_lock` repeatedly.
- `locks_held` is how I figure out, when recalculating a thread's priority,
  *all* the sources of donation it currently has. A thread could be
  holding more than one lock, each with its own waiters.

### `struct lock` (in `synch.h`)

```c
struct list_elem elem;   /* For thread->locks_held list. */
```

**Why:** so a lock can be an element in the `locks_held` list I just added
to `struct thread`.

---

## 2. The core functions I wrote (`thread.c`)

### 2.1 Recomputing effective priority — `thread_update_priority()`

```c
void
thread_update_priority (struct thread *t)
{
  enum intr_level old_level = intr_disable ();
  int max_priority = t->base_priority;
  struct list_elem *e;

  for (e = list_begin (&t->locks_held); e != list_end (&t->locks_held);
       e = list_next (e))
    {
      struct lock *l = list_entry (e, struct lock, elem);
      if (!list_empty (&l->semaphore.waiters))
        {
          struct thread *top =
              list_entry (list_front (&l->semaphore.waiters), struct thread, elem);
          if (top->priority > max_priority)
            max_priority = top->priority;
        }
    }

  t->priority = max_priority;
  intr_set_level (old_level);
}
```

**My reasoning:** a thread's effective priority should always be the
*highest* of: its own base priority, or the priority of the strongest
thread waiting on any lock it holds. I loop over every lock in
`locks_held`, and for each one look at `list_front()` of that lock's
waiters — which is already the highest-priority waiter, because Task 2
keeps that list sorted.

This function is the single source of truth for "what should my priority
be right now" — I call it any time something that could affect donation
changes (releasing a lock, changing my own base priority).

### 2.2 Propagating donation up a chain — `thread_donate_priority()`

```c
void
thread_donate_priority (struct thread *t)
{
  enum intr_level old_level = intr_disable ();
  int depth = 0;

  while (t->wait_on_lock != NULL && depth < 8)
    {
      struct thread *holder = t->wait_on_lock->holder;
      if (holder == NULL || holder->priority >= t->priority)
        break;

      holder->priority = t->priority;
      thread_reposition (holder);

      t = holder;
      depth++;
    }

  intr_set_level (old_level);
}
```

**My reasoning:** I follow `wait_on_lock` pointers up the chain, boosting
each holder's priority as I go, and stop as soon as:
- I hit a thread that isn't waiting on anything (`wait_on_lock == NULL`), or
- The next holder already has priority ≥ what I'd be donating (no point
  continuing — nothing downstream needs a boost either).

The `depth < 8` cap is just a safety net matching what the official Pintos
docs suggest — a reasonable bound on nested donation depth so a
pathological chain can't loop forever. I never expect to actually hit this
limit with correct chains; it's a guard, not a real constraint.

### 2.3 Fixing stale list position — `thread_reposition()`

```c
static void
thread_reposition (struct thread *t)
{
  if (t->status == THREAD_READY)
    {
      list_remove (&t->elem);
      list_insert_ordered (&ready_list, &t->elem, thread_higher_priority, NULL);
    }
}
```

**My reasoning:** `ready_list` is kept sorted so the scheduler can just grab
`list_front()`. But if a thread's priority changes *after* it's already
sitting in `ready_list`, its position goes stale — it might now belong
further up the list. This function pulls it out and reinserts it at the
correct spot.

I originally also had a branch here for `THREAD_BLOCKED && wait_on_lock !=
NULL`, meant to keep a lock's waiter list sorted too. **I removed that
branch** once I fixed `sema_up()` to scan for the max-priority waiter
instead of trusting the list to stay sorted (see §4) — that made this extra
branch redundant. Keeping only the `THREAD_READY` case here is the simplest
version that's actually needed.

### 2.4 `thread_set_priority()` — routing through the same logic

```c
void
thread_set_priority (int new_priority) 
{
  struct thread *cur = thread_current ();
  cur->base_priority = new_priority;
  thread_update_priority (cur);
  thread_preempt ();
}
```

**My reasoning:** I only ever change `base_priority` directly — never
`priority` directly — and let `thread_update_priority()` decide what the
*effective* priority should be. This is what makes `priority-donate-lower`
work correctly: if I'm currently boosted above `new_priority` because I'm
holding a lock someone higher-priority is waiting on, my effective
priority correctly stays at the donated level. I only actually drop once
the lock is released and there's no more donation propping me up.

---

## 3. Changes in `synch.c`

### 3.1 `lock_acquire()` — donate before blocking

```c
void
lock_acquire (struct lock *lock)
{
  struct thread *cur = thread_current ();

  ASSERT (lock != NULL);
  ASSERT (!intr_context ());
  ASSERT (!lock_held_by_current_thread (lock));

  if (lock->holder != NULL)
    {
      enum intr_level old_level = intr_disable ();
      cur->wait_on_lock = lock;
      intr_set_level (old_level);
      thread_donate_priority (cur);
    }

  sema_down (&lock->semaphore);

  cur->wait_on_lock = NULL;
  lock->holder = cur;
  list_push_back (&cur->locks_held, &lock->elem);
}
```

**My reasoning, step by step:**
- I only bother donating if the lock is actually held by someone
  (`lock->holder != NULL`). If it's free, there's nothing to donate to.
- I set `wait_on_lock` **before** calling `thread_donate_priority`, because
  that function starts its walk from `wait_on_lock` — it needs to already
  be set to know where to start.
- After `sema_down()` returns (meaning I successfully got the lock), I
  clear `wait_on_lock` back to `NULL` — I'm not waiting on it anymore, I
  own it — and add it to my `locks_held` list.

### 3.2 `lock_release()` — undo the donation, in the right order

```c
void
lock_release (struct lock *lock) 
{
  ASSERT (lock != NULL);
  ASSERT (lock_held_by_current_thread (lock));

  list_remove (&lock->elem);
  lock->holder = NULL;
  thread_update_priority (thread_current ());

  sema_up (&lock->semaphore);
}
```

**My reasoning about the ordering — this one took me a moment to get
right:** I remove the lock from my `locks_held` list and recompute my own
priority **before** calling `sema_up()`. That matters because `sema_up()`
internally calls `thread_preempt()` (from Task 2), which compares the
*current* thread's priority against the highest-priority ready thread. If
I called `sema_update_priority` *after* `sema_up`, the preemption check
would compare against my old, still-boosted priority — and I might
incorrectly *not* yield when I actually should have dropped below someone
else.

---

## 4. The bugs I actually hit, and how I found them

This is the part I most want to remember for next time — the *process* of
tracking these down mattered more than the final code.

### Bug 1: `static` function declared in a shared header

**Symptom:** build warnings like `'thread_higher_priority' declared
'static' but never defined` in almost every `.c` file, then a real linker
error: `undefined reference to 'thread_higher_priority'` in `synch.o`.

**What was wrong:** I had declared `thread_higher_priority` as `static`
inside `thread.h`. Since headers get `#include`d — i.e. textually pasted —
into every file that includes them, a `static` (internal-linkage)
declaration in a header doesn't create *one* shared function — it creates
a **separate, private declaration in every single file**. Only `thread.c`
had the real function body, so every other file's copy was an empty,
unlinkable stub. `synch.c` was the one file that actually *called* it, so
that's where the link failed.

**Fix:** remove `static` from both the header declaration and the
`thread.c` definition, so it has proper external linkage and every file
links against the one real function in `thread.o`.

**Lesson:** `static` in a header is basically always wrong if more than one
`.c` file needs to call the function. Only mark something `static` if it's
declared *and* defined in the same file and used only there.

### Bug 2: calling a `static` helper before declaring it

**Symptom:**
```
implicit declaration of function 'thread_reposition'
conflicting types for 'thread_reposition'
error: static declaration of 'thread_reposition' follows non-static declaration
```

**What was wrong:** `thread_donate_priority()` called `thread_reposition()`
before the compiler had seen any declaration for it anywhere in the file.
The compiler implicitly assumed a signature (and no `static`), then later
hit the real `static` definition further down the file and complained
about the mismatch.

**Fix:** added a forward declaration near the top of `thread.c`, alongside
the file's other `static` helper forward-declarations:
```c
static void thread_reposition (struct thread *t);
```

**Lesson:** any time a function is used before its definition appears
later in the same file, it needs a forward declaration up top — this is
just plain C, not Pintos-specific, but it's easy to trip over when adding
new helper functions into an existing large file.

### Bug 3: donation list mutations weren't interrupt-safe

**Symptom:** intermittent hangs — threads would just vanish from
scheduling rather than run in the wrong order. This is a much scarier bug
than a logic mistake because it's *timing-dependent* — it doesn't fail the
same way every run.

**What was wrong:** every other function in `synch.c` (`sema_down`,
`sema_up`, `lock_acquire`, `lock_release`) wraps its list mutations in
`intr_disable()` / `intr_set_level()`. That's the only synchronization
primitive available at this level in Pintos — there's no other way to make
a list operation atomic with respect to the timer interrupt. My new
`thread_donate_priority()` and `thread_update_priority()` did **not** do
this. `thread_reposition()` in particular does a `list_remove()` followed
by a separate `list_insert_ordered()` — if a timer tick landed in between
those two calls, the timer interrupt handler could touch `ready_list` at
exactly the moment a thread's `elem` was detached from one list and not
yet in another, corrupting the list.

**Fix:** wrapped the bodies of `thread_donate_priority()` and
`thread_update_priority()` in `intr_disable()` / `intr_set_level()`, same
pattern as everywhere else in `synch.c`.

**Lesson:** any function that touches a shared list (`ready_list`, a
semaphore's `waiters`) needs interrupts disabled around the *whole*
multi-step mutation, not just individual list calls. This is easy to
forget when writing new functions that "feel like" pure logic rather than
list surgery.

### Bug 4: donation didn't reach a thread sleeping on a plain semaphore

This was the trickiest one — `priority-donate-sema` kept failing even
after fixing the interrupt-safety issue above.

**Symptom:** test output stopped after `Thread L acquired lock.` and
`Thread M finished.` — threads `H` and `L` never printed anything else and
the test hung/failed, even though `M` (an unrelated thread) ran fine.

**What was actually happening, step by step:**
1. `L` (priority 32) acquires a lock, then calls `sema_down()` on a
   **separate, plain semaphore** — not the lock's own semaphore — and
   blocks. `L` goes into that semaphore's `waiters` list with its priority
   at the time: 32.
2. `M` (priority 34) also calls `sema_down()` on that same semaphore and
   blocks. My sorted-insert put the waiters list as `[M(34), L(32)]`.
3. `H` (priority 36) tries to acquire the lock `L` holds. `H` blocks and
   donates its priority to `L` — `L`'s priority jumps to 36.
4. My `thread_donate_priority()` called `thread_reposition(L)` to fix up
   wherever `L` was sitting — but `L` was `THREAD_BLOCKED` with
   `wait_on_lock == NULL`, because `L` isn't blocked on a *lock* at all,
   it's blocked on that plain semaphore. My `thread_reposition()` only knew
   how to handle `THREAD_READY` and `THREAD_BLOCKED` *with* `wait_on_lock`
   set — neither matched, so nothing happened. The semaphore's `waiters`
   list stayed stale: `[M(34), L(32→36)]`, even though `L` now deserved to
   be first.
5. When the main thread called `sema_up()` on that semaphore, my old code
   did `list_pop_front()` — which grabbed `M`, the *wrong* thread, purely
   because the list order was never corrected.
6. `M` woke up and finished. `L` was never woken again (only one
   `sema_up()` call was made), so it never printed anything further, and
   `H` stayed stuck forever waiting for a lock `L` never got to release.

**Why my original design missed this:** I had only taught
`thread_reposition()` about two situations — a thread sitting in
`ready_list`, or a thread blocked specifically on a *lock's* semaphore.
I hadn't accounted for a thread being blocked on an arbitrary, unrelated
semaphore that has nothing to do with locks or donation chains. The
official Pintos spec even says *"you need not implement priority donation
for the other synchronization constructs [besides locks]"* — but that
doesn't mean priority ordering can go stale for them; it just means they
don't need to *donate*. Wake-order still has to reflect current priority.

**Fix — the one that actually worked:** instead of trying to track every
possible list a thread could be sleeping in (which felt like it was
heading toward over-engineering), I changed `sema_up()` itself to **scan
for the highest-priority waiter at wake time**, rather than trusting
insertion-time sorting to still be valid:

```c
void
sema_up (struct semaphore *sema) 
{
  enum intr_level old_level;

  ASSERT (sema != NULL);

  old_level = intr_disable ();
  if (!list_empty (&sema->waiters)) 
    {
      struct list_elem *e, *max_e = list_begin (&sema->waiters);

      for (e = list_next (max_e); e != list_end (&sema->waiters); e = list_next (e))
        {
          struct thread *t = list_entry (e, struct thread, elem);
          struct thread *max_t = list_entry (max_e, struct thread, elem);
          if (t->priority > max_t->priority)
            max_e = e;
        }

      list_remove (max_e);
      thread_unblock (list_entry (max_e, struct thread, elem));
    }
  sema->value++;
  intr_set_level (old_level);

  thread_preempt ();
}
```

This makes staleness a non-issue entirely — it no longer matters *when* or
*why* a waiting thread's priority changed while it was asleep; the correct
thread is found no matter what. Once this was in place, insertion order in
`sema_down()` didn't need to be priority-sorted anymore either (a plain
`list_push_back` works fine), and the extra `THREAD_BLOCKED` branch in
`thread_reposition()` became dead weight, so I removed it — leaving that
function down to just the `THREAD_READY` case shown in §2.3.

**Lesson — the big one:** donation can change a thread's priority *after*
it's already asleep somewhere my donation code doesn't know about or care
about. Rather than trying to chase down and special-case every possible
list a thread might be sitting in, it's simpler and more robust to make
the *wake-up* logic itself resilient to staleness by scanning for the true
maximum at the moment it matters. Sorting at insertion time is an
optimization; scanning at wake time is the correctness guarantee.

---

## 5. Requirement → implementation map

| Requirement | Where |
|---|---|
| Waiting thread donates priority to lock holder | `lock_acquire()` → `thread_donate_priority()` |
| Donation propagates through nested locks | `while` loop over `wait_on_lock` chain in `thread_donate_priority()` |
| Multiple donors → holder gets the max | `thread_update_priority()` scans all held locks' waiter fronts |
| Priority correctly drops back after releasing a lock | `lock_release()` → `thread_update_priority()`, called *before* `sema_up()` |
| Manually lowering priority doesn't undercut an active donation | `thread_set_priority()` → `thread_update_priority()` (max of base vs. donations) |
| Highest-priority waiter always wakes first, even if priority changed while asleep | `sema_up()` scans for max priority at wake time |
| List position doesn't go stale after a priority boost | `thread_reposition()` (only needed for `ready_list` now) |

---

## 6. Tests and results

```bash
cd pintos/src/threads
make clean && make
make check
```

Individually:
```bash
make build/tests/threads/priority-donate-one.result
make build/tests/threads/priority-donate-multiple.result
make build/tests/threads/priority-donate-multiple2.result
make build/tests/threads/priority-donate-nest.result
make build/tests/threads/priority-donate-chain.result
make build/tests/threads/priority-donate-lower.result
make build/tests/threads/priority-donate-sema.result
```

All passing as of this writing, alongside the Task 2 tests
(`priority-change`, `priority-fifo`, `priority-preempt`, `priority-sema`,
`priority-condvar`).

---

## 7. What I deliberately didn't do

- Didn't implement donation *through* condition variables or plain
  semaphores — only through locks, matching the spec. The `sema_up()` scan
  fix handles wake-order correctly for the non-lock cases without needing
  actual donation logic there.
- Didn't try to make `thread_reposition()` generic enough to handle every
  possible list a thread could be in. Scanning at wake time in `sema_up()`
  turned out to be the simpler and more robust answer than tracking more
  state.
- Didn't touch the MLFQS scheduler — that's a separate, later part of the
  assignment and priority donation is explicitly not required to interact
  with it.
