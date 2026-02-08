# I Built an Entire Operating System Just to Say "Hello World"

## A Tale of Over-Engineering, AI Assistance, and Questionable Life Choices

*By KarAshutosh*

---

### The Beginning: A Simple Idea Gone Wrong

It started, as most disasters do, with a simple thought:

> "I should learn how operating systems work."

A reasonable goal. A noble pursuit. What could go wrong?

Everything. Everything could go wrong.

You see, normal people learn about operating systems by reading a textbook. Maybe watching a YouTube video. Perhaps taking a course.

I decided to build one. From scratch. In Rust. With an AI assistant.

To print "Hello World."

**200+ lines of code. 16KB of binary. 11 characters of output.**

This is my story.

---

### Chapter 1: "How Hard Could It Be?"

Famous last words.

I opened my IDE, cracked my knuckles, and typed words that, on hindsight, don't seem to sound too smart:

```
Me: How do I write an OS to say "Hello World"?
```

The AI, bless its silicon heart, didn't even flinch. It just started explaining bootloaders, protected mode, VGA buffers, and something called a "Global Descriptor Table."

I nodded along like I understood. I did not understand.

But here's the thing about AI assistants, they're incredibly patient. They'll explain the same concept 47 different ways until something clicks. Or until you give up. Whichever comes first.

---

### Chapter 2: The Toolchain Tango

Before writing a single line of OS code, I had to set up the development environment. This is where I learned that Rust has approximately 47,000 different ways to install itself, and I had somehow managed to install it wrong.

```
Error: the option `Z` is only accepted on the nightly compiler
```

"But I AM using nightly!" I screamed at my monitor.

Turns out, I had Homebrew's Rust AND rustup's Rust installed. They were fighting for dominance like two cats in a cardboard box.

**Lesson learned:** When `which rustc` returns `/opt/homebrew/bin/rustc` but you expected `~/.cargo/bin/rustc`, you're gonna have a bad time.

The fix? One line:
```bash
export PATH="$HOME/.cargo/bin:$PATH"
```

One hour of debugging. One line of fix. This is programming.

---

### Chapter 3: The Target Specification Saga

To compile Rust for bare metal, you need a "target specification", a JSON file that tells the compiler "hey, there's no operating system here, good luck."

My first attempt:
```json
"target-pointer-width": "32"
```

Rust's response:
```
error: invalid type: string "32", expected u16
```

Apparently, at some point between "a while ago" and "now," Rust decided strings should be integers. Nobody told me. I found out the hard way.

Then came the SSE debacle:
```
error: target feature `sse2` is required by the ABI
```

Translation: "64-bit x86 REQUIRES certain CPU features, and you can't just turn them off because you feel like it."

Solution: Use 32-bit instead. It's 2024 and I'm writing 32-bit code. My micro processors professor would be so proud. Or horrified. Probably horrified.

---

### Chapter 4: The Bootloader Betrayal

I initially tried to use a nice, pre-made bootloader crate. "Why reinvent the wheel?" I thought. "Let's use existing tools!"

The bootloader crate's response:
```
error: could not compile `serde_core` due to 5829 previous errors
```

Five thousand, eight hundred and twenty-nine errors.

From a DEPENDENCY. Of a DEPENDENCY.

At this point, the AI gently suggested: "Perhaps we should write our own bootloader."

50 lines of assembly later, we had a working bootloader. Sometimes the old ways are the best ways. Sometimes you just need to write raw assembly that pokes the CPU directly.

```asm
mov eax, cr0
or eax, 1
mov cr0, eax
; Congratulations, you're now in protected mode
; Please don't mess this up
```

---

### Chapter 5: "Booting from Hard Disk..."

The moment of truth. I built the OS. I ran QEMU. The screen displayed:

```
Booting from Hard Disk...
```

And then... nothing. A blinking cursor. Silence. The void stared back.

This began what I call "The Great Debugging Session of 2024."

**Step 1: Check the boot signature**
```bash
hexdump -C target/os.img | grep "000001f0"
```
Result: `55 aa` . Correct! The BIOS should recognize this as bootable.

**Step 2: Check if the kernel is in the image**
```bash
hexdump -C target/os.img | head -40
```
Result: Kernel code visible at offset 0x200. Good!

**Step 3: Add debug output**
```asm
mov byte [0xb8000], 'P'  ; Write 'P' to screen
```
Result: 'P' appeared! Protected mode works!

**Step 4: Stare at the screen for 20 minutes**

**Step 5: Try random QEMU flags**
```bash
qemu-system-i386 -fda target/os.img
```

IT WORKED.

The difference? `-fda` (floppy disk) vs `-drive` (hard disk). Our bootloader was written for floppy disk geometry. QEMU was treating it as a hard disk.

One flag. ONE FLAG. Hours of debugging.

---

### Chapter 6: The Borrow Checker Strikes Back

Even in bare-metal code, Rust's borrow checker is watching. Always watching.

```rust
self.buffer()[self.row][self.col] = character;
```

