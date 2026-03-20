# Lab 2: Privilege Separation and Server-Side Sandboxing

**Course**: IIC2531 - Pontificia Universidad Católica de Chile  
**Semester**: 2026-1  
**Lab Number**: Lab 2

---

**Based on**: MIT Course 6.566 Lab 2 (Spring 2026)  
**Original URL**: https://css.csail.mit.edu/6.566/2026/labs/lab2.html  
**License**: [Creative Commons Attribution 3.0 Unported](http://creativecommons.org/licenses/by/3.0/us/)

This lab is adapted from MIT's Computer Systems Security course materials.  
All original content credit goes to the MIT CSAIL teaching staff.

---

# Lab 2: Privilege separation and server-side sandboxing

## Introduction

 This lab will introduce you to privilege separation and server-side
sandboxing, in the context of a simple python web application called
`zoobar`, where users transfer "zoobars" (credits) between each other.
The main goal of privilege separation is to ensure that if an adversary
compromises one part of an application, the adversary doesn't compromise the
other parts too. To help you privilege-separate this application, the
`zookws` web server that you already saw in the previous lab is
designed to run a web application consisting of multiple components.
If you are curious, this design is based on the [OKWS](https://okws.org/) web server, described in a
[research paper](readings/okws.pdf) and used by [okcupid.com](https://www.okcupid.com/). In a large-scale modern
system, the design would probably consist of many more moving parts, such
as using Kubernetes to run all of the components of your application,
using an RPC library like gRPC to communicate between components, etc,
but `zookws` packs all of this into a relatively simple system.

In this lab, you will set up a privilege-separated web server,
examine possible vulnerabilities, and break up the application
code into less-privileged components to minimize the effects of any
single vulnerability. The lab will use modern support for privilege
separation, [Linux containers](https://linuxcontainers.org/).
(The original OKWS design used user IDs and chroot, because Linux
containers and namespaces didn't exist at that time.)

 You will also extend the Zoobar web application to support
*executable profiles*, which allow users to use Python code as
their profiles. To make a profile, a user saves a Python program
in their profile on their Zoobar home page. (To indicate that the
profile contains Python code, the first line must be `#!python`.)
Whenever another user views the user's Python profile, the server will
execute the Python code in that user's profile to generate the resulting
profile output. This will allow users to implement a variety of features
in their profiles, such as:

- A profile that greets visitors by their user name.

- A profile that keeps track of the last several visitors to that profile.

- A profile that gives a zoobar to every visitor (limit 1 per minute).

Supporting this safely requires sandboxing the profile code on the server,
so that it cannot perform arbitrary operations or access arbitrary files.
On the other hand, this code may need to keep track of persistent data in
some files, or to access existing zoobar databases, to function properly.
You will use the remote procedure call library and some shim code that we provide to
securely sandbox executable profiles on the server using WebAssembly.

To fetch the new source code, use Git to commit your Lab 1 solutions,
and then switch to the `lab2` branch.

```
student@6566-v26:~$ cd lab
student@6566-v26:~/lab$ git status
...
student@6566-v26:~/lab$ git add exploit-*.py http.c zookd.c [and any other new files...]
student@6566-v26:~/lab$ git commit -am 'my solution to lab1'
[lab1 c54dd4d] my solution to lab1
 1 files changed, 1 insertions(+), 0 deletions(-)
student@6566-v26:~/lab$ git fetch
...
student@6566-v26:~/lab$ git checkout -b lab2 origin/lab2
Branch lab2 set up to track remote branch lab2 from origin.
Switched to a new branch 'lab2'
```

Once your source code is in place, make sure that you can compile and install
the web server and the `zoobar` application:

```
student@6566-v26:~/lab$ make
cc zookd.c -c -o zookd.o -m64 -g -std=c99 -Wall -Wno-format-overflow -D_GNU_SOURCE -static -fno-stack-protector
cc http.c -c -o http.o -m64 -g -std=c99 -Wall -Wno-format-overflow -D_GNU_SOURCE -static -fno-stack-protector
...
cc zookfs.c -c -o zookfs.o -m64 -g -std=c99 -Wall -Wno-format-overflow -D_GNU_SOURCE -static -fno-stack-protector
cc -m64 zookfs.o http.o -o zookfs
cc zookd2.c -c -o zookd2.o -m64 -g -std=c99 -Wall -Wno-format-overflow -D_GNU_SOURCE -static -fno-stack-protector
cc -m64 zookd2.o http.o -o zookd2
student@6566-v26:~/lab$
```

## Prelude: What's a zoobar?

 To understand the `zoobar` application itself, we will first examine
the `zoobar` web application code.

One of the key features of the `zoobar` application is the ability to
transfer credits between users. This feature is implemented by the script
`transfer.py`.

To get a sense what transfer does, start the `zoobar` Web site. For
lab 2, we start the web server using the `zookld.py` launcher, which is
similar to the `okld` launcher daemon in OKWS:

```
student@6566-v26:~/lab$ ./zookld.py
~base: Creating container
Unpacking the rootfs

---
You just created an Ubuntu noble amd64 (20260126_07:42) container.

To enable SSH, run: apt install openssh-server
No default root or user password are set by LXC.
~base: Configuring
... lots of output ...
main: Creating container
main: Copying files
main: Running zooksvc.py
main: zooksvc.py: dispatcher main, port 8080
main: zooksvc.py: running ['./zookd2', '3', '5']
main: zookd2: Start with 1 service(s)
main: zookd2: Dispatch ^(.*)$ for service 0
main: zookd2: Host 10.1.1.4 (link 1) service 0
main: zookd2: Port 8081 for service 0
zookfs: Creating container
zookfs: Copying files
zookfs: Running zooksvc.py
zookfs: zooksvc.py: running ['./zookfs', '8081']
echo: Creating container
echo: Copying files
echo: Running zooksvc.py
echo: zooksvc.py: running ['.//zoobar/echo-server.py', '8081']
echo: Running on port 8081
student@6566-v26:~/lab$
```

The first time you run `zookld.py` it will run for minutes,
 because it is building the `base` container, which involves
 building a Linux image and installing all the software that zoobar
 needs. Then, it builds three other
 containers: `main`, `zookfs`, and `echo`.
 Each of these containers has the contents of your `/home/student/lab`
 directory copied into `/app`, so that you can run your code inside
 the container.
 The containers themselves are stored in the `~/.local/share/lxc`
 directory, if you are curious about how the containers are implemented.
 `zookld.py`
 uses `zookconf.py` to build and configure the containers.
 If you get errors about creating or starting containers, try
 resetting the container state using the ./zookclean.py
 command and/or rebooting your VM.

You can manipulate the containers with the following commands:
 
 
- `zookld.py`: starts containers listed in `zook.conf`
 
- `zookps.py`: lists the status of the containers listed
 in `zook.conf` and the processes running on them.
 
- `zookstop.py`: stop the containers listed in `zook.conf`
 
- `zookclean.py`: deletes all containers from `~/.local/share/lxc`, for if you want to start over from
 scratch again.
 
 Each of these commands can also take the name of an individual container
 (e.g., `main`) and apply the operation to just that container.

All of these commands are wrappers around
 the [LXC Python
 API](https://linuxcontainers.org/lxc/documentation/#python). You can also run many of the LXC commands
 from [command line](https://linuxcontainers.org/lxc/manpages/);
 in particular, lxc-attach -n name will give you a
 root shell in container `name`, which you might find useful for debugging.

 Run `zookps.py` to see if your containers are running. Then, make
sure you can run the web server, and access the web site from your browser, as
follows:

```
student@6566-v26:~/lab$ ip addr show dev eth0
2: eth0: mtu 1500 qdisc fq_codel state UP group default qlen 1000
 link/ether 00:0c:29:b4:55:8e brd ff:ff:ff:ff:ff:ff
 altname enp0s3
 altname ens3
 inet 192.168.24.128/24 brd 192.168.24.255 scope global eth0
 valid_lft forever preferred_lft forever
 inet6 fe80::20c:29ff:feb4:558e/64 scope link
 valid_lft forever preferred_lft forever
```

In this particular example, you would want
to open your browser and go to `http://192.168.24.128:8888/zoobar/index.cgi/`,
or, if you are using KVM, to `http://localhost:8888/zoobar/index.cgi/`.
If you are using AWS EC2, you will need to setup port-forwarding,
[described in Step 5.5](labs/lab-aws.html), then go to 
`http://localhost:8888/zoobar/index.cgi/`
You should see the `zoobar` web site.

Note the different port, 8888 rather than 8080, in the URLs above.
This is because we actually want to connect to the `main`
container, but it's on an internal virtual network that's only reachable
from the VM itself. We configured the Linux kernel in the VM to forward
connections to port 8888 to port 8080 on the `main` container.
You can see the `iptables` rules we used to achieve this in
`/etc/rc.local` in your VM.

If you are having issues with seeing the web site, one possible cause
is that the fixes that you made in Lab 1 to fix the buffer overflow bugs
may be overly strict. If so, please fix it before continuing.

 

> Exercise 1.
In your browser, connect to the `zoobar` Web site, and create two user
accounts. Login in as one of the users, and transfer zoobars from one user to
another by clicking on the transfer link and filling out the form. Play
around with the other features too to get a feel for what it allows users to
do. In short, a registered user can update his/her profile, transfer
"zoobars" (credits) to another user, and look up the zoobar balance, profile,
and transactions of other users in the system.

Read through the code of `zoobar` and see how `transfer.py`
gets invoked when a user sends a transfer on the transfer page. A good place to
start for this part of the lab is `templates/transfer.html`,
`__init__.py`, `transfer.py`, and `bank.py` in the `zoobar` directory.

 Note: You don't need to turn in anything for this exercise, but make sure
that you understand the structure of the zoobar application--it will save you
time in the future!

## Privilege separation

 Having surveyed the zoobar application code, it is worth starting to think
about how to apply privilege separation to the `zookws` and
`zoobar` infrastructure so that bugs in the infrastructure don't allow an
adversary, for example, to transfer zoobars to the adversary account.

The web server for this lab uses containers for different parts of the web
server. The containers are defined in `zook.conf`. 
`zookld.py` uses `zookconf.py` to read `zook.conf`
(via `readconf.py`) and sets up the containers. As part of this lab
you will define new containers and modify how containers are set up. For that
purpose you need to work with LXC and you may find it useful to refer to the
documentation on [LXC](https://linuxcontainers.org/lxc/introduction/).

Two aspects make privilege separation challenging in the real world and in
 this lab. First, privilege separation requires that you take apart the
 application and split it up in separate pieces. Although we have tried to
 structure the application well so that it is easy to split, there are places
 where you must redesign certain parts to make privilege separation possible.
 Second, you must ensure that each piece runs with minimal privileges, which
 requires setting permissions precisely and configuring the pieces
 correctly. Hopefully, by the end of this lab, you'll have a better
 understanding of why many applications have security vulnerabilities related
 to failure to properly separate privileges: proper privilege separation is
 hard!

One problem that you might run into is that it's tricky to debug a complex
 application that's composed of many pieces. To help you, we have provided
 a simple debug library in `debug.py`, which is imported by every
 Python script we give you. The `debug` library provides a single
 function, `log(msg)`, which prints the message `msg` to
 stderr (which should go to the terminal where you ran `zookld`),
 along with a stack trace of where the `log` function was called from.

All output from code running in various containers that you see in your
 terminal should also be prefixed with the name of the container.
 For example, when you see main: zookd2: Forwarding to 10.1.1.4:8081
 for /zoobar/media/zoobar.css, it means the message is coming from
 the `main` container.

If something doesn't seem to be working, try to figure out what went wrong, or
contact the course staff, before proceeding further.

## Part 1: Privilege-separate the web server setup using containers

 In lab 1, `zookws` consisted of essentially one process:
 `zookd`. From the security point of view, this structure is not ideal:
 for example, any buffer overrun you found, you can use to take
 over `zookws`. For example, you can invoke the dynamic scripts with
 arguments of your choice (e.g., giving many zoobars to yourself), or simpler,
 just write the database that contains the zoobar accounts directly.

This lab refactors `zookd`
following the [OKWS](https://okws.org/) design. Similar to OKWS,
`zookws` consists of a launcher program `zookld.py` that
launches services configured in the file `zook.conf`, a
`zookd` that just routes requests to corresponding services, as well as
several services. For simplicity `zookws` does not implement helper or
logger daemon as OKWS does. You can think of `zookld.py` as a minimalist
version of Kubernetes.

 
We will run each component in a separate Linux container. Linux containers
 provide the illusion of a virtual Linux machine without using virtual machines;
 they are implemented using Linux processes. A process in a container is
 stronger isolated than standard Linux processes: a process inside a container
 has limited access to kernel names spaces, has limited access to systems calls,
 and doesn't have access to the file system. In many ways, they behave like virtual
 machines: they are started from a VM image, have their own IP address, their
 own file system, etc. You assign IP addresses to them, copy the right files
 into them, and arrange for remote procedure calls between them.

 
We will use *unprivileged* containers, which run as an unprivileged
 user process (i.e., not root). Even if the process running inside the
 container runs with root privileges, the container itself runs as an
 unprivileged user.

 
With containers, even if there is an exploit (e.g., another buffer overrun
 in `zookd`), the container running `zookd` will give the attacker
 little control. For example, taking over `zookd`, will not allow the
 attacker to invoke the dynamic scripts or write the database directly that runs
 inside another container. Furthermore, `zookd` cannot break out of its
 container and take control over the underlying Linux kernel.

The file `zook.conf` is the configuration file that specifies how
each container should run. For example, the `main` entry:

```
[main]
 cmd = zookd2
 dir = /app
 lxcbr = 0
 port = 8080
 http_svcs = zookfs
```

specifies that the command to run main is `zookd2`, in directory
`/app` on a container connected to virtual network 0
(lxcbr, short for LXC bridge), and
that `cmd` gets port 8080 to receive/send requests on.

In the lab VM that you are using, we pre-created 10 virtual networks,
`lxcbr0` through `lxcbr9`, which you can use, corresponding
to network addresses `10.1.0.*` through `10.1.9.*`
respectively (or `10.1.0.0/24` through `10.1.9.0/24` in
[CIDR](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing)
notation). Each container gets the `.4` IP address in its virtual
network's subnet; for example, if you assign some service to `lxcbr = 7`,
it will have the IP address `10.1.7.4`.

The reason for having multiple virtual networks is to provide strong network
isolation between containers. Containers connected to the same virtual network
can send each other packets directly, and a compromised container could spoof
packets from another container's IP address. On the other hand, containers
on different virtual networks must go through the host's Linux kernel to route
packets, and the Linux kernel ensures that packets coming from one virtual
network do not spoof a source address of a different virtual network (enabled
by the [`rp_filter`](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt) kernel option).

By default, each container allows incoming packets from any other
container. You will restrict this later in the lab, and the use of
separate virtual networks will ensure that a compromised container cannot
bypass these network-level restrictions.

The `zook.conf` file configures only one HTTP service (through
the line `http_svcs`), `zookfs`, that both serves static
files and executes dynamic scripts. Later on in this lab, you will need
to run multiple HTTP services, which you can do by listing all of them,
separated by comma, in the `http_svcs` line; for example,
`http_svcs = first,second,third`.

The `zookfs` service works by
invoking the executable `zookfs`, which runs in the directory
`/app` on the container connected to `lxcbr = 1`
(and thus with IP address 10.1.1.4).

The `fwrule` lines that you see commented out in the `zookfs`
service description in `zook.conf`
specify filters that control communication to that container:

```
[zookfs]
 cmd = zookfs
 url = .*
 dir = /app
 lxcbr = 1
 port = 8081
 ## Filter rules are inserted in the order they appear in this file.
 ## Thus, in the below example (commented out initially) the first
 ## filters applied are the ACCEPT ones, and then the REJECT one.
 ## Use `iptables -nvL INPUT' on the appropriate container to see all
 ## the filters that are in effect on that container.
 # fwrule = -s main -j ACCEPT
 # fwrule = -s echo -j ACCEPT
 # fwrule = -j REJECT
```

For example, the commented-out rules allow packets from the `main`
and `echo` containers, but block all other packets. You will
change these rules (and define similar rules for other containers)
as we further privilege separate zookws.

We will start to further privilege-separate the `zookfs` service that
handles both static files and dynamic scripts. Although it runs in a container,
some Python scripts could easily have security holes; a vulnerable Python script
could be tricked into deleting important static files that the server is
serving. Conversely, the static file serving code might be tricked into serving
up the databases used by the Python scripts, such as `person.db`
and `transfer.db`. A better organization is to split `zookfs`
into two services, one for static files and the other for Python scripts,
running as different users.

> 

Exercise 2.
Modify `zook.conf` to replace `zookfs` with
two separate services, `dynamic` and `static`.
Both should use `cmd = zookfs`.

`dynamic` should execute just `/zoobar/index.cgi`
(which runs all the Python scripts), but should not serve any static
files.
`static` should serve static files but not execute anything.

To remove the `zookfs` container so that `zookld.py`
doesn't start it, run ./zookclean.py zookfs.

Run the dynamic and static services on different virtual networks.

This separation requires `zookd` to determine which
service should handle a particular request.
You may use `zookws`'s URL filtering to do this,
without modifying the application or the URLs that it uses.
The URL filters are specified in `zook.conf`,
and support regular expressions.
For example, `url = .*` matches all requests, while
`url = /zoobar/(abc|def)\.html` matches requests
to `/zoobar/abc.html` and `/zoobar/def.html`.

For this exercise, you should only modify `zook.conf`; don't modify any C or
Python code.

Run make check-lab2 to verify that
your modified configuration passes our tests.

We would like to control with whom dynamic and static can communicate. For
 example, even if `static` were compromised, we want the attacker be
 unable to communicate with `dynamic`, making it harder for the attacker
 to compromise `dynamic` too. Therefore, we would like to set up firewall
 rules in each container to enforce this isolation. We will use
 [`iptables`](https://wiki.archlinux.org/index.php/iptables)
 to insert filter rules in each container, so that `static` will only
 accept packets from `main` (which runs `zookd`). This ensures
 that `dynamic` cannot send packets directly to `static`.
 Similarly, `dynamic` should only accept packets from `main`,
 and not from `static`. Setting security rules likes these is
 often referred to as network segregation.

 
> 
 Exercise 3.
 Write appropriate `fwrule` entries
 for `main`, `static`, and `dynamic` to limit the
 communication as specified above.

 
If you get the filters wrong, you may be unable to connect with any
 container. You can reset the firewall to allow all
 communication by stopping and starting the containers again,
 using `zookstop.py` followed by `zookld.py`.

Run make handin.zip and upload the resulting
`handin.zip` file to
[the submission web site](labs/handin.html).

## Interlude: RPC library
 

 In this part, you will privilege-separate the `zoobar` application
itself in several processes, running in different containers. We would like to
limit the damage from any future bugs that come up. That is, if one piece of
the `zoobar` application has an exploitable bug, we'd like to prevent an
attacker from using that bug to break into other parts of the `zoobar`
application.

A challenge in splitting the `zoobar` application into several containers
is that the processs inside containers must have a way to communicate with each
other. You will first study a Remote Procedure Call (RPC) library that allows
processes to communicate. Then, you will use that library to
separate `zoobar` into several processes, each inside its own container,
that communicate using RPC.

To illustrate how our RPC library might be used, we have implemented a simple
"echo" service for you, in `zoobar/echo-server.py`. This service is
invoked by `zookld` and runs inside its own container; look for
the `echo` section of `zook.conf` to see how it is started.

`echo-server.py` is implemented by defining an RPC class
`EchoRpcServer` that inherits from `RpcServer`, which in turn
comes from `zoobar/rpclib.py`. The `EchoRpcServer` RPC class
defines the methods that the server supports, and `rpclib` invokes those
methods when a client sends a request. The server defines a simple method that
echos the request from a client.

`echo-server.py` starts the server by calling
`run_fork(port)`. This function listens on a
TCP socket. The port comes from the argument, which in
this case is `8081` (specified in `zook.conf`).
When a client connects to this socket, the function forks the current
process. One copy of the process receives messages and responds on
the just-opened connection, while the other process listens for other
clients that might open the socket.

We have also included a simple client of this `echo` service as part
of the Zoobar web application. In particular, if you go to the
URL `/zoobar/index.cgi/echo?s=hello`, the request is routed
to `zoobar/echo.py`. That code uses the RPC client (implemented
by `rpclib`) to connect to the echo service at (host IP address,
port). The client looks up the (host IP address, port)
in `zook.conf`. The client invokes the `echo` operation. Once it
receives the response from the echo service, it returns a web page containing
the echoed response.

The RPC client-side code in `rpclib` is implemented by the
`call` method of the `RpcClient` class. This methods
formats the arguments into a string, writes the string on the connection
to the server, and waits for a response (a string). On receiving the
response, `call` parses the string, and returns the results to
the caller.

## Part 2: Privilege-separating the login service in Zoobar

We will now use the RPC library to improve the security of the user
passwords stored in the Zoobar web application. Right now, an adversary that
exploits a vulnerability in any part of the Zoobar application can obtain all
user passwords from the `person` database.

The first step towards protecting passwords will be to create a service
that deals with user passwords and cookies, so that only that service
can access them directly, and the rest of the Zoobar application cannot.
In particular, we want to separate the code that deals with user authentication
(i.e., passwords and tokens) from the rest of the application code. The current
`zoobar` application stores everything about the user (their profile,
their zoobar balance, and authentication info) in the `Person` table
(see `zoodb.py`). We want to move the authentication info out of
the `Person` table into a separate `Cred` table (Cred stands
for Credentials), and move the code that accesses this authentication
information (i.e., `auth.py`) into a separate service.

Note that it is not completely necessary to split the data into
 separate tables for security: each container would end up with its
 own copy of the database, and would have no data in the parts of the
 database that it never populated. We split up the data into separate
 tables anyway, because it helps understand how to split up the services
 correctly.

Specifically, your job will be as follows:

 
- Decide what interface your authentication service should provide
	(i.e., what functions it will run for clients). Look at the
	code in `login.py` and `auth.py`, and decide what
	needs to run in the authentication service, and what can run in
	the client (i.e., be part of the rest of the zoobar code). Keep
	in mind that your goal is to protect both passwords and tokens.
	We have provided initial RPC stubs for the client in the file
	`zoobar/auth_client.py`.

 
- Create a new `auth` service for user authentication, along the lines of
	`echo-server.py`. We have provided an initial file for
	you, `zoobar/auth-server.py`, which you should modify for
	this purpose. The implementation of this service should use the
	existing functions in `auth.py`.

 
- Modify `zook.conf` to start the `auth-server`
	appropriately (in a container on a different virtual network).

 
- Split the user credentials (i.e., passwords and tokens) from
	the `Person` database into a separate `Cred`
	database, stored in `/zoobar/db/cred`. Don't keep any
	passwords or tokens in the old `Person` database.

 
- Modify the login code in `login.py` to invoke your
	auth service instead of calling `auth.py` directly.

> 

Exercise 4.
Implement privilege separation for user authentication, as described above.

A good starting point would be to first split out the credentials
originally stored in the `Person` database into a separate
`Cred` database, but still do everything in the `dynamic`
service. Don't forget to create a regular `Person` database
entry for newly registered users.

Once that works, add explicit function calls between the code that you
expect will still live in the `dynamic` service (which does not
touch `Cred`) and the functions that you will eventually move
into `auth-server` (which does touch `Cred`). Make sure
this works with just functions first, without actually moving the code
into a separate `auth-server`.

Finally, set up the `auth-server.py` to run the code handling the
`Cred` database, and turn the function calls from the previous
step into actual RPCs to `auth-server`.

Run make check-lab2 to verify that
your privilege-separated authentication service passes our tests.

> 
 Exercise 5.
 Specify the appropriate `fwrule` entries for `auth`.

	

Now, we will further improve the security of passwords, by using
hashing and salting. The current authentication code stores an exact
copy of the user's password in the database. Thus, if an adversary
somehow gains access to the `cred.db` file, all of the user
passwords will be immediately compromised. Worse yet, if users have
the same password on multiple sites, the adversary will be able to
compromise users' accounts there too!

Hashing protects against this attack, by storing a hash of the user's
password (i.e., the result of applying a hash function to the password),
instead of the password itself. If the hash function is difficult to
invert (i.e., is a cryptographically secure hash), an adversary will
not be able to directly obtain the user's password. However, a server
can still check if a user supplied the correct password during login:
it will just hash the user's password, and check if the resulting hash
value is the same as was previously stored.

One weakness with hashing is that an adversary can build up a giant
table (called a "rainbow table"), containing the hashes of all possible
passwords. Then, if an adversary obtains someone's hashed password,
the adversary can just look it up in its giant table, and obtain the
original password.

To defeat the rainbow table attack, most systems use *salting*.
With salting, instead of storing a hash of the password, the server
stores a hash of the password concatenated with a randomly-generated
string (called a salt). To check if the password is correct, the server
concatenates the user-supplied password with the salt, and checks if the
result matches the stored hash. Note that, to make this work, the server
must store the salt value used to originally compute the salted hash!
However, because of the salt, the adversary would now have to generate
a separate rainbow table for every possible salt value. This greatly
increases the amount of work the adversary has to perform in order to
guess user passwords based on the hashes.

A final consideration is the choice of hash function. Most hash
functions, such as MD5 and SHA1, are designed to be fast. This means that
an adversary can try lots of passwords in a short period of time, which
is not what we want! Instead, you should use a special hash-like function
that is explicitly designed to be *slow*. A good example of such a
hash function is [PBKDF2](https://en.wikipedia.org/wiki/PBKDF2),
which stands for Password-Based Key Derivation Function (version 2).

> 

Exercise 6.
Implement password hashing and salting in your authentication service.
In particular, you will need to extend your `Cred` table to include
a `salt` column; modify the registration code to choose a random
salt, and to store a hash of the password together with the salt, instead
of the password itself; and modify the login code to hash the supplied
password together with the stored salt, and compare it with the stored
hash. Don't remove the password column from the `Cred` table
(the check for exercise 5 requires that it be present); you can store
the hashed password in the existing `password` column.

To implement PBKDF2 hashing, you can use the
[Python PBKDF2
module](https://www.dlitz.net/software/python-pbkdf2/). Roughly, you should `import pbkdf2`, and then hash
a password using `pbkdf2.PBKDF2(password, salt).hexread(32)`.
We have provided a copy of `pbkdf2.py` in the `zoobar`
directory. Do not use the `random.random` function to generate a salt
as [the documentation of the random module](https://docs.python.org/3/library/random.html)
states that it is not cryptographically secure. A secure alternative is
the function `os.urandom`.

Run make check-lab2 to verify that your
hashing and salting code passes our tests. Keep in mind that our tests
are not exhaustive.

A surprising side-effect of using a very computationally expensive
hash function like PBKDF2 is that an adversary can now use this
to launch denial-of-service (DoS) attacks on the server's CPU.
For example, the popular Django web framework posted a [security
advisory](https://www.djangoproject.com/weblog/2013/sep/15/security/) about this, pointing out that if an adversary tries to log
in to some account by supplying a very large password (1MB in size),
the server would spend an entire minute trying to compute PBKDF2 on that
password. Django's solution is to limit supplied passwords to at most 4KB
in size. For this lab, we do not require you to handle such DoS attacks.

> 

Challenge 1! (optional)
For extra credit, implement the
[honeywords
proposal](https://people.csail.mit.edu/rivest/pubs/JR13.pdf) from Ari Juels and Ron Rivest in your authentication service.
Consider implementing the honeychecker as a separate service running with
its own user ID. If you decide to complete this challenge, please include a
file named `honeywords.txt` in the top level of your lab submission
that gives a brief overview of your approach and solution.

## Part 3: Privilege-separating the bank in Zoobar

Finally, we want to protect the zoobar balance of each user from
adversaries that might exploit some bug in the Zoobar application.
Currently, if an adversary exploits a bug in the main Zoobar application,
they can steal anyone else's zoobars, and this would not even show up
in the `Transfer` database if we wanted to audit things later.

To improve the security of zoobar balances, our plan is similar to what
you did above in the authentication service: split the `zoobar`
balance information into a separate `Bank` database, and set
up a `bank` service, whose job it is to perform operations on the
new `Bank` database and the existing `Transfer` database.
As long as only the `bank` service can modify the `Bank`
and `Transfer` databases, bugs in the rest of the Zoobar application
should not give an adversary the ability to modify zoobar balances, and
will ensure that all transfers are correctly logged for future audits.

> 

Exercise 7.
Privilege-separate the bank logic into a separate `bank` service, along the
lines of the authentication service. Your service should implement
the `transfer` and `balance` functions, which are currently
implemented by `bank.py` and called from several places in the
rest of the application code.

You should split the `zoobar` balance information into
a separate `Bank` database (in `zoodb.py`);
implement the bank server by modifying `bank-server.py`;
add the bank service to `zook.conf`;
create client RPC stubs for invoking the bank service;
and modify the rest of the application code to invoke the RPC
stubs instead of calling `bank.py`'s functions directly.

Don't forget to handle the case of account creation, when the new user
needs to get an initial 10 zoobars. This may require you to change the
interface of the bank service.

Run make check-lab2 to verify that
your privilege-separated bank service passes our tests.

Finally, we need to fix one more problem with the bank service.
In particular, an adversary that can access the bank service (i.e.,
can send it RPC requests) can perform transfers from *anyone's*
account to their own. For example, it can steal 1 zoobar from any victim
simply by issuing a `transfer(victim, adversary, 1)` RPC request.
The problem is that the bank service has no idea who is invoking the
`transfer` operation. Some RPC libraries provide authentication,
but our RPC library is quite simple, so we have to add it explicitly.

To authenticate the caller of the `transfer` operation, we
will require the caller to supply an extra `token` argument,
which should be a valid token for the sender. The bank service should
reject transfers if the token is invalid.

> 

Exercise 8.
Add authentication to the `transfer` RPC in the bank service.
The current user's token is accessible as `g.user.token`.
How should the bank validate the supplied token?

> 

Exercise 9.
 Specify the appropriate `fwrule` entries for the `bank`
 service.

Run make handin.zip and upload the resulting
`handin.zip` file to
[the submission web site](labs/handin.html).

## Part 4: Server-side sandboxing for executable profiles

In this final part, you will implement sandboxing for executable profiles,
which is challenging because we need to isolate the profiles of different
users from one another and from the rest of the system.

As an example, here is the kind of functionality we would like to support
with executable profiles. A user could set their profile to the following
code (which you can find in `profiles/hello-user.py`):

```
#!python
import time
import api
print('Hello, *', api.call('get_visitor'), '*')
print('
Current time:', time.time())
```

Whenever someone views this user's profile in Zoobar, the server will
run this Python code, and display the resulting output as if that was
the user's profile. For instance, if user `alice` views this
profile, she might see the following result on her screen:

**Hello, * alice *

Current time: 1708447443.7460613
**

The initial implementation we provided for executable profiles runs the
profile code directly in the server's Python process, which means that
a malicious user could tamper with the server by injecting arbitrary
code through his profile.

We will use WebAssembly to run a user's executable profile in isolation,
so that one user's executable profile cannot tamper with the profiles
of other users. WebAssembly is useful here because it makes creating
a new isolated execution environment lightweight, and the creation can
be performed easily on-demand, in contrast with containers (creating
containers is somewhat resource-intensive and requires special privileges
that are not available to a container itself).

To isolate arbitrary Python code using WebAssembly, we have supplied you
with a version of the Python interpreter that can run inside a WebAssembly
module. We will then run this entire Python interpreter, plus the
user-supplied Python profile code, inside the WebAssembly module.
This version of Python is installed as
`/usr/local/share/python-3.12.0.wasm` inside your VM.

If you want to get a feel for this isolated WebAssembly Python
interpreter, you can run it from the command-line in your VM as follows:

```
student@6566-v26:~/lab$ wasmtime run -- /usr/local/share/python-3.12.0.wasm
Python 3.12.0 (tags/v3.12.0:0fb18b0, Dec 11 2023, 11:45:20) [Clang 16.0.0 ] on wasi
Type "help", "copyright", "credits" or "license" for more information.
>>>
```

The above command uses the
[`wasmtime` WebAssembly runtime](https://docs.wasmtime.dev/)
to run the Python interpreter from `/usr/local/share/python-3.12.0.wasm`.

Since executable profiles are a somewhat dangerous part of the overall
system design (we are running arbitrary code, after all), we will be
running these executable profiles in a separate container dedicated to
that purpose, much as in the earlier parts of this lab. This does not
isolate the profiles from one another, but at least provides some degree
of isolation between all of the executable profiles and the rest of the
system.

You should familiarize yourself with the `profiles/` directory,
which contains several executable profiles that you will use as examples:

 
- `profiles/hello-user.py` is a simple profile
	that prints back the name of the visitor when the
	profile code is executed, along with the current time.
 
- `profiles/visit-tracker.py` keeps track of the
	last time that each visitor looked at the profile, and
	prints out the last visit time (if any).
 
- `profiles/last-visits.py` records the last three
	visitors to the profile, and prints them out.
 
- `profiles/xfer-tracker.py` prints out the last
	zoobar transfer between the profile owner and the visitor.
 
- `profiles/granter.py` gives the visitor one zoobar.
	To make sure visitors can't quickly steal all zoobars from
	a user, this profile grants a zoobar only if the profile
	owner has some zoobars left, the visitor has less than 20
	zoobars, and it has been at least a minute since the last
	time that visitor got a zoobar from this profile.

The final piece of code is `zoobar/profile-server.py`: an
RPC server that accepts requests to run some user's profile code,
and returns the output from executing that code. When running a
profile, `profile-server.py` sets up an RPC server that
allows the profile code to get access to things outside of the
sandbox, such as the zoobar balances of different users.
The `ProfileAPIServer` implements this interface;
`profile-server.py` forks off a separate process
to run the `ProfileAPIServer`.

One challenge is that the profile service performs `rpc_xfer`
from the profile owner's account, which (depending on your design for
privilege-separating the bank in exercise 8 above) may require a token
for the owner. You cannot just add an RPC to the `auth` service
to obtain a token for the profile owner, because then any service could
ask for it; we want only the `profile` service to be able to
do this transfer. Similarly, we cannot add an RPC to the `bank`
service to do a transfer from anyone's account without a token, since
that would break the security guarantees for exercise 8.

For the purposes of this lab, we want to do the transfer from the profile
owner's account only if the request came from the `profile`
service. Thus, the `bank` service must be able to authenticate
that a request came from the `profile` service. To help you
do so, `rpcsrv.py` provides the name of the calling service in
`self.caller`, based on the IP address of the connection from
which it received the request.

To get started, you will need 
