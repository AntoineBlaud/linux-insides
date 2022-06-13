# LockDown

#### Putting a leash on the root account

The new feature's primary function will be to strengthen the divide between userland processes and kernel code by preventing even the root account from interacting with kernel code -- something that it's been able to do, by design, until now.

When enabled, the new "lockdown" feature will restrict some kernel functionality, even for the root user, making it harder for compromised root accounts to compromise the rest of the OS.

"The lockdown module is intended to allow for kernels to be locked down early in \[the] boot \[process]," said Matthew Garrett, the Google engineer who proposed the feature a few years back.

"When enabled, various pieces of kernel functionality are restricted," said Linus Torvalds, Linux kernel creator, and the one who put the [final stamp of approval](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=aefcf2f4b58155d27340ba5f9ddbe9513da8286d) on the module yesterday.
