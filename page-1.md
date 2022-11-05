# Page 1



Dans ce post je part du principe que vous connaissez les concepts de kernel et de système d’exploitation. Je vais vous expliquer comment extraire `vmlinux` de son format compressé `vmlinuz`.

### vmlinuz

`vmlinuz` est le noyau Linux compressé au format Gzip. Celui-ci permet de charger le système en mémoire.

Le fichier en question se trouve dans le repertoire `/boot` :

``

```
$ ls /boot 
abi-4.10.0-35-generic         memtest86+.bin
abi-4.10.0-37-generic         memtest86+.elf
config-4.10.0-35-generic      memtest86+_multiboot.bin
config-4.10.0-37-generic      System.map-4.10.0-35-generic
grub                          System.map-4.10.0-37-generic
initrd.img-4.10.0-35-generic  vmlinuz-4.10.0-35-generic
initrd.img-4.10.0-37-generic  vmlinuz-4.10.0-37-generic
```

Les fichiers de configuration lors du demarrage du noyau Linux sont aussi présent.

En utilisant la commande `file` et comme argument `vmlinuz`, on peut faire une analyse basique du fichier.

``

```
$ sudo file vmlinuz-4.10.0-37-generic 
vmlinuz-4.10.0-37-generic: Linux kernel x86 boot executable bzImage, version 4.10.0-37-generic (buildd@lgw01-amd64-037) #41~16.04.1-Ubuntu S, RO-rootFS, swap_dev 0x7, Normal VGA
```

On voit donc que le fichier est un executable demarrable (bootable) pour architecture **x86**. De plus il nous donne la version du dît noyau : `4.10.0-37-generic`.

Il faut aussi savoir que le fichier `vmlinuz` à la racine des _rootfs_ n’est qu’un lien symbolique vers `/boot/vmlinuz-4.10.0-37-generic`.

