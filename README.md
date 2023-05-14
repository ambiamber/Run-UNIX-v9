# Run Ninth Edition UNIX on a TME Sun3/160 emulator

## Glossary of some files in this repo which are found in Norman Wilson's Norman_v9 folder on TUHS:

* batterpudding.tar is the sources and binaries for the Sun 3 port of unix v9.
* README-dak is David Kapilow's README from 1989 with some instructions on how to create a unix v9 system on SunOS and other things about v9.

## V9 Usage Notes

* symbolic links on UNIX v9 are derived from UNIX v8. UNIX v8 symbolic links are older than BSD symbolic links. Programs like tar and ls have different conventions.
* Symbolic links in tar files under v9 are stored differently from BSD tar files.  The tar program uses L as an argument to indicate that a symbolic link should be stored in the tar file instead of the contents of the file that it points to.
* The output of `ls -l` shows the letter L after the link count if the file is a symbolic link. Use `ls -lL` to see what the link points to.
* To see multi-column output from ls use `ls | mc` to pipe the output through the multi-column (mc) program.
* There is no vi, use Warren Montgomery emacs and /bin/ed.  For documenation of that emacs, see [DTIC ADA137264: Beginners' Tutorial for JHU UNIX at BRL (Ballistic Research Laboratory)](https://archive.org/details/DTIC_ADA137264/page/n79/mode/2up).
* The batterpudding.tar tape is probably missing things that are in other v9 distributions. See README-dak for statements about where the tape came from.
* It might be possible to existing v8 and v10 source code to fill in some gaps.  For instance v8 and v10 uniq.c are identical.

## How to do this

* clone and build https://github.com/ambiamber/Run-Sun3-SunOS-4.1.1 and when you install SunOS using it, don't install optional packages, use the "required" option on the Software Form as you need free space for v9.
* clone https://github.com/ambiamber/Run-UNIX-v9 
* copy Run-Sun3-SunOS-4.1.1/sun3-disk.img to Run-UNIX-v9/
* create a new empty disk image for v9. Note that the dd will not remove an old v9-disk.img, if you need to try again, rm the img.
```
cd Run-UNIX-v9
unxz -v batterpudding.tar.xz
dd if=/dev/zero of=v9-disk.img bs=1 count=1 seek=999999999
```
* Possibly edit the `mkroms` script before running it.
```
./mkroms
```
* Start the TME `tmesh` program you installed with the Run-Sun3-SunOS software, or use a newer TME if one is available.  The `SUN3-SunOS` TME configuration file boots from a SunOS disk and has the v9 disk as a second drive. The `SUN3-SunOS` file sets the v9 disk to SCSI ID 1 but it shows up in SunOS as sd2 not sd1.  Maybe someday I'll find out why.
```
./run-tmesh SUN3-SunOS
```
* Once SunOS boots, log in as root
* Change the backspace key and enable file completion.  Unlike Linux bash tab completion, the SunOS csh uses Esc to complete file names.
```
login: root
stty erase '^h'
set filec
```
* Use the SunOS `format` command to partition and label the v9 disk.
```
format
choose disk 1, sd2
At the menu select `5. other`

Enter number of data cylinders: 2046
Enter number of alternate cylinders [2]:
Enter number of physical cylinders [2048]:
Enter number of heads: 16
Enter number of data sectors/track: 16
Enter rpm of drive [3600]:
Enter buffer skew [2]:
Enter write precomp cylinder [2048]:
Enter disk type name (remember quotes): Awesome
selecting sd0: <Awesome>
[disk formatted, no defect list found]
No defined partition tables.
```
disk partition sizes:
```
   Start  Length
a    0    156 cyl is about  20 MB (root) # blocks 39936
b  157    122 cyl is about  20 MB (swap) # blocks 32940
c    0   2046 cyl is about 260 MB (all ) # blocks 523776
g  278   1768 cyl is about 200 MB (/usr) # blocks 452608
```
Enter the partitioner and use these values:
```
format> partition

partition> a
Enter new starting cyl [0]:
Enter new # blocks [0, 0/0/0]; 39936

Repeat that for all of these values:

part  cyl   blocks
a       0    39936
b     157    40960
c       0   523776
g     317   442624
```
You can use the `print` command to see the partition table.
You can use `?` to display the menu.
```
partition> name
Enter table name (remember quotes): awesome
```
When you are satisfied that the partition table is correct, label the disk.
```
partition> label
Ready to label disk, continue? y

partition> quit
format> quit
```
* Extract the batterpudding.tar file into /usr/9th.  The README-dak file doesn't say where to extract the tar, it mentions that you have already done it based on the assumption that the README would not be available if you had not unarchived the tar file.
```
cd /usr
mkdir 9th
cd 9th
# Extract the batterpudding.tar file.
tar xf /dev/rst8
cd sys/construct
pwd
/usr/9th/sys/construct
```
* Build v9 tar and create compressed tar files of the source code. There is a per-file limit on the size of a file that mkfs can install via a proto file.  This procedure splits the source code into three tar files that fit inside the mkfs per-file size limit.
```
cd host
pwd
/usr/9th/sys/construct/host
make v9tar
mv v9tar /usr/9th
cd /usr/9th
# note that using Esc to complete file names (like bash tab completion) helps here:
v9tar cLf - cmd ipc jerq jtools libc libtermlib netb | compress -cv > src1.tar.Z
v9tar cLf - X11 | compress -cv > src2.tar.Z
# This avoids backing up sys/construct because it's too big. You're welcome to try doing it a different way.
v9tar cLf - sys/conf* sys/[3bdhimns]* | compress -cv > src3.tar.Z
du -s *.Z
4576  src1.tar.Z
3688  src2.tar.Z
1792  src3.tar.Z
mv *.Z sys/construct
# the csh history mechanism will change $! into the last word on the previous line
cd !$
```
Files can only be loaded into 9th by way of mkfs using proto files now. Sooner or later someone is going to get the tape drive or network to function and then that can be a way of transporting files.
```
cp proto0a.cdc proto0a.new
cp proto0g.cdc proto0g.new
```
* See README-dak for explanations about the mkfs proto files.
* Edit `proto0a.new` changing the second line from `2500 12800` to `2496 12672`.
* Edit `proto0g.new` changing the second line from `14037 65280` to `27664 65280`
* Keep editing `proto0g.new` near the bottom add this right before the last line with a dollar `$`
* Note that in vi, typing 3yyP will copy the current line and the next two as well, so you can copy three lines starting with src.
```
src1	d--775 3 4
	src1.tar.Z      ---644 3 2 src1.tar.Z
	$
src2	d--775 3 4
	src2.tar.Z      ---644 3 2 src2.tar.Z
	$
src3	d--775 3 4
	src3.tar.Z      ---644 3 2 src3.tar.Z
	$
```

There are SunOS binaries for v9 mkfs and fsck but I don't trust them; they may have been compiled for a different version of SunOS.  I just rebuild them from source.

```
cd host
make mkfs fsck
# ignore the messages that go "may be non-portable"
mv mkfs fsck ..
cd ..
```

The instructions in `README-dak` say to modify `construct/test/etc/passwd` but who are we kidding now that there is John the Ripper?  You can still modify the v9 passwd file if you like, but here are the passwords it comes with:
```
odak  1b415   # old dak (David A. Kapilow) account
dak   1b415   # regular dak (David A. Kapilow) account
root  halleys
```
In the `README-dak` he suggested modifying some other files like `construct/test/etc/whoami` which is similar to `/etc/hostname` in that it contains the hostname of the emulated UNIX v9.  Note that hostnames are limited to eight characters long.

Note that the "nothing" line at the start of the proto file normally is supposed to be the name of the file containing the boot block program but it doesn't work. The boot block is installed another way. There will be a message from mkfs `nothing: cannot open init` because of this.

Run `mkfs` to create the v9 file systems and load the files specified in the proto files into file systems mkfs creates.
```
./mkfs /dev/rsd2a proto0a.new
./mkfs /dev/rsd2g proto0g.new
```
Install the boot block.
```
cd test/stand
installboot bootpr /dev/rsd2a
cd ../..
```
The SunOS `fsck`, `mount`, etc. will not work for the v9 file systems. The v9 `fsck` will not recognize SunOS device special files but it will still work if you tell it to, as if it's working on a regular file.
```
./fsck /dev/rsd2a

file is not a block or character device; OK? y
[... skip fsck output ...]

./fsck /dev/rsd2g

file is not a block or character device; OK? y
[... skip fsck output ...]
```
If the file systems are clean you have a working v9 disk.  Halt SunOS
```
cd /
shutdown -h now
```
Ctrl-C the tmesh program.

README-dak says to boot into single user mode by using the `-s` option to the `b` command (`> b sd() -s`) but that needs you to be at the `>` prompt and that requires being able to interrupt the boot process.

A Sun3 workstation can be interrupted and sent to the boot PROM by typing what is called El-one-aye which happens when you hold down the L1 key and type the letter a.  A real Sun3 workstation has an L1 key the way a PeeCee has an F1 key.  The `sun-macros.txt` file controls what you type into tme to generate an L1. Unfortunately, I've never gotten this part to work.

Since that doesn't work, you can boot up to multi-user mode and then shutdown to single user.
```
./run-tmesh 9TH
```
Then in the emulator window run:
```
login: root
kill 1
# #
umount /dev/sd0g
```
The first `#` is the normal shell prompt you get after typing a command. The second `#` is because the system switched to single-user mode and started a new shell.  Unmounting /usr (/dev/rsd0g) is required because the startup script expects to be able to mount /usr.

Now you can run the script that unpacks the tar files for the commands and things other than the source code which is done as an add-on to the instructions in the README-dak.  The butterpudding.tar file contains the source code but the v9 construction steps documented in README-dak do not include installing the source code while my instructions do.

```
/startup
cd /usr
zcat src1/* | tar xvLf -
zcat src2/* | tar xvLf -
zcat src3/* | tar xvLf -
```
I don't yet know why you get `ranlib: warning: /lib/libC.a(misc.o): no symbol table` but it seems harmless. Same for `tar: backspace error: No such device or address`.

You can remove the source code tar.Z files or keep them as a backup.

To switch to multi-user mode type `exit`.

To shutdown the system use `shutdown` and then `halt`.
