# Lab 1: Buffer overflows

## Introduction

You will do a sequence of labs in 6.566. These labs will give you practical experience with common attacks and counter-measures. To make the issues concrete, you will explore the attacks and counter-measures in the context of the zoobar web application in the following ways:

- **Lab 2:** you will improve the zoobar web application by using privilege separation, so that if one component is compromised, the adversary doesn't get control over the whole web application.
- **Lab 3:** you will explore the zoobar web application, and use buffer overflow attacks to break its security properties.
- **Lab 4:** you will improve the zoobar application against browser attacks.
- **Lab 5:** you will add HTTPS support and security key (WebAuthn) authentication.

Each lab requires you to learn a new programming language or some other piece of infrastructure. For example, in lab 3 you must become intimately familiar with certain aspects of the C language, x86 assembly language, `gdb`, etc. Detailed familiarity with many different pieces of infrastructure is needed to understand attacks and defenses in realistic situations: security weaknesses often show up in corner cases, and so you need to understand the details to craft exploits and design defenses for those corner cases. These two factors (new infrastructure and details) can make the labs time consuming. You should start early on the labs and work on them daily for some limited time (each lab has several exercises), instead of trying to do all exercises in a single shot just before the deadline. Take the time to understand the relevant details. 

Several labs ask you to design exploits. These exploits are realistic enough that you might be able to use them for a real attack, but you should *not* do so. The point of the designing exploits is to teach you how to defend against them, not how to use them---attacking computer systems is illegal and can get you into serious trouble. Don't do it.

**NOTE:** Since we re-use the same lab assignments across years, we ask that you please do not make your lab code publicly accessible (e.g., by checking your solutions into a public repository on GitHub). This helps keep the labs fair and interesting for students in future years.

## Lab infrastructure