Malheuresement on ne peut pas aller plus loin sans décompresser `vmlinuz`. Un exemple avec l’outil `readelf`, il n’est pas possible d’afficher les informations du header du noyau. En effet étant compressé, il n’est pas considéré comme fichier [ELF](https://fr.wikipedia.org/wiki/Executable\_and\_Linkable\_Format).

``

```
$ readelf -h vmlinuz-4.10.0-35-generic
readelf: Error: Not an ELF file - it has the wrong magic bytes at the start
```

#### bzImage

Avant de décompresser l’image, il faut savoir ce qu’il s’est passé.

Le noyau Linux étant arrivé à maturité, la taille des kernels générés ont dépassé la taille maximale imposée par certaines architectures.

De plus avec l’expension des objets connectés - qui la plupart tournent sous un noyau Linux - ont besoin d’un maximum d’espace disponible pour un minumum de stockage. C’est pour celà que le format bzimage a été developpé : en divisant le noyaux en plusieurs régions de mémoire qui ne se suivent pas.

La création de l’image passe par le [Makefile](https://github.com/torvalds/linux/blob/master/Makefile), à l’aide de l’argument `bzImage`, après à avoir compilé le kernel, le Makefile va compresser l’image au format Gzip.

``

```
$ make help | grep bzImage
* bzImage      - Compressed kernel image (arch/x86/boot/bzImage)
```

Voici ce qui se passe quand on compresse l’image `vmlinux`

\


```
$ make bzImage
  CHK     include/config/kernel.release
  CHK     include/generated/uapi/linux/version.h
  CHK     include/generated/utsrelease.h
  CHK     include/generated/bounds.h
  CHK     include/generated/timeconst.h
  CHK     include/generated/asm-offsets.h
  CALL    scripts/checksyscalls.sh
  CHK     scripts/mod/devicetable-offsets.h
  CHK     include/generated/compile.h
  DATAREL arch/x86/boot/compressed/vmlinux
Kernel: arch/x86/boot/bzImage is ready  (#1)
$ file arch/x86/boot/bzImage
arch/x86/boot/bzImage: Linux kernel x86 boot executable bzImage, version 4.12.0 (mathieu@ubuntu) #1 SMP Tue Oct 17 09:48:34 PDT 2017, RO-rootFS, swap_dev 0x6, Normal VGA
```

`</table>`

Voici l’architecture de bzImage

[![bzimg](https://blog.matteyeux.com/images/bzimage.png)](https://blog.matteyeux.com/images/bzimage.png)

Vous pouvez voir ci-dessous le Makefile qui créé [l’image compressée](https://github.com/torvalds/linux/blob/5924bbecd0267d87c24110cbe2041b5075173a25/arch/nios2/boot/compressed/Makefile) de `vmlinux`:

``

```
targets		:= vmlinux head.o misc.o piggy.o vmlinux.lds
asflags-y	:=

OBJECTS = $(obj)/head.o $(obj)/misc.o

LDFLAGS_vmlinux := -T

$(obj)/vmlinux: $(obj)/vmlinux.lds $(OBJECTS) $(obj)/piggy.o FORCE
	$(call if_changed,ld)

LDFLAGS_piggy.o := -r --format binary --oformat elf32-littlenios2 -T

$(obj)/piggy.o: $(obj)/vmlinux.scr $(obj)/../vmlinux.gz FORCE
	$(call if_changed,ld)
```

La première fois que j’ai voulu faire du reverse engineering sur un kernel Ubuntu, j’ai ouvert `vmlinuz-4.10.0-35-generic` extrait du repertoire `/boot` pour l’ouvrir dans [IDA](https://www.hex-rays.com/products/ida/index.shtml).

Or celui-ci n’a pas reconnu le fichier comme étant un fichier ELF x86\_64.

[![idafail](https://blog.matteyeux.com/images/ida\_fail.png)](https://blog.matteyeux.com/images/ida\_fail.png)

En cherchant un peu j’ai compris qu’il était compressé, voici donc comme le décompresser.

#### Prérequis

Pour extraire `vmlinux`, il vous faut bien sur une distribution Linux. Pour ma part je vais prendre [Ubuntu 16.04](https://www.ubuntu.com/desktop).

Ensuite il faut que vous téléchargiez les headers du noyau Linux pour la version votre noyau installé.

Vous pouvez obtenir la version de votre noyau avec la commande `uname` comme ceci :

``

```
$ uname -r
4.10.0-37-generic
```

Si vous avez une distribution de type [Debian-like](https://www.debian.org/misc/children-distros) alors vous pouvez utiliser le gestionnaire de paquets `apt-get`:

``

```
$ sudo apt-get remove linux-headers-$(uname -r)
Reading package lists... Done
Building dependency tree       
Reading state information... Done
[...]
Preparing to unpack .../linux-headers-4.10.0-37-generic_4.10.0-37.41~16.04.1_amd64.deb ...
Unpacking linux-headers-4.10.0-37-generic (4.10.0-37.41~16.04.1) ...
Setting up linux-headers-4.10.0-37-generic (4.10.0-37.41~16.04.1) ...
```

Avant de pouvoir decompresser le fichier, il faut le deplacer quelque part ou l’on a un peu plus de droits que `/boot` , car il nous est impossible de le modifier ici. Comme vous le voyez, seul le user **root** peut lire ou écrire ce fichier.

``

```
$ ls -la /boot/vmlinuz-$(uname -r)
-rw------- 1 root root 7405184 Oct  6 17:35 vmlinuz-4.10.0-37-generic
```

Pour ce faire on va copier `vmlinuz` dans un repertoire `tmp` que l’on va créer puis changer les droits du fichier pour un user un peu moins privilégié.

``

```
$ mkdir ~/tmp
$ sudo cp /boot/vmlinuz-$(uname -r) ~/tmp
$ chown $(whoami) ~/tmp/vmlinuz-$(uname -r)
```

Il ne reste plus qu’a extraire `vmlinux` à l’aide d’un script téléchargé lors de l’installation des `linux-headers` de votre kernel:

`$ /usr/src/linux-headers-$(uname -r)/scripts/extract-vmlinux ~/tmp/vmlinuz-$(uname -r) > vmlinux`

Le script [extract-vmlinux](https://github.com/torvalds/linux/blob/5924bbecd0267d87c24110cbe2041b5075173a25/scripts/extract-vmlinux) va extraire `vmlinux` de l’image du noyau précédement copiée `~/tmp`.

Vous avez maintenant votre `vmlinux` reconnu comme fichier ELF :

``

```
$ file ~/tmp/vmlinux
vmlinux: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, BuildID[sha1]=a5d6efe91b4658a7a920aacfc8b20021143a1dd5, stripped
```

Et finalement IDA reconnait le fichier.

[![nofail](https://blog.matteyeux.com/images/idanofail.png)](https://blog.matteyeux.com/images/idanofail.png)

[![idacool](https://blog.matteyeux.com/images/idacool.png)](https://blog.matteyeux.com/images/idacool.png)

L’extraction de `vmlinux` n’est pas en soit une tache complexe, mais il faut comprendre pourquoi et comment extraire ce fameux fichier.

Bref c’est tout. N’hésitez pas à me corriger si j’ai fait une erreur ou une/des faute\[s] !

Sources :

* https://en.wikipedia.org/wiki/Vmlinux
* https://fr.wikipedia.org/wiki/Executable\_and\_Linkable\_Format
