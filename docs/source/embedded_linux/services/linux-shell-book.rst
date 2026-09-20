Bash workload cookbook and examples
===================================

.. rubric:: What you will learn

This page is a small **Bash workload cookbook**: one toolkit section plus
**3 copy-paste workload examples**, from gentle to aggressive, from
unbounded to time-bounded. Rewritten from
``linux_embedded_study_notes.rst`` for easier reading and reuse.

.. contents:: On this page
   :local:
   :depth: 2

.. list-table:: The 3 examples at a glance
   :header-rows: 1
   :widths: 12 28 30 30

   * - #
     - Example
     - What it stresses
     - How it stops
   * - 1
     - Memory + CPU loop (``/dev/shm``)
     - RAM read + CPU counting
     - Never (``while true`` + ``sleep 0.2``)
   * - 2
     - Parallel CPU burner (4 jobs)
     - 4 cores at 100%
     - Never (``wait`` on 4 infinite jobs)
   * - 3
     - Bounded workload (``timeout``)
     - Short controlled CPU burst
     - Auto-stops after 0.5 s

.. tip::

   Run Example 1 to observe, Example 2 to saturate, Example 3 to measure.
   Stop any infinite example with ``Ctrl+C`` or ``kill``.

.. rubric:: Toolkit: redirection, dd, head, while, numeric tests

.. list-table:: Redirection cheat sheet
   :header-rows: 1

   * - Command
     - Keeps on screen
     - Discards
   * - ``command >/dev/null``
     - stderr
     - stdout
   * - ``command 2>/dev/null``
     - stdout
     - stderr
   * - ``command >/dev/null 2>&1``
     - nothing
     - stdout + stderr

.. code-block:: bash

   # ``2`` = stderr, ``>`` = redirect, ``/dev/null`` = black hole.
   command 2>/dev/null
   ls /does-not-exist 2>/dev/null
   echo $?   # message hidden, but exit code can still be non-zero

.. note::

   ``2>/dev/null`` only hides the error message. The command can still
   return a non-zero exit status in ``$?``.

.. code-block:: bash

   # dd: copy/convert raw bytes and devices.
   dd if=<input> of=<output> [options]
   dd if=/dev/sdb of=disk.img bs=4M status=progress
   dd if=/dev/zero of=test.img bs=1M count=100
   dd if=/dev/urandom of=random.bin bs=1M count=10

   # head: first lines (-n) or bytes (-c); read -r keeps backslashes raw.
   head file.txt
   head -n 20 file.txt
   head -c 100 file.bin
   dmesg | head -n 20
   head -n 5 file.txt | while read -r line; do
       echo "$line"
   done

.. warning::

   Check ``dd of=`` carefully. Writing to the wrong disk destroys data.

Bash numeric tests compare two integers inside ``[ ... ]``. The letters
are abbreviations of plain English words:

.. list-table:: Numeric test abbreviations
   :header-rows: 1
   :widths: 12 28 30 30

   * - Test
     - Full English
     - C equivalent
     - Example
   * - ``-eq``
     - equal
     - ``==``
     - ``[ $i -eq 5 ]`` means ``i == 5``
   * - ``-ne``
     - not equal
     - ``!=``
     - ``[ $i -ne 5 ]`` means ``i != 5``
   * - ``-lt``
     - less than
     - ``<``
     - ``[ $i -lt 500000 ]`` means ``i < 500000``
   * - ``-le``
     - less than or equal
     - ``<=``
     - ``[ $i -le 5 ]`` means ``i <= 5``
   * - ``-gt``
     - greater than
     - ``>``
     - ``[ $i -gt 5 ]`` means ``i > 5``
   * - ``-ge``
     - greater than or equal
     - ``>=``
     - ``[ $i -ge 5 ]`` means ``i >= 5``

.. tip::

   Read ``-e`` as **equal**, ``-n`` as **not**, ``-l`` as **less**,
   ``-g`` as **greater**, ``-t`` as **than**; a trailing ``-e`` adds
   **or equal**. So ``-lt`` = less-than, ``-le`` = less-or-equal,
   ``-gt`` = greater-than, ``-ge`` = greater-or-equal.

.. code-block:: bash

   count=0
   while [ $count -lt 5 ]; do   # count < 5 in C
       echo "count = $count"
       count=$((count + 1))
   done

.. rubric:: Example 1 — Gentle memory + CPU loop (/dev/shm)

:Duration: infinite (``while true`` + ``sleep 0.2``).
:Load: light RAM reads + light CPU counting.
:Use when: you want a safe observable background load.

.. code-block:: bash

   dd if=/dev/urandom of=/dev/shm/browser_mem bs=1M count=100 2>/dev/null

   while true; do
       head -c 10M /dev/shm/browser_mem > /dev/null
       i=0; while [ $i -lt 500000 ]; do i=$((i+1)); done
       sleep 0.2
   done

