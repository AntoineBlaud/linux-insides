# Integrity

[The kernel integrity sub-system](https://sourceforge.net/p/linux-ima/wiki/Home/) can be used to detect if a file has been altered (accidently or maliciously), both remotely and/or locally. It does that by appraising a file's measurement (its hash value) against a "good" value stored previously as an extended attribute ([on file systems which support extended attributes like ext3, ext4. etc](https://en.wikipedia.org/wiki/Extended\_file\_attributes).). Similar, but complementary, mechanisms are provided by other security technologies like SELinux which depending on policy can attempt to protect file integrity.

The Linux IMA (Integrity Measurement Architecture) subsystem introduces hooks within the Linux kernel to support creating and collecting hashes of files when opened, before their contents are accessed for read or execute. [The IMA measurement subsystem was added in linux-2.6.30 and is supported by Red Hat Enterprise Linux 8.](https://access.redhat.com/documentation/en-us/red\_hat\_enterprise\_linux/8/html/managing\_monitoring\_and\_updating\_the\_kernel/enhancing-security-with-the-kernel-integrity-subsystem\_managing-monitoring-and-updating-the-kernel)

The kernel integrity subsystem consists of two major components. The Integrity Measurement Architecture (IMA) is responsible for collecting file hashes, placing them in kernel memory (where userland applications cannot access/modify it) and allows local and remote parties to verify the measured values. The Extended Verification Module (EVM) detects offline tampering (this could help mitigate [evil-maid attacks](https://en.wikipedia.org/wiki/Evil\_maid\_attack)) of the security extended attributes.&#x20;

IMA maintains a runtime measurement list and, if anchored in a hardware Trusted Platform Module(TPM), an aggregate integrity value over this list. The benefit of anchoring the aggregate integrity value in the TPM is that the measurement list is difficult to compromise by a software attack, without it being detectable. Hence, on a trusted boot system, IMA-measurement can be used to attest to the system's runtime integrity.

#### &#x20;
