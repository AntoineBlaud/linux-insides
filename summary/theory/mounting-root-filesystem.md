---
description: Mounting root filesystem
---

# Mounting root filesystem

Mounting the root filesystem is a crucial part of system initialization. It is a fairly complex procedure because the Linux kernel allows the root filesystem to be stored in many different places, such as a hard disk partition, a floppy disk, a remote filesystem shared via NFS, or even a fictitious block device kept in RAM.

To keep the description simple, let's assume that the root filesystem is stored in a partition of a hard disk (the most common case, after all). While the system boots, the kernel finds the major number of the disk that contains the root filesystem in the root\_dev variable.

The root filesystem can be specified as a device file in the /dev directory either when compiling the kernel or by passing a suitable "root" option to the initial bootstrap loader. Similarly, the mount flags of the root filesystem are stored in the root mountflags variable. The user specifies these flags either by using the rdev external program on a compiled kernel image or by passing a suitable rootflags option to the initial bootstrap loader (see Appendix A).

Mounting the root filesystem is a two-stage procedure, shown in the following list.

1\. The kernel mounts the special rootfs filesystem, which just provides an empty directory that serves as initial mount point.

2\. The kernel mounts the real root filesystem over the empty directory.

Why does the kernel bother to mount the rootfs filesystem before the real one? Well, the rootfs filesystem allows the kernel to easily change the real root filesystem. In fact, in some cases, the kernel mounts and unmounts several root filesystems, one after the other. For instance, the initial bootstrap floppy disk of a distribution might load in RAM a kernel with a minimal set of drivers, which mounts as root a minimal filesystem stored in a RAM disk. Next, the programs in this initial root filesystem probe the hardware of the system (for instance, they determine whether the hard disk is EIDE, SCSI, or whatever), load all needed kernel modules, and remount the root filesystem from a physical block device.

The first stage is performed by the init\_mount\_tree( ) function, which is executed during system initialization:

struct file\_system\_type root\_fs\_type; root fs type.name = "rootfs";

