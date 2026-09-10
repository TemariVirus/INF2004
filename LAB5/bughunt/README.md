# BUG HUNT #5 — It failed once, an hour in

> **Guidance level: minimal.** The specification, the defect count, and one
> pointer. No hints, no worked examples, no questions list. You have done four
> of these. Method: [`../../BUGHUNT.md`](../../BUGHUNT.md).

| | |
|---|---|
| **Algorithm** | Ring buffer and moving average, under concurrency |
| **Investigation areas** | **9** — 8 in `bughunt5.c`, 1 in `FreeRTOSConfig.h`; stack exhaustion is build-dependent |
| **Runs on** | Pico W + FreeRTOS-Kernel (the one you set up earlier this lab) |
| **Time** | 90 minutes |
| **Optional practice notes** | `LOGBOOK.md` and the stack high-water table |

---

## The specification

Four tasks. A sensor task samples the RP2040's internal temperature sensor every
**100 ms** and publishes each reading. A moving-average task maintains the mean
of the **last ten** samples. A running-mean task maintains the mean **since
boot**. A print task is the **only** task permitted to call `printf` — the same
rule as the Lab 5 exercise.

Every published sample must reach **both** consumers. The two consumers must not
influence each other's numbers. Nothing may be silently discarded.

None of that currently holds.

---

## The pointer

You cannot read your way to nine concurrency defects, and you cannot `printf`
your way there either — `printf` is one of the things perturbing the system.

**The operating system already knows what is wrong. It has been told not to
mention it.**

Turn the kernel's own reporting on before you do anything else. Look up what
each of these does, decide which ones you want, and switch them on:

```c
configASSERT(x)
configCHECK_FOR_STACK_OVERFLOW
vApplicationStackOverflowHook()
uxTaskGetStackHighWaterMark()
configUSE_MALLOC_FAILED_HOOK
vTaskList()
```

These instruments can expose precondition failures and inadequate stack margins.
Print a table of every
task's stack high-water mark once a second and keep it on screen while you work —
use that table to justify any change in stack allocation. Do not assume an
overflow must occur on your SDK or formatting library.

> `configASSERT` is not a debugging luxury. It is the kernel's contract with you:
> kernel checks written with this macro disappear when it is empty. Enable it
> during development. Whether to retain assertions in production is a separate
> design decision; disabling them is not itself C undefined behaviour.

---

## What you should expect

The program may initially show no temperature readings: a message larger than
the receiver's buffer stays queued, so repeated receives return zero. Compare
the send size and receive capacity first. After that is repaired, later defects
can produce plausible but wrong numbers. Their frequency depends on the build.

Things worth being suspicious about, in no particular order and with no promise
that each maps to exactly one defect:

- A `#define` at the top of the file that nothing reads.
- A synchronisation primitive that is created and never used again.
- Two tasks calling `xMessageBufferReceive()` on the same buffer. Go and read
  what the FreeRTOS documentation says about how many readers a message buffer
  supports. It is a shorter answer than you expect, and it is not a suggestion.
- A `static` variable inside a function that two different tasks call.
- A function whose return value is being ignored, when the whole point of that
  return value is to tell you it failed.
- An average that divides by a constant when it should divide by however many
  samples it actually has.
- A `sizeof` applied to something that is not what you think it is.

---

## Two techniques for this hunt

**Make it frequent.** An intermittent fault is not debuggable. Raise the sample
rate, shrink `RING_LEN`, drop `PRINT_Q_LEN` to 2, run the sensor task at the same
priority as the consumers. You are trying to turn "once an hour" into "twice a
second". Once you can reproduce it on demand, the rest is ordinary work.

**Change one scheduler knob at a time.** Priority, tick rate, stack size, queue
length. Each one, on its own, then back. Record what moves and what does not. A
symptom that moves when you change `configTICK_RATE_HZ` is a timing defect. A
symptom that does not is a logic defect. That single distinction will halve your
search space.

---

## A warning about "fixes"

Several of these defects can be made to *stop happening* without being fixed:

- Give a task more stack without checking whether stack exhaustion caused the fault.
- Slow the sensor task down, and the race gets rarer.
- Make the queue longer, and the drops get less frequent.

Each of those is rule 5 from the method — deleting a symptom. If your logbook
contains a row where the conclusion is "increased the buffer and it went away",
you have not finished; you have made the bug harder for the next person to find.
Say *why* the resource was too small, or find the real defect.

---

## Reflect on your attempt

- `LOGBOOK.md`, at least **nine** defect rows plus your hypothesis trail.
- A stack high-water-mark table for all four tasks, before and after your fixes.
- In the reflection, answer this:

> Pick the defect you would have been least likely to find by reading the source
> code alone. What instrument found it, and what would have happened if this
> firmware had shipped without anyone finding it?

## After your attempt

This is ungraded practice; the logbook and reflection prompts are optional.
Compare your reasoning with the separate [answer guide and corrected source](../../answers/bughunt5/README.md).