A small change in the compiler, environment variables, or the way the program is executed can result in slightly different memory layout and code structure, thus requiring a different exploit. For this reason, this lab uses a [virtual machine](https://en.wikipedia.org/wiki/Virtual_machine) to run the vulnerable web server code.

### Downloading the VM

We have prepared a virtual machine for you containing a standard installation of [Ubuntu](https://ubuntu.com/) 24.04 Linux, which you can download [here](https://web.mit.edu/6.858/2026/6.566-standalone-v26.zip) and unpack it on your computer. To start working on this lab assignment, you'll need some way to run this virtual machine. There are a few options; if you're unsure of how to run a VM, running the VM on AWS (the last option) may be a good default plan:

- **Linux with an x86 CPU.**
  Use [KVM](https://www.linux-kvm.org/), which is built into the Linux kernel. KVM should be available through your distribution, and is preinstalled on Athena cluster computers; on Debian or Ubuntu, try `apt-get install qemu-kvm`. KVM requires hardware virtualization support in your CPU, and you must enable this support in your BIOS (which is often, but not always, the default). If you have another virtual machine monitor installed on your machine (e.g., VMware), that virtual machine monitor may grab the hardware virtualization support exclusively and prevent KVM from working.

  To start the VM with KVM, unpack the zip file you downloaded above and run `./6.566-standalone-v26.sh` from a terminal (`Ctrl+A x` to force quit). If you get a permission denied error from this script, try adding yourself to the `kvm` group with `sudo gpasswd -a \`whoami\` kvm`, then log out and log back in.

- **Windows with an x86 CPU (most of them).**
  Download [VMware Workstation](https://ist.mit.edu/vmware/workstation) from IS&T. To start the course VM using VMware, import `6.566-standalone-v26.vmdk`. Go to File > New, select "create a custom virtual machine", choose Linux > Debian 9.x 64-bit, choose Legacy BIOS, and use an existing virtual disk (and select the `6.566-standalone-v26.vmdk` file, choosing the "Take this disk away" option). Finally, click Finish to complete the setup.

- **Mac with an x86 CPU (e.g., not M1).**
  Download [UTM](https://mac.getutm.app/). For setup, click "Create a New Virtual Machine", click "Emulate", click "Other", check "Skip ISO boot", specify drive size as 24 GB, and continue through to creation. After creation, right click on your VM, click "edit", right click on "IDE Drive", click "delete", click "New" Drive, click "Import", and open the "vmdk" file from the course VM download. To add port forwarding, click "Network", switch "Network Mode" to "Emulated VLAN", switch "Network Card" to "virtio-net-pci", click the "Port Forward" dropdown, click "New", fill "22" into the 2nd box and "2222" into the 4th box.

- **For users of non-x86 computers.**
  If you are using a computer with a non-x86 processor (e.g., Mac laptops with the ARM M1 processor), you can run the virtual machine using qemu. To do this, first install [Homebrew](https://brew.sh/), then install qemu by running [brew install qemu](https://formulae.brew.sh/formula/qemu), and finally edit the 6.858-x86_64-v22.sh shell script that was part of the course VM image, and remove the -enable-kvm flag. At this point, you should be able to start the course VM by running ./6.858-x86_64-v22.sh as above.

### Logging in

You will use the `student` account in the VM for your work. The password for the `student` account is `student`. You can also get access to the `root` account in the VM using `sudo`; for example, you can install new software packages using `sudo apt-get install pkgname`.

You can either log into the virtual machine using its console, or use ssh to log into the virtual machine over the (virtual) network. The latter also lets you easily copy files into and out of the virtual machine with `scp` or `rsync`. How you access the virtual machine over the network depends on how you're running it. If you're using VMWare, you'll first have to find the virtual machine's IP address. To do so, log in on the console, run `ip addr show dev eth0`, and note the IP address listed beside `inet`. With kvm, you can use `localhost` as the IP address for ssh and HTTP. You can now log in with ssh by running the following command from your host machine: `ssh -p 2222 student@IPADDRESS`.

For security, SSH does not allow logging in over the network using a password (and, in this specific case, the password is known to everyone). To log in via SSH, you will need to set up an [SSH Key](https://www.booleanworld.com/set-ssh-keys-linux-unix-server/).

You may also find it helpful to create a host alias for your 6.566 VM in your `~/.ssh/config` file, so that you can simply run, for example, `ssh 566vm` or `scp file.txt 566vm:lab/file.txt`. To do this, add the following lines to your `~/.ssh/config` file, adjusted as needed:

```
Host 566vm
  User student
  HostName localhost
  Port 2222
```

## Getting started

The files you will need for this and subsequent labs are distributed using the [Git](https://git-scm.com/) version control system. You can also use Git to keep track of any changes you make to the initial source code. Here's an [overview of Git](https://missing.csail.mit.edu/2020/version-control/) and the [Git user's manual](https://www.kernel.org/pub/software/scm/git/docs/user-manual.html), which you may find useful.

The course Git repository is available at [https://github.com/mit-pdos/6.566-lab-2026](https://github.com/mit-pdos/6.566-lab-2026). To get the lab code, log into the VM using the `student` account and clone the source code for lab 1 as follows:

```
student@6566-v26:~$ git clone https://github.com/mit-pdos/6.566-lab-2026 lab
Cloning into 'lab'...
student@6566-v26:~$ cd lab
student@6566-v26:~/lab$
```

It's important that you clone the course repository into the `lab` directory, because the length of pathnames will matter in this lab.

---

**Basado en**: MIT Course 6.566 Lab 2 (Spring 2026)  
**URL Original**: https://css.csail.mit.edu/6.566/2026/labs/lab2.html  
**Licencia**: [Creative Commons Attribution 3.0 Unported](http://creativecommons.org/licenses/by/3.0/us/)

Este laboratorio está adaptado de los materiales del curso Computer Systems Security del MIT.  
Todo el crédito del contenido original corresponde al equipo docente de MIT CSAIL.