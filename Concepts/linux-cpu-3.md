# The initcall mechanism

## Introduction

As you may understand from the title, this part will cover an interesting and important concept in the Linux kernel which is called `initcall`. We already saw definitions like these:

```
early_param("debug", debug_kernel);
```

or

```
arch_initcall(init_pit_clocksource);
```

in some parts of the Linux kernel. Before we will see how this mechanism is implemented in the Linux kernel, we must know actually what is it and how the Linux kernel uses it. Definitions like these represent a [callback](https://en.wikipedia.org/wiki/Callback\_\(computer\_programming\)) function which is called during Linux kernel initialization. Actually the main point of the `initcall` mechanism is to determine correct order of the built-in modules and subsystems initialization. For example let's look at the following function:

```
static int __init nmi_warning_debugfs(void)
{
    debugfs_create_u64("nmi_longest_ns", 0644,
                       arch_debugfs_dir, &nmi_longest_ns);
    return 0;
}
fs_initcall(nmi_warning_debugfs);
```

from the [arch/x86/kernel/nmi.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/nmi.c) source code file. As we may see it just creates the `nmi_longest_ns` [debugfs](https://en.wikipedia.org/wiki/Debugfs) file in the `arch_debugfs_dir` directory. Actually, this `debugfs` file may be created only after the `arch_debugfs_dir` will be created. Creation of this directory occurs during the architecture-specific initialization of the Linux kernel. Actually this directory will be created in the `arch_kdebugfs_init` function from the [arch/x86/kernel/kdebugfs.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/kdebugfs.c) source code file. Note that the `arch_kdebugfs_init` function is marked as `initcall` too:

```
arch_initcall(arch_kdebugfs_init);
```

The Linux kernel calls all architecture-specific `initcalls` before the `fs` related `initcalls`. So, our `nmi_longest_ns` file will be created only after the `arch_kdebugfs_dir` directory will be created. Actually, the Linux kernel provides eight levels of main `initcalls`:

* `early`;
* `core`;
* `postcore`;
* `arch`;
* `subsys`;
* `fs`;
* `device`;
* `late`.

All of their names are represented by the `initcall_level_names` array which is defined in the [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) source code file:

```
static char *initcall_level_names[] __initdata = {
	"early",
	"core",
	"postcore",
	"arch",
	"subsys",
	"fs",
	"device",
	"late",
};
```

All functions which are marked as `initcall` by these identifiers, will be called in the same order presented in the `initcall_level_names` array, in other words, at first `early initcalls` will be called, then `core initcalls` and so forth. From this moment we know a little about `initcall` mechanism, so we can start to dive into the source code of the Linux kernel to see how this mechanism is implemented.

## Implementation initcall mechanism in the Linux kernel

The Linux kernel provides a set of macros from the [include/linux/init.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/init.h) header file to mark a given function as `initcall`. All of these macros are pretty simple:

```
#define early_initcall(fn)		__define_initcall(fn, early)
#define core_initcall(fn)		__define_initcall(fn, 1)
#define postcore_initcall(fn)		__define_initcall(fn, 2)
#define arch_initcall(fn)		__define_initcall(fn, 3)
#define subsys_initcall(fn)		__define_initcall(fn, 4)
#define fs_initcall(fn)			__define_initcall(fn, 5)
#define device_initcall(fn)		__define_initcall(fn, 6)
#define late_initcall(fn)		__define_initcall(fn, 7)
```

and as we may see these macros just expand to the call of the `__define_initcall` macro from the same header file. Moreover, the `__define_initcall` macro takes two arguments:

* `fn` - callback function which will be called during call of `initcalls` of the certain level;
* `id` - identifier to identify `initcall` to prevent error when two the same `initcalls` point to the same handler.

The implementation of the `__define_initcall` macro looks like:

```
#define __define_initcall(fn, id) \
	static initcall_t __initcall_##fn##id __used \
	__attribute__((__section__(".initcall" #id ".init"))) = fn; \
	LTO_REFERENCE_INITCALL(__initcall_##fn##id)
```

To understand the `__define_initcall` macro, first of all let's look at the `initcall_t` type. This type is defined in the same [header](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/init.h) file and it represents pointer to a function which returns [integer](https://en.wikipedia.org/wiki/Integer) which will be result of the `initcall`:

```
typedef int (*initcall_t)(void);
```

Now let's return to the `__define_initcall` macro. The [##](https://gcc.gnu.org/onlinedocs/cpp/Concatenation.html) provides ability to concatenate two symbols. In our case, the first line of the `__define_initcall` macro produces the definition of a given function, `__initcall_<function-name>_<id>`, which is located in the `.initcall <id> .init` [ELF section](http://www.skyfree.org/linux/references/ELF\_Format.pdf) and marked with the `__user` [gcc](https://en.wikipedia.org/wiki/GNU\_Compiler\_Collection) attribute. If we look at [include/asm-generic/vmlinux.lds.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/asm-generic/vmlinux.lds.h) header file, which represents data for the kernel [linker](https://en.wikipedia.org/wiki/Linker\_\(computing\)) script, we will see that all of `initcalls` sections will be placed in the `.data` section:

```
#define INIT_CALLS					\
		VMLINUX_SYMBOL(__initcall_start) = .;	\
		*(.initcallearly.init)					\
		INIT_CALLS_LEVEL(0)					    \
		INIT_CALLS_LEVEL(1)					    \
		INIT_CALLS_LEVEL(2)					    \
		INIT_CALLS_LEVEL(3)					    \
		INIT_CALLS_LEVEL(4)					    \
		INIT_CALLS_LEVEL(5)					    \
		INIT_CALLS_LEVEL(rootfs)				\
		INIT_CALLS_LEVEL(6)					    \
		INIT_CALLS_LEVEL(7)					    \
		VMLINUX_SYMBOL(__initcall_end) = .;

#define INIT_DATA_SECTION(initsetup_align)	\
	.init.data : AT(ADDR(.init.data) - LOAD_OFFSET) {	   \
        ...                                                \
        INIT_CALLS						                   \
        ...                                                \
	}
```

and their names are going to be as follows (got from System.map):

```
...
ffffffff8320ce60 t __initcall_arch_kdebugfs_init3
...
ffffffff8320d0e0 t __initcall_nmi_warning_debugfs5
...
```

The attribute `__used` is defined in the [include/linux/compiler-gcc.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/compiler-gcc.h) header file and it expands to the definition of the following `gcc` attribute:

```
#define __used   __attribute__((__used__))
```

which prevents `variable defined but not used` warning. The last line of the `__define_initcall` macro is:

```
LTO_REFERENCE_INITCALL(__initcall_##fn##id)
```

depends on the `CONFIG_LTO` kernel configuration option and just provides stub for the compiler [Link time optimization](https://gcc.gnu.org/wiki/LinkTimeOptimization):

```
#ifdef CONFIG_LTO
#define LTO_REFERENCE_INITCALL(x) \
        static __used __exit void *reference_##x(void)  \
        {                                               \
                return &x;                              \
        }
#else
#define LTO_REFERENCE_INITCALL(x)
#endif
```

In order to prevent any problem when there is no reference to a variable in a module, it will be moved to the end of the program. That's all about the `__define_initcall` macro. So, all of the `*_initcall` macros will be expanded during compilation of the Linux kernel, and all `initcalls` will be placed in their sections and all of them will be available from the `.data` section and the Linux kernel will know where to find a certain `initcall` to call it during initialization process.

As `initcalls` can be called by the Linux kernel, let's look how the Linux kernel does this. This process starts in the `do_basic_setup` function from the [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) source code file:

```
static void __init do_basic_setup(void)
{
    ...
    ...
    ...
   	do_initcalls();
    ...
    ...
    ...
}
```

which is called during the initialization of the Linux kernel, right after main steps of initialization like memory manager related initialization, `CPU` subsystem and others are already finished. The `do_initcalls` function just goes through the array of `initcall` levels and call the `do_initcall_level` function for each level:

```
static void __init do_initcalls(void)
{
	int level;

	for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++)
		do_initcall_level(level);
}
```

The `initcall_levels` array is defined in the same source code [file](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) and contains pointers to the sections which were defined in the `__define_initcall` macro:

```
static initcall_t *initcall_levels[] __initdata = {
	__initcall0_start,
	__initcall1_start,
	__initcall2_start,
	__initcall3_start,
	__initcall4_start,
	__initcall5_start,
	__initcall6_start,
	__initcall7_start,
	__initcall_end,
};
```

If you are interested, you can find these sections in the `arch/x86/kernel/vmlinux.lds` linker script which is generated after the Linux kernel compilation:

```
.init.data : AT(ADDR(.init.data) - 0xffffffff80000000) {
    ...
    ...
    ...
    ...
    __initcall_start = .;
    *(.initcallearly.init)
    __initcall0_start = .;
    *(.initcall0.init)
    *(.initcall0s.init)
    __initcall1_start = .;
    ...
    ...
}
```

If you are not familiar with this then you can know more about [linkers](https://en.wikipedia.org/wiki/Linker\_\(computing\)) in the special [part](https://0xax.gitbook.io/linux-insides/summary/misc/linux-misc-3) of this book.

As we just saw, the `do_initcall_level` function takes one parameter - level of `initcall` - and does the following two things:

* parses the `initcall_command_line` which is copy of usual kernel [command line](https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/kernel-parameters.rst) which may contain parameters for modules with the `parse_args` function from the [kernel/params.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/params.c) source code file;
* call the `do_on_initcall` function for each level:

```
for (fn = initcall_levels[level]; fn < initcall_levels[level+1]; fn++)
		do_one_initcall(*fn);
```

The `do_one_initcall` does the main job for us. As we may see, this function takes one parameter which represent `initcall` callback function and does the call of the given callback:

```
int __init_or_module do_one_initcall(initcall_t fn)
{
	int count = preempt_count();
	int ret;
	char msgbuf[64];

	if (initcall_blacklisted(fn))
		return -EPERM;

	if (initcall_debug)
		ret = do_one_initcall_debug(fn);
	else
		ret = fn();

	msgbuf[0] = 0;

	if (preempt_count() != count) {
		sprintf(msgbuf, "preemption imbalance ");
		preempt_count_set(count);
	}
	if (irqs_disabled()) {
		strlcat(msgbuf, "disabled interrupts ", sizeof(msgbuf));
		local_irq_enable();
	}
	WARN(msgbuf[0], "initcall %pF returned with %s\n", fn, msgbuf);

	return ret;
}
```

Let's try to understand what does the `do_one_initcall` function does. First of all we increase [preemption](https://en.wikipedia.org/wiki/Preemption\_\(computing\)) counter so that we can check it later to be sure that it is not imbalanced. After this step we can see the call of the `initcall_backlist` function which goes over the `blacklisted_initcalls` list which stores blacklisted `initcalls` and releases the given `initcall` if it is located in this list:

```
list_for_each_entry(entry, &blacklisted_initcalls, next) {
	if (!strcmp(fn_name, entry->buf)) {
		pr_debug("initcall %s blacklisted\n", fn_name);
		kfree(fn_name);
		return true;
	}
}
```

This blacklist is filled during early Linux kernel initialization from the Linux kernel command line.

After the blacklisted `initcalls` are handled, the next part of code directly calls the `initcall` callback:

```
if (initcall_debug)
	ret = do_one_initcall_debug(fn);
else
	ret = fn();
```

`initcall_debug` variable defines if the call should be handled through the debug codepath (with more information being printed to the [kernel log buffer](https://en.wikipedia.org/wiki/Dmesg)) or not, where the callback will finally be executed. The `initcall_debug` variable is defined in the [same](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) source code file:

```
bool initcall_debug;
```

The value of the variable can be set from the kernel commands via the `initcall_debug` parameter, as we can read from the [documentation](https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/kernel-parameters.rst) of the Linux kernel command line:

```
initcall_debug	[KNL] Trace initcalls as they are executed.  Useful
                      for working out where the kernel is dying during
                      startup.
```

And that's true. If we will look at the implementation of the `do_one_initcall_debug` function, we will see that it does the same as the `do_one_initcall` function, i.e. the `do_one_initcall_debug` function calls the given `initcall` and prints some information (like the [pid](https://en.wikipedia.org/wiki/Process\_identifier) of the currently running task, duration of execution of the `initcall` and etc.) related to the execution of the given `initcall`:

```
static int __init_or_module do_one_initcall_debug(initcall_t fn)
{
	ktime_t calltime, delta, rettime;
	unsigned long long duration;
	int ret;

	printk(KERN_DEBUG "calling  %pF @ %i\n", fn, task_pid_nr(current));
	calltime = ktime_get();
	ret = fn();
	rettime = ktime_get();
	delta = ktime_sub(rettime, calltime);
	duration = (unsigned long long) ktime_to_ns(delta) >> 10;
	printk(KERN_DEBUG "initcall %pF returned %d after %lld usecs\n",
		 fn, ret, duration);

	return ret;
}
```

As an `initcall` was called by the one of the `do_one_initcall` or `do_one_initcall_debug` functions, we may see two checks in the end of the `do_one_initcall` function. The first one checks the amount of possible `__preempt_count_add` and `__preempt_count_sub` calls inside of the executed initcall, and if this value is not equal to the previous value of the preemptible counter, we add the `preemption imbalance` string to the message buffer and set correct value of the preemptible counter:

```
if (preempt_count() != count) {
	sprintf(msgbuf, "preemption imbalance ");
	preempt_count_set(count);
}
```

The last check the state of local [IRQs](https://en.wikipedia.org/wiki/Interrupt\_request\_\(PC\_architecture\)) and if they are disabled, we add the `disabled interrupts` strings to log buffer and enable `IRQs` for the current processor to make sure that `IRQs` are enabled after each `initcall` is completed (in case the callback disabled it and didn't enable before exiting):

```
if (irqs_disabled()) {
	strlcat(msgbuf, "disabled interrupts ", sizeof(msgbuf));
	local_irq_enable();
}
```

That's all. In this way the Linux kernel does initialization of many subsystems in a correct order. From now on, we know what is the `initcall` mechanism in the Linux kernel. In this part, we covered main general portion of the `initcall` mechanism but we left some important concepts. Let's make a short look at these concepts.

First of all, we have missed one level of `initcalls`, this is `rootfs initcalls`. You can find definition of the `rootfs_initcall` in the [include/linux/init.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/init.h) header file along with all similar macros which we saw in this part:

```
#define rootfs_initcall(fn)		__define_initcall(fn, rootfs)
```

As we may understand from the macro's name, its main purpose is to store callbacks which are related to the [rootfs](https://en.wikipedia.org/wiki/Initramfs). Besides this goal, it may be useful to initialize other components after initialization related to filesystems level was already done, but before devices related initcalls. For example, the decompression of the [initramfs](https://en.wikipedia.org/wiki/Initramfs) which occurred in the `populate_rootfs` function from the [init/initramfs.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/initramfs.c) source code file:

```
rootfs_initcall(populate_rootfs);
```

From this place, we may see familiar output:

```
[    0.199960] Unpacking initramfs...
```

Besides the `rootfs_initcall` level, there are additional `console_initcall`, `security_initcall` and other secondary `initcall` levels. The last thing that we have missed is the set of the `*_initcall_sync` levels. Almost each `*_initcall` macro that we have seen in this part, has macro companion with the `_sync` prefix:

```
#define core_initcall_sync(fn)		__define_initcall(fn, 1s)
#define postcore_initcall_sync(fn)	__define_initcall(fn, 2s)
#define arch_initcall_sync(fn)		__define_initcall(fn, 3s)
#define subsys_initcall_sync(fn)	__define_initcall(fn, 4s)
#define fs_initcall_sync(fn)		__define_initcall(fn, 5s)
#define device_initcall_sync(fn)	__define_initcall(fn, 6s)
#define late_initcall_sync(fn)		__define_initcall(fn, 7s)
```

The main goal of these additional levels is to wait for completion of all modules related initialization routines for a certain level.

Another point worthy of mention is the `module_init(x)` macro, defined at [include/linux/module.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/module.h) as:

```
#define module_init(x)   __initcall(x);
```

If we follow and check what's the definition of `__initcall(x)` at [include/linux/init.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/init.h) we can see that it's being set as an `device_initcall`:

```
#define __initcall(fn) device_initcall(fn)
```

With that we can conclude that when a function set as `__init` of certain module isn't explicitly added to a specific initcall category, but using `module_init()` macro, it is added to device initcall list by default.

That's all.

## Conclusion

In this part we saw the important mechanism of the Linux kernel which allows to call a function which depends on the current state of the Linux kernel during its initialization.

If you have questions or suggestions, feel free to ping me in twitter [0xAX](https://twitter.com/0xAX), drop me [email](mailto:anotherworldofworld@gmail.com) or just create [issue](https://github.com/0xAX/linux-insides/issues/new).

**Please note that English is not my first language and I am really sorry for any inconvenience. If you found any mistakes please send me PR to** [**linux-insides**](https://github.com/0xAX/linux-insides)**.**

## Links

* [callback](https://en.wikipedia.org/wiki/Callback\_\(computer\_programming\))
* [debugfs](https://en.wikipedia.org/wiki/Debugfs)
* [integer type](https://en.wikipedia.org/wiki/Integer)
* [symbols concatenation](https://gcc.gnu.org/onlinedocs/cpp/Concatenation.html)
* [GCC](https://en.wikipedia.org/wiki/GNU\_Compiler\_Collection)
* [Link time optimization](https://gcc.gnu.org/wiki/LinkTimeOptimization)
* [Introduction to linkers](https://0xax.gitbook.io/linux-insides/summary/misc/linux-misc-3)
* [Linux kernel command line](https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/kernel-parameters.rst)
* [Process identifier](https://en.wikipedia.org/wiki/Process\_identifier)
* [IRQs](https://en.wikipedia.org/wiki/Interrupt\_request\_\(PC\_architecture\))
* [rootfs](https://en.wikipedia.org/wiki/Initramfs)
* [previous part](https://0xax.gitbook.io/linux-insides/summary/concepts/linux-cpu-2)
