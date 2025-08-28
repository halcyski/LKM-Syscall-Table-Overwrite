
- Kernel text/rodata are read-only. 
- On x86-64 the CR0.WPP bit enforces write protection even in ring 0.
- In order to modify ``sys_call_table`` you must temporarily clear WP, write one pointer, then restore WP. Disable preemption to keep the critical section atomic on the current CPU.  


This can be achieved like so:
```c
#include <asm/processor-flags.h>  // X86_CR0_WP
#include <asm/special_insns.h>    // read_cr0(), write_cr0()

static unsigned long saved_cr0;

static void sct_write_begin(void) {
	saved_cr0 = read_cr0();
	preempt_disable();
	barrier();
	write_cr0(savedcr0 & ~X86_CR0_WP);
	barrier();
}

static void sct_write_end(void) {
	barrier();
	write_cr0(saved_cr0);
	barrier();
	preempt_enable();
}
```
Breakdown: 
``read_cr0`` will read the CPU's CR0 control register and stash it to be restored later.

``preempt_disable()`` will turn off task preemption on the current CPU so that he schedular can't migrate our process to a different core mid-operation. 
- CR0 is per-CPU meaning that if the schedular moves us to another CPU, we could miss our chance to revert CR0 and/or assume the state of WP in the new CPU
- Disabling task preemption will prevent this

``barrier()`` is a compiler barrier that prevents the compiler from reordering memory operations across critical points.
- The CR0 write is a serial but ``barrier()`` keeps the compiler "honest" and to document intent. 

``write_cr0(saved_cr0 & ~X86_CR0_WP)`` clears the Write Protect bit (WP) in CR0 on this CPU only. With this bit disabled (0), any ring 0 writes to pages marked as read-only (like ``sys_call_table`` inside of ``.rodata``) become temporarily enabled.

## A note on concurrency 
- Per-CPU semantics
	- CR0 is per-CPU. Clearing WP only affections the current core.
	- ``preempt_disbale()`` guarantees our write stays on this cpu
- Interrupts and races
	- We do not disable interrupts during our WP window 
	- An interrupt *could* occur while WP is cleared
	- The window that is created occurs in such a small portion of time but it can be made stronger by changing the signature of our ``sct_write`` functions and including two extra functions:
```c
static void sct_write_begin_irqsave(unsigned long *flags)
{
    *flags = 0;
    preempt_disable();
    barrier();
    local_irq_save(*flags);                    // mask local IRQs
    barrier();
    write_cr0(read_cr0() & ~X86_CR0_WP);
    barrier();
}

static void sct_write_end_irqrestore(unsigned long flags)
{
    barrier();
    write_cr0(read_cr0() | X86_CR0_WP);       // restore WP; read_cr0() ok too
    barrier();
    local_irq_restore(flags);                 // unmask IRQs
    barrier();
    preempt_enable();
}
```

- Atomicity of the pointer swap
	- A naturally aligned 64-bit pointer (on x86-64) is atomic 