root\_fs\_type.read\_super = rootfs\_read\_super; root\_fs\_type.fs\_flags = FS\_NOMOUNT; [register\_filesystem](https://www.halolinux.us/kernel-reference/filesystem-type-registration.html)(\&root\_fs\_type);

root vfsmnt = do kern mount("rootfs", 0, "rootfs", NULL);

The root\_fs\_type variable stores the descriptor object of the rootfs special filesystem; its fields are initialized, and then it is passed to the register\_filesystem( ) function (see the earlier section Section 12.3.2). The do\_kern\_mount( ) function mounts the special filesystem and returns the address of a new mounted filesystem object; this address is saved by init\_mount\_tree( ) in the root\_vfsmnt variable. From now on, root\_vfsmnt represents the root of the tree of the mounted filesystems.

The do\_kern\_mount( ) function receives the following parameters:

type

The type of filesystem to be mounted flags

The mount flags (see Table 12-13 in the later section Section 12.4.2)

name

The device file name of the block device storing the filesystem (or the filesystem type name for special filesystems)

data

Pointers to additional data to be passed to the read\_super method of the filesystem

The function takes care of the actual mount operation by performing the following operations:

1\. Checks whether the current process has the privileges for the mount operation (the check always succeeds when the function is invoked by init\_mount\_tree( ) because the system initialization is carried on by a process owned by root).

2\. Invokes get\_fs\_type( ) to search in the list of filesystem types and locate the name stored in the type parameter; get\_fs\_type( ) returns the address of the corresponding file\_system\_type descriptor.

3\. Invokes alloc\_vfsmnt( ) to allocate a new mounted filesystem descriptor and stores its address in the mnt local variable.

4\. Initializes the mnt->mnt\_devname field with the content of the name parameter.

5\. Allocates a new superblock and initializes it. do\_kern\_mount( ) checks the flags in the file\_system\_type descriptor to determine how to do this:

a. If fs\_requires\_dev is on, invokes get\_sb\_bdev( ) (see the later section Section 12.4.2)

b. If fs\_single is on, invokes get\_sb\_single( ) (see the later section Section 12.4.2)

c. Otherwise, invokes get\_sb\_nodev( )

6\. If the fs\_nomount flag in the file\_system\_type descriptor is on, sets the ms\_nouser flag in the superblock object.

7\. Initializes the mnt->mnt\_sb field with the address of the new superblock object.

8\. Initializes the mnt->mnt\_root and mnt->mnt\_mountpoint fields with the address of the [dentry object](https://www.halolinux.us/kernel-reference/dentry-objects.html) corresponding to the root directory of the filesystem.

9\. Initializes the mnt->mnt\_parent field with the value in mnt (the newly mounted filesystem has no parent).

10\. Releases the s\_umount semaphore of the superblock object (it was acquired when the object was allocated in Step 5).

11\. Returns the address mnt of the mounted filesystem object.

When the do\_kern\_mount( ) function is invoked by init\_mount\_tree( ) to mount the rootfs special filesystem, neither the fs\_requires\_dev flag nor the fs\_single flag are set, so the function uses get\_sb\_nodev( ) to allocate the superblock object. This function executes the following steps:

1\. Invokes get\_unnamed\_dev( ) to allocate a new fictitious block device identifier (see the earlier section Section 12.3.1).

2\. Invokes the read\_super( ) function, passing to it the filesystem type object, the mount flags, and the fictitious block device identifier. In turn, this function performs the following actions:

a. Allocates a new superblock object and puts its address in the local variable s.

b. Initializes the s->s\_dev field with the block device identifier.

c. Initializes the s->s\_flags field with the mount flags (see Table 12-13).

d. Initializes the s->s\_type field with the filesystem type descriptor of the filesystem.

e. Acquires the sb\_lock [spin lock](https://www.halolinux.us/kernel-reference/spin-locks.html).

f. Inserts the superblock in the global circular list whose head is super blocks.

g. Inserts the superblock in the filesystem type list whose head is s->s\_type->fs\_supers.

i. Acquires for writing the s->s\_umount read/write semaphore. j. Acquires the s->s\_lock semaphore.

k. Invokes the read\_super method of the filesystem type.

l. Sets the ms\_active flag in s->s\_flags.

m. Releases the s->s\_lock semaphore.

n. Returns the address s of the superblock.

3\. If the filesystem type is implemented by a kernel module, increments its usage counter.

4\. Returns the address of the new superblock.

The second stage of the mount operation for the root filesystem is performed by the mount\_root( ) function near the end of the system initialization. For the sake of brevity, we consider the case of a disk-based filesystem whose device files are handled in the traditional way (we briefly discuss in Chapter 13 how the devfs virtual filesystem offers an alternative way to handle device files). In this case, the function performs the following operations:

1\. Allocates a buffer and fills it with a list of filesystem type names. This list is either passed to the kernel in the rootfstype boot parameter or is built by scanning the elements in the simply linked list of filesystem types.

2\. Invokes the bdget( ) and blkdev\_get( ) functions to check whether the ROOT\_dev root device exists and is properly working.

3\. Invokes get\_super( ) to search for a superblock object associated with the root dev device in the super blocks list. Usually none is found because the root filesystem is still to be mounted. The check is made, however, because it is possible to remount a previously mounted filesystem. Usually the root filesystem is mounted twice during the system boot: the first time as a read-only filesystem so that its integrity can be safely checked; the second time for reading and writing so that normal operations can start. We'll suppose that no superblock object associated with the root dev device is found in the super blocks list.

4\. Scans the list of filesystem type names built in Step 1. For each name, invokes get\_fs\_type( ) to get the corresponding file\_system\_type object, and invokes read\_super( ) to attempt to read the corresponding superblock from disk. As described earlier, this function allocates a new superblock object and attempts to fill it by using the method to which the read\_super field of the file\_system\_type object points. Since each filesystem-specific method uses unique magic numbers, all read\_super( ) invocations will fail except the one that attempts to fill the superblock by using the method of the filesystem really used on the root device. The read\_super( ) method also creates an inode object and a dentry object for the root directory; the dentry object maps to the inode object.

5\. Allocates a new mounted filesystem object and initializes its fields with the root\_dev block device name, the address of the superblock object, and the address of the dentry object of the root directory.

6\. Invokes the graft\_tree( ) function, which inserts the new mounted filesystem object in the children list of root\_vfsmnt, in the global list of mounted filesystem objects, and in the mount\_hashtable hash table.

7\. Sets the root and pwd fields of the fs\_struct table of current (the init process) to the dentry object of the root directory.