```
error[E0503]: cannot use `self.row` because it was mutably borrowed
```

In any other language, this would work fine. In Rust, the compiler said "I see you're trying to borrow `self` mutably for `buffer()` and then also read `self.row`. I'm going to need you to not do that."

The fix involved raw pointers and `write_volatile`. Because when the borrow checker says no, you go around it with `unsafe`. 

(Don't tell the Rust evangelists I said that.)



---

### Chapter 7: What I Actually Built

After all that, here's what the OS does:

1. BIOS loads 512 bytes of bootloader
2. Bootloader loads kernel from disk
3. Bootloader switches CPU to protected mode
4. Kernel writes to VGA memory
5. Infinite loop

That's it. That's the whole OS.

The output:
```
==========================================================================
   _   _      _ _        __        __         _     _ _ 
  | | | | ___| | | ___   \ \      / /__  _ __| | __| | |
  | |_| |/ _ \ | |/ _ \   \ \ /\ / / _ \| '__| |/ _` | |
  |  _  |  __/ | | (_) |   \ V  V / (_) | |  | | (_| |_|
  |_| |_|\___|_|_|\___/     \_/\_/ \___/|_|  |_|\__,_(_)

  By KarAshutosh. From Rust. With love. And zero dependencies on libc.
==========================================================================

  [OK] Booted into 32-bit protected mode
  [OK] VGA text mode: 80x25

  This OS does exactly one thing, and it does it well.

  Now entering infinite loop. As one does.

  Made by KarAshutosh
```

Beautiful. Majestic. Completely unnecessary.

---

### Chapter 8: Learning with AI. The Good Parts

Here's the thing, I couldn't have done this without AI assistance. Not because I'm not smart enough (debatable), but because:

1. **Instant context switching**: "Explain GDT" → explanation. "Now show me the assembly" → code. "Why doesn't this work" → debugging. No tab-switching, no Stack Overflow rabbit holes.

2. **Patient repetition**: I asked "why do we need `write_volatile`" approximately 3 times. The AI explained it 3 times. No judgment. No "I already told you this."

3. **Real-time debugging**: When something broke, we debugged together. "Try hexdump." "Check the boot signature." "Add debug output here." It was like pair programming with someone who never gets tired.

4. **Learning by doing**: Instead of reading about bootloaders, I WROTE one. Instead of studying VGA buffers, I POKED one. The AI guided, I typed, and somehow it worked.

---

### Chapter 9: Learning with AI. The Honest Parts

It wasn't all smooth sailing:

1. **Outdated information**: The AI suggested code that worked on older Rust nightly. Current nightly had breaking changes. We had to adapt.

2. **Dependency hell**: The bootloader crate suggestion was a dead end. Sometimes the AI's "easy path" isn't actually easy.

3. **Still need to understand**: The AI can write code, but if you don't understand WHY, you can't debug. I had to actually learn what protected mode means.

4. **Garbage in, garbage out**: Vague questions get vague answers. "Why doesn't it work" doesn't help. "Why does QEMU show nothing after 'Booting from Hard Disk'" does.

---

### Chapter 10: The Final Tally

**Time spent:** More than I'd like to admit

**Lines of code:**
- Bootloader: ~50 lines of assembly
- Kernel: ~100 lines of Rust
- Build script: ~10 lines of bash
- Config files: ~50 lines of TOML/JSON

**Errors encountered:** At least 15 distinct error types

**Times I wanted to quit:** 3

**Times the AI talked me off the ledge:** 3

**Final binary size:** 16KB

**Characters of output:** ~500 (including ASCII art)

**Was it worth it?** Absolutely.

---

### What I Actually Learned

1. **Computers are magic**: The fact that ANY of this works is a miracle. From power-on to "Hello World," there are approximately 47 billion things that could go wrong.

2. **Abstractions are gifts**: We take `println!("Hello")` for granted. Under the hood, there's a VGA buffer, a font renderer, memory-mapped I/O, and decades of engineering.

3. **Rust is strict for a reason**: Every time the borrow checker yelled at me, it was preventing a potential bug. Annoying? Yes. Correct? Also yes.

4. **Over-engineering is fun**: Could I have just used `println!` in a normal Rust program? Yes. Would I have learned anything? No. Sometimes the journey IS the destination.

---

### Conclusion: Would I Do It Again?

Yes. But differently.

Next time, I might try:
- 64-bit mode (with paging!)
- Keyboard input
- A simple shell
- Maybe even... two programs running at once?

But for now, I have an operating system that says "Hello World." It boots on real hardware (probably). It's written in Rust (definitely). And it was built with AI assistance (absolutely).

Is it useful? No.
Is it impressive? Questionable.
Did I learn something? Absolutely.

And isn't that what matters?

---

*The full source code, tutorial, and my tears are available on [GitHub](https://github.com/KarAshutosh/The-HelloWorld-Rust-OS).*

*Made by KarAshutosh, with assistance from an AI that never once asked "why would you want to do this?"*
