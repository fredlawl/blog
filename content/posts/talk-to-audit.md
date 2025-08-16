+++
date = '2025-08-10T16:18:53-05:00'
draft = false
title = 'Talk to Audit'
categories = ["learning"]
tags = ["os"]
next = true
toc = true
+++

## Audit subsystem

[Audit](https://github.com/linux-audit/audit-documentation/wiki) is a Linux kernel subsystem responsible for reporting the WHO, WHAT,
and WHEN of actions taken against the system. Audit messages can be reported
at the kernel level, or from user space.

You might be familiar with them already:

```sh
journalctl -b0 -a -g "pam" --no-pager -n4 _TRANSPORT=audit
Aug 10 16:14:11 olympus audit[35778]: USER_START pid=35778 uid=1000 auid=1000 ses=3 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='op=PAM:session_open grantors=pam_keyinit,pam_limits,pam_keyinit,pam_limits,pam_systemd,pam_unix acct="root" exe="/usr/bin/sudo" hostname=olympus addr=? terminal=/dev/pts/4 res=success'
Aug 10 16:14:11 olympus audit[35778]: USER_END pid=35778 uid=1000 auid=1000 ses=3 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='op=PAM:session_close grantors=pam_keyinit,pam_limits,pam_keyinit,pam_limits,pam_systemd,pam_unix acct="root" exe="/usr/bin/sudo" hostname=olympus addr=? terminal=/dev/pts/4 res=success'
Aug 10 16:14:11 olympus audit[35778]: CRED_DISP pid=35778 uid=1000 auid=1000 ses=3 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='op=PAM:setcred grantors=pam_env,pam_fprintd acct="root" exe="/usr/bin/sudo" hostname=olympus addr=? terminal=/dev/pts/4 res=success'
Aug 10 16:14:22 olympus audit[35804]: USER_ACCT pid=35804 uid=1000 auid=1000 ses=3 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='op=PAM:accounting grantors=pam_unix acct="fred" exe="/usr/bin/sudo" hostname=olympus addr=? terminal=/dev/pts/4 res=success'
```

[PAM](https://github.com/linux-pam/linux-pam) is a common user space module
that reports user login attempts on the system. Administrators may use this
information to aggregate and then report to the police. (Whoever does
security log monitoring)

While reading some of the logs of my own system, I was curious how PAM was
sending these messages. I'm familiar with the Linux API for sending
audit messages, but not from a user space perspective.

I assumed that [netlink](https://docs.kernel.org/userspace-api/netlink/intro.html)
is involved, because that's the underlying technology used to create audit
filters from user space. (At work, I once didn't have
[auditctl(8)](https://man7.org/linux/man-pages/man8/auditctl.8.html)
available to me early enough in the boot process to set up audit filters, so I
wrote a trivial python3 tool to setup filters)

## Reverse engineering

Before I got started, I read on the auditctl manual page that
messages can be sent via:

```sh
sudo auditctl -m "test"
...
journalctl -b0 -a -g "test" --no-pager _TRANSPORT=audit
Aug 10 16:11:20 olympus audit[35096]: USER pid=35096 uid=0 auid=1000 ses=3 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='text=test exe="/usr/bin/auditctl" hostname=olympus addr=? terminal=pts/5 res=success'
```

This can be combined with [strace(1)](https://man7.org/linux/man-pages/man1/strace.1.html):

```sh
sudo strace -z auditctl -m "test"
execve("/usr/sbin/auditctl", ["auditctl", "-m", "test"], 0x7ffc9d292958 /* 26 vars */) = 0
brk(NULL)                               = 0x55c85b5d6000
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fe894f35000
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=119847, ...}) = 0
mmap(NULL, 119847, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7fe894f17000
close(3)                                = 0
openat(AT_FDCWD, "/lib64/libaudit.so.1", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=180016, ...}) = 0
mmap(NULL, 225728, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7fe894edf000
mmap(0x7fe894eed000, 114688, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0xe000) = 0x7fe894eed000
mmap(0x7fe894f09000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x2a000) = 0x7fe894f09000
mmap(0x7fe894f0b000, 45504, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7fe894f0b000
close(3)                                = 0
openat(AT_FDCWD, "/lib64/libauparse.so.0", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=152192, ...}) = 0
mmap(NULL, 151584, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7fe894eb9000
mmap(0x7fe894ece000, 61440, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x15000) = 0x7fe894ece000
mmap(0x7fe894edd000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x23000) = 0x7fe894edd000
close(3)                                = 0
openat(AT_FDCWD, "/lib64/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\00007\0\0\0\0\0\0"..., 832) = 832
pread64(3, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 784, 64) = 784
fstat(3, {st_mode=S_IFREG|0755, st_size=2443336, ...}) = 0
pread64(3, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 784, 64) = 784
mmap(NULL, 2034736, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7fe894cc8000
mmap(0x7fe894e36000, 479232, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x16e000) = 0x7fe894e36000
mmap(0x7fe894eab000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1e2000) = 0x7fe894eab000
mmap(0x7fe894eb1000, 31792, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7fe894eb1000
close(3)                                = 0
openat(AT_FDCWD, "/lib64/libcap-ng.so.0", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=32216, ...}) = 0
mmap(NULL, 28712, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7fe894cc0000
mmap(0x7fe894cc4000, 8192, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x4000) = 0x7fe894cc4000
mmap(0x7fe894cc6000, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x6000) = 0x7fe894cc6000
mmap(0x7fe894cc7000, 40, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7fe894cc7000
close(3)                                = 0
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fe894cbe000
arch_prctl(ARCH_SET_FS, 0x7fe894cbec80) = 0
set_tid_address(0x7fe894cbef50)         = 35542
set_robust_list(0x7fe894cbef60, 24)     = 0
rseq(0x7fe894cbeb80, 0x20, 0, 0x53053053) = 0
mprotect(0x7fe894eab000, 16384, PROT_READ) = 0
mprotect(0x7fe894cc6000, 4096, PROT_READ) = 0
mprotect(0x7fe894f09000, 4096, PROT_READ) = 0
mprotect(0x7fe894edd000, 4096, PROT_READ) = 0
mprotect(0x55c8325d6000, 4096, PROT_READ) = 0
mprotect(0x7fe894f73000, 8192, PROT_READ) = 0
prlimit64(0, RLIMIT_STACK, NULL, {rlim_cur=8192*1024, rlim_max=RLIM64_INFINITY}) = 0
munmap(0x7fe894f17000, 119847)          = 0
openat(AT_FDCWD, "/proc/sys/kernel/cap_last_cap", O_RDONLY) = 3
fstatfs(3, {f_type=PROC_SUPER_MAGIC, f_bsize=4096, f_blocks=0, f_bfree=0, f_bavail=0, f_files=0, f_ffree=0, f_fsid={val=[0x17, 0]}, f_namelen=255, f_frsize=4096, f_flags=ST_VALID|ST_NOSUID|ST_NODEV|ST_NOEXEC|ST_RELATIME}) = 0
read(3, "40\n", 7)                      = 3
close(3)                                = 0
prctl(PR_CAPBSET_READ, CAP_CHOWN)       = 1
prctl(PR_GET_SECUREBITS)                = 0
prctl(PR_GET_NO_NEW_PRIVS, 0, 0, 0, 0)  = 0
prctl(PR_CAP_AMBIENT, PR_CAP_AMBIENT_IS_SET, CAP_CHOWN, 0, 0) = 0
getrandom("\x34\x8d\x0c\x77\x11\xaa\xa2\xf2", 8, GRND_NONBLOCK) = 8
brk(NULL)                               = 0x55c85b5d6000
brk(0x55c85b5f7000)                     = 0x55c85b5f7000
capget({version=0 /* _LINUX_CAPABILITY_VERSION_??? */, pid=0}, NULL) = 0
gettid()                                = 35542
capget({version=_LINUX_CAPABILITY_VERSION_3, pid=35542}, {effective=1<<CAP_CHOWN|1<<CAP_DAC_OVERRIDE|1<<CAP_DAC_READ_SEARCH|1<<CAP_FOWNER|1<<CAP_FSETID|1<<CAP_KILL|1<<CAP_SETGID|1<<CAP_SETUID|1<<CAP_SETPCAP|1<<CAP_LINUX_IMMUTABLE|1<<CAP_NET_BIND_SERVICE|1<<CAP_NET_BROADCAST|1<<CAP_NET_ADMIN|1<<CAP_NET_RAW|1<<CAP_IPC_LOCK|1<<CAP_IPC_OWNER|1<<CAP_SYS_MODULE|1<<CAP_SYS_RAWIO|1<<CAP_SYS_CHROOT|1<<CAP_SYS_PTRACE|1<<CAP_SYS_PACCT|1<<CAP_SYS_ADMIN|1<<CAP_SYS_BOOT|1<<CAP_SYS_NICE|1<<CAP_SYS_RESOURCE|1<<CAP_SYS_TIME|1<<CAP_SYS_TTY_CONFIG|1<<CAP_MKNOD|1<<CAP_LEASE|1<<CAP_AUDIT_WRITE|1<<CAP_AUDIT_CONTROL|1<<CAP_SETFCAP|1<<CAP_MAC_OVERRIDE|1<<CAP_MAC_ADMIN|1<<CAP_SYSLOG|1<<CAP_WAKE_ALARM|1<<CAP_BLOCK_SUSPEND|1<<CAP_AUDIT_READ|1<<CAP_PERFMON|1<<CAP_BPF|1<<CAP_CHECKPOINT_RESTORE, permitted=1<<CAP_CHOWN|1<<CAP_DAC_OVERRIDE|1<<CAP_DAC_READ_SEARCH|1<<CAP_FOWNER|1<<CAP_FSETID|1<<CAP_KILL|1<<CAP_SETGID|1<<CAP_SETUID|1<<CAP_SETPCAP|1<<CAP_LINUX_IMMUTABLE|1<<CAP_NET_BIND_SERVICE|1<<CAP_NET_BROADCAST|1<<CAP_NET_ADMIN|1<<CAP_NET_RAW|1<<CAP_IPC_LOCK|1<<CAP_IPC_OWNER|1<<CAP_SYS_MODULE|1<<CAP_SYS_RAWIO|1<<CAP_SYS_CHROOT|1<<CAP_SYS_PTRACE|1<<CAP_SYS_PACCT|1<<CAP_SYS_ADMIN|1<<CAP_SYS_BOOT|1<<CAP_SYS_NICE|1<<CAP_SYS_RESOURCE|1<<CAP_SYS_TIME|1<<CAP_SYS_TTY_CONFIG|1<<CAP_MKNOD|1<<CAP_LEASE|1<<CAP_AUDIT_WRITE|1<<CAP_AUDIT_CONTROL|1<<CAP_SETFCAP|1<<CAP_MAC_OVERRIDE|1<<CAP_MAC_ADMIN|1<<CAP_SYSLOG|1<<CAP_WAKE_ALARM|1<<CAP_BLOCK_SUSPEND|1<<CAP_AUDIT_READ|1<<CAP_PERFMON|1<<CAP_BPF|1<<CAP_CHECKPOINT_RESTORE, inheritable=1<<CAP_WAKE_ALARM}) = 0
openat(AT_FDCWD, "/proc/35542/status", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0444, st_size=0, ...}) = 0
read(3, "Name:\tauditctl\nUmask:\t0022\nState"..., 1024) = 1024
lseek(3, 0, SEEK_CUR)                   = 1024
lseek(3, 853, SEEK_SET)                 = 853
close(3)                                = 0
openat(AT_FDCWD, "/proc/35542/status", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0444, st_size=0, ...}) = 0
read(3, "Name:\tauditctl\nUmask:\t0022\nState"..., 1024) = 1024
lseek(3, 0, SEEK_CUR)                   = 1024
lseek(3, 878, SEEK_SET)                 = 878
close(3)                                = 0
socket(AF_NETLINK, SOCK_RAW|SOCK_CLOEXEC, NETLINK_AUDIT) = 3
readlink("/proc/self/exe", "/usr/bin/auditctl", 4096) = 17
ioctl(0, TCGETS, {c_iflag=ICRNL|IXON|IUTF8, c_oflag=NL0|CR0|TAB0|BS0|VT0|FF0|OPOST|ONLCR, c_cflag=B38400|CS8|CREAD, c_lflag=ISIG|ICANON|ECHO|ECHOE|ECHOK|IEXTEN|ECHOCTL|ECHOKE, ...}) = 0
fstat(0, {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0x5), ...}) = 0
readlink("/proc/self/fd/0", "/dev/pts/5", 31) = 10
newfstatat(AT_FDCWD, "/dev/pts/5", {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0x5), ...}, 0) = 0
newfstatat(AT_FDCWD, "/dev/pts/5", {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0x5), ...}, AT_SYMLINK_NOFOLLOW) = 0
uname({sysname="Linux", nodename="olympus", ...}) = 0
sendto(3, [{nlmsg_len=104, nlmsg_type=0x3ed /* NLMSG_??? */, nlmsg_flags=NLM_F_REQUEST|NLM_F_ACK, nlmsg_seq=1, nlmsg_pid=0}, "\x74\x65\x78\x74\x3d\x74\x65\x73\x74\x20\x65\x78\x65\x3d\x22\x2f\x75\x73\x72\x2f\x62\x69\x6e\x2f\x61\x75\x64\x69\x74\x63\x74\x6c"...], 104, 0, {sa_family=AF_NETLINK, nl_pid=0, nl_groups=00000000}, 12) = 104
poll([{fd=3, events=POLLIN}], 1, 500)   = 1 ([{fd=3, revents=POLLIN}])
recvfrom(3, [{nlmsg_len=36, nlmsg_type=NLMSG_ERROR, nlmsg_flags=NLM_F_CAPPED, nlmsg_seq=1, nlmsg_pid=35542}, {error=0, msg={nlmsg_len=104, nlmsg_type=AUDIT_USER, nlmsg_flags=NLM_F_REQUEST|NLM_F_ACK, nlmsg_seq=1, nlmsg_pid=0}}], 8988, MSG_PEEK|MSG_DONTWAIT, {sa_family=AF_NETLINK, nl_pid=0, nl_groups=00000000}, [12]) = 36
recvfrom(3, [{nlmsg_len=36, nlmsg_type=NLMSG_ERROR, nlmsg_flags=NLM_F_CAPPED, nlmsg_seq=1, nlmsg_pid=35542}, {error=0, msg={nlmsg_len=104, nlmsg_type=AUDIT_USER, nlmsg_flags=NLM_F_REQUEST|NLM_F_ACK, nlmsg_seq=1, nlmsg_pid=0}}], 8988, MSG_DONTWAIT, {sa_family=AF_NETLINK, nl_pid=0, nl_groups=00000000}, [12]) = 36
close(3)                                = 0
+++ exited with 0 +++
```

This output is a bit unreadable, but I found what I was looking for. A
call to [sendto(2)](https://man7.org/linux/man-pages/man2/send.2.html). I then
refined this to get some thing more usable:

```sh
sudo strace -z -s 256 -e sendto auditctl -m "test"
sendto(3, [{nlmsg_len=104, nlmsg_type=0x3ed /* NLMSG_??? */, nlmsg_flags=NLM_F_REQUEST|NLM_F_ACK, nlmsg_seq=1, nlmsg_pid=0}, "\x74\x65\x78\x74\x3d\x74\x65\x73\x74\x20\x65\x78\x65\x3d\x22\x2f\x75\x73\x72\x2f\x62\x69\x6e\x2f\x61\x75\x64\x69\x74\x63\x74\x6c\x22\x20\x68\x6f\x73\x74\x6e\x61\x6d\x65\x3d\x6f\x6c\x79\x6d\x70\x75\x73\x20\x61\x64\x64\x72\x3d\x3f\x20\x74\x65\x72\x6d\x69\x6e\x61\x6c\x3d\x70\x74\x73\x2f\x35\x20\x72\x65\x73\x3d\x73\x75\x63\x63\x65\x73\x73\x00\x00\x00\x00"], 104, 0, {sa_family=AF_NETLINK, nl_pid=0, nl_groups=00000000}, 12) = 104
+++ exited with 0 +++
```

While strace does have the ability to actually show strings sometimes, in this
case I just have the hex, so I need to do more processing with [xxd(1)](https://linux.die.net/man/1/xxd):

```sh
echo -n "\x74\x65\x78\x74\x3d\x74\x65\x73\x74\x20\x65\x78\x65\x3d\x22\x2f\x75\x73\x72\x2f\x62\x69\x6e\x2f\x61\x75\x64\x69\x74\x63\x74\x6c\x22\x20\x68\x6f\x73\x74\x6e\x61\x6d\x65\x3d\x6f\x6c\x79\x6d\x70\x75\x73\x20\x61\x64\x64\x72\x3d\x3f\x20\x74\x65\x72\x6d\x69\x6e\x61\x6c\x3d\x70\x74\x73\x2f\x35\x20\x72\x65\x73\x3d\x73\x75\x63\x63\x65\x73\x73\x00\x00\x00\x00" | xxd
00000000: 7465 7874 3d74 6573 7420 6578 653d 222f  text=test exe="/
00000010: 7573 722f 6269 6e2f 6175 6469 7463 746c  usr/bin/auditctl
00000020: 2220 686f 7374 6e61 6d65 3d6f 6c79 6d70  " hostname=olymp
00000030: 7573 2061 6464 723d 3f20 7465 726d 696e  us addr=? termin
00000040: 616c 3d70 7473 2f35 2072 6573 3d73 7563  al=pts/5 res=suc
00000050: 6365 7373 0000 0000                      cess....
```

Overall this seems relatively simple to setup. All we need to do is establish
a netlink socket, and then send this trivial payload. The rest of the original
strace output deals with handling messages coming back from the socket.

## Code time

_talk-to-audit.py_

```python
import socket
import struct
import os

# https://elixir.bootlin.com/linux/v6.15.9/source/include/uapi/linux/netlink.h
NETLINK_AUDIT = 9

NLM_F_REQUEST = 1
NLM_F_ACK = 4

# https://elixir.bootlin.com/linux/v6.15.9/source/include/uapi/linux/audit.h#L348
AUDIT_USER = 0x3ED  # It's ironic this is set to deprecated when auditctl uses it

payload_fmt = "=LHHLL"

msg = b'text="hello, from python"'
payload_len = len(msg) + struct.calcsize(payload_fmt)

payload_hdr = struct.pack(
    payload_fmt + f"{len(msg)}s",
    payload_len,
    AUDIT_USER,
    NLM_F_ACK | NLM_F_REQUEST,
    1,
    os.getpid(),
    msg,
)

s = socket.socket(
    socket.AF_NETLINK, socket.SOCK_RAW | socket.SOCK_CLOEXEC, NETLINK_AUDIT
)
s.send(payload_hdr)
# Reader excercise to handle recev messages back
s.close()
```

And to test:

```sh
sudo python3 ./talk-to-audit.py
...
journalctl -b0 -a -g "hello, from python" --no-pager _TRANSPORT=audit
Aug 10 18:02:43 olympus audit[53387]: USER pid=53387 uid=0 auid=1000 ses=3 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='text="hello, from python'
```

And that's that to send messages into audit!