Steps: create 100 MiB in ``/dev/shm``, read the first 10 MiB each loop,
discard it to ``/dev/null``, run a CPU counting loop, sleep 0.2 s, repeat.
``/dev/shm`` is normally a RAM-backed tmpfs for temporary memory-backed
data without persistent disk storage.

.. caution::

   Treat this as a simple workload simulation, not an accurate
   browser-memory model.

.. rubric:: Example 2 — Parallel CPU burner (4 background jobs)

:Duration: infinite (4 jobs blocked in ``wait``).
:Load: heavy, up to 4 busy cores.
:Use when: you want to saturate CPU and watch scheduling/temperature.

This ``for`` loop creates 4 background processes, each running an
infinite CPU-counting loop. The trailing ``&`` means: run this task
in the background, so the shell does not wait for it and continues
with the next iteration immediately. The final ``wait`` makes the
parent shell wait for all 4 background jobs instead of exiting.

.. code-block:: bash

   for i in 1 2 3 4; do
       (
           while true; do
               i=0
               while [ $i -lt 1000000 ]; do
                   i=$((i+1))
               done
           done
       ) &
   done

   wait

How it works:

- ``for i in 1 2 3 4`` -- repeat the body 4 times.
- ``( ... )`` -- run the body in a subshell, so each iteration gets
  its own isolated ``i`` counter.
- Inner ``while [ $i -lt 1000000 ]`` -- count from 0 to 1000000
  (``i < 1000000`` in C), purely to burn CPU.
- Outer ``while true`` -- repeat the counting forever.
- ``&`` -- put each subshell in the background; without it the loop
  would block on the first iteration and never start jobs 2-4.
- ``wait`` -- block the parent shell until all background jobs finish
  (here: effectively forever, until you stop them with ``Ctrl+C`` or
  ``kill`` / ``kill %1``).

.. tip::

   Check them with ``jobs`` or ``ps``; stop everything with ``kill %1 %2 %3 %4``
   or ``pkill -f <script-name>``.

.. rubric:: Example 3 — Bounded workload (timeout 0.5 s)

:Duration: at most 0.5 seconds, auto-stops.
:Load: one short controlled CPU burst (100,000 increments).
:Use when: you want a measurable workload without manual ``kill``.

Read it from outside in:

.. code-block:: text

   timeout 0.5
       |
       v
   sh -c '...'
       |
       v
   shell code: i = 0 -> while i < 100000 -> i = i + 1

.. rubric:: 3.1. sh -c runs a one-shot shell

.. code-block:: bash

   sh -c 'i=0; while [ $i -lt 100000 ]; do i=$((i+1)); done'

``sh`` runs a shell, and ``-c`` means: execute the command given as a
string. So ``sh -c 'echo hello'`` asks a shell to run ``echo hello``.
Here the shell runs:

.. code-block:: bash

   i=0
   while [ $i -lt 100000 ]; do
       i=$((i+1))
   done

.. rubric:: 3.2. The counting loop has an end

This is the CPU workload:

.. code-block:: text

   i = 0 -> i < 100000? --yes--> i++ -> ... -> i = 100000 -> STOP

Unlike ``while true``, it has an end point: it stops when
``100000 < 100000`` becomes false.

.. rubric:: 3.3. timeout sets the time box

This is the important part:

.. code-block:: bash

   timeout 0.5 command

means: run ``command`` for at most 0.5 seconds. If the command finishes
earlier, ``timeout`` exits with it. If it is still running at 0.5 s,
``timeout`` asks the process to terminate:

.. code-block:: text

   0.0s       0.5s
    |           |
    v           v
   command -----X
                |
             terminated

.. rubric:: 3.4. Infinite versus bounded: choose your pattern

.. code-block:: bash

   timeout 0.5 sh -c 'i=0; while [ $i -lt 100000 ]; do i=$((i+1)); done'

Translation: run a shell that increments ``i`` 100,000 times, but allow
at most 0.5 seconds of runtime.

Compare with Example 2:

.. code-block:: bash

   while true; do
       i=0
       while [ $i -lt 1000000 ]; do
           i=$((i+1))
       done
   done

.. list-table:: Which example should you run?
   :header-rows: 1

   * - If you want...
     - Run
   * - A safe always-on background load
     - Example 1
   * - Maximum parallel CPU pressure
     - Example 2
   * - A short auto-stopping measurement
     - Example 3

That is an infinite CPU workload, while the ``timeout 0.5 ...`` form is
a CPU workload of at most 0.5 seconds. This is a useful pattern when you
want a workload with a controlled duration instead of writing your own
stop logic for ``while true``.

.. note::

   ``0.5`` is a time limit, not a guarantee of running exactly 0.5 s.
   If the 100,000 iterations finish in 0.05 s, the command ends then.

.. rubric:: References

- `GNU Bash Manual <https://www.gnu.org/software/bash/manual/>`_
- `Linux man pages <https://man7.org/linux/man-pages/>`_
