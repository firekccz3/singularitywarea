---
title: Security
sidebar: admin_docs
permalink: docs-security
folder: docs
---

Once Singularity is installed in it's default configuration you may find that there is a SETUID component installed at `$PREFIX/libexec/singularity/sexec-suid`. The purpose of this is to do the require privilege escalation necessary for Singularity to operate properly. There are a few aspects of Singularity's functionality that require escalated privileges:

1. Mounting (and looping) the Singularity container image
2. Creation of the necessary namespaces in the kernel
3. Binding host paths into the container

In general, it is impossible to implement a container system that employs the features that Singularity offers without requiring extended privileges, but if this is a concern to you, the SUID components can be disabled via either the configuration file, changing the physical permissions on the sexec-suid file, or just removing that file. Depending on the kernel you are using and what Singularity features you employ this may (or may not) be an option for you. But first a warning...

Many people (ignorantly) claim that the 'user namespace' will solve all of the implementation problems with unprivileged containers. While it does solve some, it is currently feature limited. With time this may change, but even on kernels that have a reasonable feature list implemented, it is known to be very buggy and cause kernel panics. Additionally very few distribution vendors are shipping supported kernels that include this feature. For example, Red Hat considers this a "technology preview" and is only available via a system modification, while other kernels enable it and have been trying to keep up with the bugs it has caused. But, even in it's most stable form, the user namespace does not completely alleviate the necessity of privilege escalation unless you also give up the desire to support images (#1 above).

### How do other container solutions do it?
Docker and the like implement a root owned daemon to control the bring up, teardown, and functions of the containers. Users have the ability to control the daemon via a socket (either a UNIX domain socket or network socket). Allowing users to control a root owned daemon process which has the ability to assign network addresses, bind file systems, spawn other scripts and tools, is a large problem to solve and one of the reasons why Docker is not typically used on multi-tenant HPC resources.

### Security mitigations
SUID programs are common targets for attackers because they provide a direct mechanism to gain privileged command execution. These are some of the baseline security mitigations for Singularity:

1. Keep the escalated bits within the code as simple and transparent so it can be easily audit-able
2. Check the return value of every system call, command, and check and bomb out early if anything looks weird
3. Make sure that proper permissions of files and directories are owned by root (e.g. the config must be owned by root to work)
4. Don't trust any non-root inputs (like config values) unless they have been checked and/or sanitized
5. As much IO as possible is done via the calling user (not root)
6. Put as much system administrator control into the configuration file as possible
7. Drop permissions before running any non trusted code pathways
8. Limit all user actions within the container to that single user (disable escalation of privileges within a container)
9. Even though the user owns the image, it utilizes a POSIX like file system inside so files inside the container owned by root can only be modified by root

Additionally Singularity offers a very comprehensive auditing mechanism within it's debugging output by printing UID, PID, and location of every call it is making. For example:

```
$ singularity --debug shell /tmp/Centos7.img
...
DEBUG   [U=1000,P=33160]   privilege.c:152:singularity_priv_escalate(): Temporarily escalating privileges (U=1000)
VERBOSE [U=0,P=33160]      tmp.c:79:singularity_mount_tmp()           : Mounting directory: /tmp
DEBUG   [U=0,P=33160]      privilege.c:179:singularity_priv_drop()    : Dropping privileges to UID=1000, GID=1000
DEBUG   [U=1000,P=33160]   privilege.c:191:singularity_priv_drop()    : Confirming we have correct UID/GID
...
```

In the above output you can see that we are starting as UID 1000 (U=1000) and PID 33160 and we are escalating privileges. Once privileges have been increased, Singularity can properly mount /tmp and then it immediately drops permissions back to the calling user.

For comparison, the below output is when being called with the user namespace. Notice that I am not able to use the Singularity image format, and instead I am referencing a raw directory which contains the contents of the Singularity image:

```
$ singularity --debug shell -u /tmp/Centos7/
...
DEBUG   [U=1000,P=111121]  privilege.c:142:singularity_priv_escalate(): Not escalating privileges, user namespace enabled
VERBOSE [U=1000,P=111121]  tmp.c:80:singularity_mount_tmp()           : Mounting directory: /tmp
DEBUG   [U=1000,P=111121]  privilege.c:169:singularity_priv_drop()    : Not dropping privileges, user namespace enabled
...
```
