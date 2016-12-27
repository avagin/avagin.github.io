# Upgrade a Travis VM to Ubuntu 16.04 on the fly

Today I found a challenge for myself. Some Travis users want to use Ubuntu
16.04, but for now Ubuntu 14.04 is the only one available (see
[https://github.com/travis-ci/travis-ci/issues/5821](https://github.com/travis-ci/travis-ci/issues/5821)).
Initially I thought I’d
spend one hour on this, but it ended up being a whole day project, so I decided
to share my experience.

From the first sight, it’s trivial to update all the packages:

```
sed -i -e "s/trusty/xenial/g" /etc/apt/sources.list
apt-get update && apt-get dist-upgrade -y
```

But when a system needs a reboot. The problem is, once rebooted, Travis is
losing the VM -- it runs a control process via ssh, and this process is
not supposed to be stopped. So, the key to success is to find a way to not
destroy this ssh session with Travis control process. This is where CRIU can
help us! We can try to checkpoint the ssh session, reboot the system, and
restore the session back. Is it challenging? As usual, the devil is in details.

First of all, we need to get the PID of the root process in the session. Here is
the process tree:

```
12253 ?        Ss     0:03 /usr/sbin/sshd -D
32443 ?        Ss     0:00  \_ sshd: root@pts/0
32539 pts/0    Ss     0:00  |   \_ -bash
```

To find the relevant process, we need to enumerate all the parent processes, and
get a second sshd process from the init:

```bash
ppid=""
pid=$$
while :; do
    p=$(awk '/^PPid:/ { print $2 }' /proc/$pid/status)
    test “$p” -eq 1 && break
    ppid=$pid
    pid=$p
done
echo $pid
```

Now we are ready to dump the travis session:

```
./criu/criu dump -D /imgs -o dump.log -t $pid --tcp-established \
    --ext-unix-sk -v4 --file-locks --link-remap
```

The ``--tcp-established`` option is required because Travis is connected to the VM
via ssh. The ``--link-remap`` is required to restore unlinked files. We are going to
update all the packages in the system, but the Travis process will continue to use
old files (shared libraries and any other opened files), so those removed but opened
files have to be restored as ghost files. The ``--ext-unix-sk`` is
required to handle the dbus socket (more on that later).

We are going to dump a TCP connection, and for that we need to block all the
network packets of this connection until a Travis process is restored. CRIU dump
will add a few iptables rules for this, and we have to restore them back
once a system is booted into a new kernel.

```
cat > /etc/network/if-pre-up.d/iptablesload << EOF
#!/bin/sh
iptables-restore < /etc/iptables.rules
unlink /etc/network/if-pre-up.d/iptablesload
unlink /etc/iptables.rules
exit 0
EOF

chmod +x /etc/network/if-pre-up.d/iptablesload
iptables-save -c > /etc/iptables.rules
```

Next, we need to create a new service to restore the Travis ssh session once the
system is rebooted:

```
cat > /lib/systemd/system/crtr.service << EOF
[Unit]
Description=Restore a Travis process

[Service]
Type=idle
ExecStart=/root/criu/scripts/travis/kexec-restore.sh $d $f

[Install]
WantedBy=multi-user.target
EOF
```

Now we are finally ready to reboot the system:

```
kernel=$(ls /boot/vmlinuz* | tail -n 1 | sed 's/.*vmlinuz-\(.*\)/\1/')
echo $kernel
kexec -l /boot/vmlinuz-$kernel --initrd=/boot/initrd.img-$kernel --reuse-cmdline
```

The new system uses systemd and executes more processes than a previous one. Can
it be a problem? CRIU can restore processes only with the same PIDs, but in the
new system some of them can already be used by other processes. A workaround is
to restore processes in a new PID namespace, and switch back into the root
PID namespace later by using the nsenter tool.

```
unshare -pfm --mount-proc --propagation=private ./criu/criu restore \
-D /imgs -o restore.log -j --tcp-established --ext-unix-sk \
-v4 -l --link-remap &
```

It is time to run this job, isn’t it? No, I don’t think so. Something may fail
in between dump and restore, and as long as we have logs it is simple to
investigate. So, we need to find a way to upload the logs to some external
resource. Dropbox allows to use 2Gb for free, and it has a good python API, so
let’s use it:

```python
#!/usr/bin/env python2
import dropbox, sys, os
access_token = os.getenv("DROPBOX_TOKEN")
client = dropbox.client.DropboxClient(access_token)
f = open(sys.argv[1])
fname = os.path.basename(sys.argv[1])
response = client.put_file(fname, f)
print 'uploaded: ', response
print "====================="
print client.share(fname)['url']
print "====================="
```

Now we are ready to test our job. How big is a chance for it to be successful
from the first try? You are right, it is near zero. We’ve got the first fail.
CRIU returned an error, saying there is an external stream unix socket which is
connected to /run/dbus/system_bus_socket. I don’t know how to handle this socket
properly, but we can create a new stream socket and connect it to this name,
hoping this will be sufficient. For that, we have to patch CRIU:

```patch
diff --git a/criu/sk-unix.c b/criu/sk-unix.c
index 5cbe07a..f856552 100644
--- a/criu/sk-unix.c
+++ b/criu/sk-unix.c
@@ -708,5 +708,4 @@ static int dump_external_sockets(struct unix_sk_desc *peer)
                                if (peer->type != SOCK_DGRAM) {
                                        show_one_unix("Ext stream not supported", peer);
                                        pr_err("Can't dump half of stream unix connection.\n");
-                                       return -1;
                                }
```

Let’s recompile CRIU with this patch and try again! Surely, here’s the second
error. Now CRIU restore could not restore a PTY pair with a required index, as
this is in use by someone else. How to fix this problem? We can try to mount a
new devpts with the newinstance options, but it was deprecated in new kernels:

>    - The newinstance mount option continues to be accepted but is now
>       Ignored.
> // Eric W. Biederman

Woo hoo! Now it’s time to patch CRIU image files. We need to change PTY indexes
in tty-info.img and also fix paths to terminals in reg-files.img. For image
editing, let’s use CRIT tool, which can decode binary CRIU images to JSON, and
encode from JSON back to binary. For simple JSON editing, sed is sufficient.

```
./crit/crit show /imgs/tty-info.img  | \
    sed 's/"index": \([0-9]*\)/"index": 1\1/' | \
    ./crit/crit encode > /imgs/tty-info.img.new
./crit/crit show /imgs/reg-files.img  | \
    sed 's|/dev/pts/\([0-9]*\)|/dev/pts/1\1|' | \
    ./crit/crit encode > /imgs/reg-files.img.new
```

With this in place, let’s try again for the third time. Now we find an external
FIFO in /run/systemd/sessions. I know absolutely nothing about it and how it is
used, but when a new kernel is booted, we need to create this FIFO so restore
won’t block.

```
f=$(lsof -p $1 | grep /run/systemd/sessions | awk '{ print $9 }')
...
criu dump
kexec
mkfifo $f
criu restore
```

And now we try for the fourth time and get yet another error: CRIU restore
failed because sys_prctl(PR_SET_MM, PR_SET_MM_MAP, …) returned EACCES.
According to the man page, it means that .exe_fd points to a non-executable
file. This means that /proc/pid/exe symlink destination is non-executable during
the dump. What is the root cause of this problem? My guess, as the system was
updated, dpkg changed permissions for old binaries files before removing them.
Let’s try to check this:

```
# strace -e chmod,link,unlink -f apt-get install --reinstall sudo
...
3331  link("/usr/bin/sudo", "/usr/bin/sudo.dpkg-tmp") = 0
3331  chmod("/usr/bin/sudo.dpkg-tmp", 0600) = 0
3331  unlink("/usr/bin/sudo.dpkg-tmp")  = 0
...
```

Indeed, dpkg drops the execution bit. It’s time for another CRIU patch:

```
diff --git a/criu/cr-restore.c b/criu/cr-restore.c
index 12f13ae..39277cf 100644
--- a/criu/cr-restore.c
+++ b/criu/cr-restore.c
@@ -2278,6 +2278,23 @@ static int prepare_mm(pid_t pid, struct task_restore_args *args)
        if (exe_fd < 0)
                goto out;
 
+       {
+               struct stat st;
+
+               if (fstat(exe_fd, &st)) {
+                       pr_perror("Unable to stat a file");
+                       return -1;
+               }
+
+               if (!(st.st_mode & (S_IXUSR | S_IXGRP | S_IXOTH))) {
+                       pr_debug("Add the execution bit for %d (st_mode %o)\n", exe_fd, st.st_mode);
+                       if (fchmod(exe_fd, st.st_mode | S_IXUSR)) {
+                               pr_perror("Unable to add the execution bit");
+                               return -1;
+                       }
+               }
+       }
+
        args->fd_exe_link = exe_fd;
        ret = 0;
 out:
```

Now we can open a bottle of fine champagne, pour a glass, and watch how our
Travis job works for the first time!!! See, it works:

[https://travis-ci.org/avagin/criu/builds/181822758](https://travis-ci.org/avagin/criu/builds/181822758)

Well, to tell you the truth it was a very brief version of the story. In
reality, I tried 33 times before it worked. Anyway, let’s drink to the happy
ending!

---
Authored by: @avagin and [@kolyshkin](https://github.com/kolyshkin)
