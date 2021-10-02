## Super Mario Land: Starman Crash Explanation/Fix

### Introduction
In the Game Boy game Super Mario Land, there is a non-zero chance of
the console locking up after the main level music restarts when the
invincibility of the star runs out. This is due to a small bug found
in both versions of the original cartridge.

### Bank switching
Like many other Game Boy games, SML makes use of ROM bank switching.
This way, multiple different areas of the ROM can be loaded into the
memory region `$4000`–`$7fff`, thus allowing games to be bigger in size.
Switching banks works by writing the desired bank number to any address
in the range `$2000`–`$3fff`, which communicates with the mapper chip,
but only in one direction, as this is write-only.

This is why most bank-switching games that utilize interrupts save the bank number
into a variable before they actually switch to it.
If the program flow were linear, this wouldn't be needed, but interrupts
(like the name implies) interrupt the program flow and if the code executed
inside the interrupt routine also wants to make use of bank-switching, it
needs to know what the current bank actually is, so it can come back to it at the end.

In SML, the current rom bank normally gets saved at memory location `$fffd`, so
switching to bank 2, for example, and calling a function inside this bank would look like this:
```
  ld   a, 2
  ld   [$fffd], a
  ld   [$2000], a
  call $4567
```
This loads the value `2` into the register `A`, and stores this value
in the memory location for the current bank and then tells the mapper to actually switch to it.
Now, if an interrupt would occur inside function `$02:$4567`, it would know the program
currently is switched to bank 2, and it can switch back to bank 2 after its own
bank-switching.

### The actual bug
Now, let's look at the routine that starts the main level music in SML v1.0:
```
  0791:  ld   a, 3
  0793:  ld   [$2000], a
  0796:  call $7ff3
  0799:  ld   a, [$fffd]
  079b:  ld   [$2000], a
```
This code only tells the mapper to switch to bank `3` without saving it in `$fffd`.
So when an interrupt occurs inside function `$03:$7ff3`, the interrupt routine
cannot know that we are in bank `3` (remember, the current bank is write-only),
and so switches to the wrong bank at the end, leading to an area of the ROM that
isn't code, and normally landing at an illegal instruction, which locks up the
whole console, requiring a hard reset. There are not very many instructions behind
the function `$03:$7ff3`, which is why it is relatively unlikely that an interrupt
would fire during it, making the bug rather infrequent.

But why didn't they save the bank in `$fffd` like they do everywhere else?
For one, it already holds the current bank, so they'd have to
save it in yet another variable.
Maybe the developers assumed that this is not needed, because interrupts are disabled
while loading the level. But they are very much enabled during the level
(and thus, while the music restarts after an invincibility star).

### Possible fix
If we were to assume there is a chance
that the function can get called from a specific bank (like the code hints at),
we could just save it in another memory location, then overwrite `$fffd` with
that value again after the function, so it would look something like this:
```
  ld   a, [$fffd]
  ld   [OLD_ROM_BANK], a
  ld   a, 3
  ld   [$fffd], a
  ld   [$2000], a
  call $7ff3
  ld   a, [OLD_ROM_BANK]
  ld   [$fffd], a
  ld   [$2000], a
```

### Patches

Attached are two patches:
1. The file `sml1_starman_crash_v10.bsdiff` creates a ROM from SMLv1.0 which loads the checkpoint next to the star in 2–1 and adds a busy loop during the execution of function `$03:$7ff3`, which guarantees an interrupt during it, triggering the crash after the end of the star song.
2. If the ROM created in step 1 gets patched with the second file `sml1_starman_fix_v10.bsdiff` (which essentially does the same thing as the proposed fix above), the game doesn't crash anymore after the invincibility runs out.
