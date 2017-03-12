# Implementing a Linux System Call
Before we begin, I highly suggest downloading a virtual machine to do kernel compilation and testing. The last thing you want is to leave your computer in a unuseable state. OSBoxes has a collection of prebuilt virtual machines for VirtualBox and VMWare which you can download for free. I suggest using Arch Linux 64-bit (that is wht this tutorial will follow) as it is a minimal distro. Link here: http://www.osboxes.org/arch-linux/.

## Setting Up
First we need to download the Linux kernel. We will not clone the kernel from GitHub as that contains the very latest patches, instead, we want a stable build which we can get from www.kernel.org. Specifically, we will use version 4.7.1, simply because it has very few compatiblity issues with this tutorial. To download it, open a terminal and run 
```
curl -O -J https://www.kernel.org/pub/linux/kernel/v4.x/linux-4.7.1.tar.xz
```

This will download the archive for the kernel, which we will extract by running

```
tar xvf linux-4.7.1.tar.xz
```

Now, before we start modifying the kernel, we want to make sure all of the build tools are installed and updated. Specifically we need to install `bc`, a calculator for floating point numbers, and then upgrade our system to make sure that all of our libraries are at the correct version, this can be done by:
```
sudo pacman -S bc
```
Then to upgrade the system:
```
sudo pacman -Syu
```
If you get errors about corrupt packages or invalid PGP keys, simply run this to update your local keyring
```
sudo pacman -S archlinux-keyring
```
And then rerun the `-Syu` command. Finally, reboot your system to ensure all upgrades are in effect.

## Kernel Configuration

Next we need to configure our kernel to our spec. First `cd` into the directory where you extacted the kernel. Next we need to write a `.config` file to tell the Linux Makefile how to compile as per our needs. The problem here is that modern Linux can have thousands of options, so instead, we will take our operating system's config file, and just use that. It will ensure we have a good config (one that works) and we haven't missed any options. To do this run (within the extracted Linux code)
```
zcat /proc/config.gz > .config
```
This will create our `.config` file. We do need to manually change one thing though. Open up the `.config` file and find the line beginning with `CONFIG_LOCALVERSION`. It may have some value like `-arch`. Change that to something of your liking, for example `-sigops`. This ensures our kernel version is no longer arch, but something unique. We are now ready to start writing our system call!

## Writing the System Call

Linux indexes system calls through a table of information about these calls located in `arch/x86/entry/syscalls/syscall_64.tbl`. Upon opening this file, you will see all the system calls and their corresponding numbers. Go to the bottom of the first block of systems calls (on 4.7.1, 328) and lets add ours to this list. It will look like this:
```
329     common  sigops                  sys_sigops
```
Note that we use tabs to seperate the values. But what do these values mean? `329` is our syscall number, `common` tells linux that both 64 and 32-bit architecures can use our system call, `sigops` is the name of our syscall and `sys_sigops` is the internal name.

With this done, we can write the code for our system call. Open up the file `kernel/sys.c`. All miscellaneous are placed here, so it's the ideal place to put our hello world system call. Go to the bottom of the file and add the following code:
```
SYSCALL_DEFINE1(sigops, char *, msg) {
    char buf[256];
    long copied = strncpy_from_user(buf, msg, sizeof(buf));
    if (copied < 0 || copied == sizeof(buf)) {
        return -EFAULT;
    }
    printk(KERN_INFO "SIGOPS syscall called with \"%s\"\n", buf);
    return 0;
}
```

What is this code doing? First, `SYSCALL_DEFINE1` is a macro that builds a function header for a 1 argument system call. It takes in 3 parameters, first is the name of the syscall, next is the type of the argument to the syscall, and finally, the name of the argument passed in. Next, we create a buffer to store the argument, and we do a `strncpy_from_user`. This is required because the kernel and the user have seperate memory spaces, meaning that the kernel cannot directly operate on user data, so it must first copy it over to the kernel, and then operate on it, hence the "from user" part. Next we do some simple error checking, and finally, we print out whatever mesasge was passed in. One thing to notice is the `KERN_INFO` statment. This is another Linux macro that will let us automatically print to the Linux logging system, thus not spamming the user terminal with print statements. 

And that's all there is to it! We've implemented a system call! Now we have to compile and install our modified kernel.

## Compiling and Installing the Kernel
Compiling the kernel is a easy, but tedious task. I will list out the steps needed to do it. In essense, what we are doing is compiling the kernel, installing it as a bootable kernel, and making a option in our GRUB bootloader to let it boot form that.
 - `make -j2` 
 - `make modules_install`
 - `cp arch/x86_64/boot/bzImage /boot/vmlinuz-linux-sigops`
 - `sed s/linux/linux-sigops/g </etc/mkinitcpio.d/linux.preset >/etc/mkinitcpio.d/linux-sigops.preset`
 - `mkinitcpio -p linux-sigops`
 - `grub-mkconfig -o /boot/grub/grub.cfg`

With those steps done, go ahead and reboot. When in the GRUB menu, pick "Advanced options" and select your modified kernel. If everything was done right, you will boot into your new kernel! You can now test it.

## Testing the system call

To test the system call, I have written a small program:

```
#define _GNU_SOURCE
#include <unistd.h>
#include <sys/syscall.h>
#include <stdio.h>

/*
 * Put your syscall number here.
 */
#define SYS_sigops 329

int main(int argc, char **argv)
{
  if (argc <= 1) {
    printf("Must provide a string to give to system call.\n");
    return -1;
  }
  char *arg = argv[1];
  printf("Making system call with \"%s\".\n", arg);
  long res = syscall(SYS_sigops, arg);
  printf("System call returned %ld.\n", res);
  return res;
}
```

This program simply invokes our syscall with any string passed in, for example, if compiled with `gcc -o systest systest.c` then doing `./systest 'Hello World!'` will run the program with "Hello World!" being our string. But where did it print? Remember that we set our logging level in the kernel, so to see our output, run the `dmesg` command, and at the very end, you should see the output of your system call.

If everything worked, then congrats! You have successfully written a system call, compiled the Linux kernel, and tested your syscall!
