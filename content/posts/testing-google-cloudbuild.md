---
title: "Using Google Cloud Build To Build Linux Desktop Disk Images"
date: 2021-10-05T20:00:00-05:00
draft: false
---

For the SCUSA project, we need to ship bootable Linux desktop images to all of the sites that they
can use to easily convert a public computer lab into a contest environment. We have varied what we
have shipped over the years but the last few we have opted for a LiveCD-based environment since it
is minimally impacting to an existing computer lab. Read more about 
[SCUSA disk images]({{< ref "/posts/scusa-disk-image" >}}).

<!--more-->

In order to build the image we use `livemedia-creator` which is used upstream in Fedora to build
the LiveCD environments for that project. I got started in systems administration building
Fedora/CentOS servers with PXE-booted Anaconda/Kickstart and so it is nice to see anaconda make it's
way into the [livemedia-creator](https://weldr.io/lorax/livemedia-creator.html) project as well.

Although there has been a lot of good work done to bring Anaconda to regular systems, I observed it
leaving artifacts on the system if the process exits abnormally. Because of this, I was using a
combination of Vagrant and Ansible to run `livemedia-creator` in a clean environment for every build.

While using Vagrant with a local VirtualBox instance on my laptop worked, the image build process was
taking so long that I would frequently leave my laptop and it would go to sleep and mess up the Ansible
process. To fix this, I experimented with Vagrant using a remote libvirt+ssh instance on one of my local
servers which worked but was still cumbersome to set up and manage over time.

In looking to further refine this process I started to experiment with Vagrant + Google Compute
Engine plugin since I use GCE for hosting other aspects of the SCUSA infrastructure. I ran across
Google Cloud Build and looked into if it would serve my needs better.

Requirements:

- Be able to run `livemedia-creator` which requires mount privileges (SYS_ADMIN capabilities)
- Clean build environment
- "Fire and forget" functionality where I can kick off a build and walk away
- Composable build process to split the layers of configuration and compose them into the final
  product


Basic steps of the image build process:

- Use `livemedia-creator` which runs `anaconda` to build a image and outputs a tarball
- Extract tarball to temporary directory and create a SquashFS filesystem
- Create a raw disk image with partitions and configure grub to boot
- Add kernel, initrd, SquashFS to disk image

Downstream the disk image can be directly written to a USB drive for booting during contest

## Cloud Build Configuration

Set up with a simple Cloud Build configuration to build my builder image:

```yaml
steps:
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build',
           '-t', 'gcr.io/$PROJECT_ID/scusa-image-builder:latest',
           '.']

images:
  - 'gcr.io/$PROJECT_ID/scusa-image-builder:latest'
```

Second Cloud Build configuration that runs that builder image to build my image:

```yaml
steps:
- name: 'gcr.io/cloud-builders/docker'
  args:
  - run
  - --privileged
  - --tty
  - --name
  - build
  - --volume
  - /workspace:/workspace
  - gcr.io/$PROJECT_ID/scusa-image-builder:latest
  - /workspace/centos8.ks

timeout: 3600s

artifacts:
  objects:
    location: 'gs://[STORAGE_LOCATION]/'
    paths: ['output/disk.tar.gz']
```

I have to run docker with `--privileged` because throughout the build process we are mounting
filesystems

## First Issue: Immediate Exit from Missing DBUS

This was an issue in just porting the workflow to containers and was an issue in my initial testing
on my local docker instance.

I found that Anaconda seemed to be selecting the RHEL profile (not sure why?) and was therefore
attempting to instantiate the Subscription module. By removing it I was able to continue:

```bash
sed -i '/.*Subscription/d' /etc/anaconda/product.d/rhel.conf
```

## Another Issue: Syslog in Containers

It was clear from the command line arguments that `livemedia-creator` and `anaconda` wanted to log
to files or syslog. In order to debug my problems I really needed stdout/stderr since that was the
easiest to get from the cloud builder. I was about to reach for rsyslog/fluentd or something of that
nature but after some quick googling I stumbled upon the excellent
[ossobv/syslog2stdout](https://github.com/ossobv/syslog2stdout)

```dockerfile
RUN git clone https://github.com/ossobv/syslog2stdout /opt/syslog2stdout && \
    cd /opt/syslog2stdout && \
    cc -O3 -o /bin/syslog2stdout syslog2stdout.c
```

And in my entrypoint builder script:

```bash
syslog2stdout /dev/log &
```

## Debugging Anaconda Storage module

After some iteration I was able to successfully build an image with local Docker but when I tried to
run it with Cloud Build I noticed it would hang trying to activate all of the modules.

```log
2021-10-07 11:59:56,572 INFO pylorax: Running anaconda.
daemon.info: dbus-daemon[15]: Activating service name='org.fedoraproject.Anaconda.Boss' requested by ':1.0' (uid=0 pid=11 comm="/usr/libexec/platform-python /usr/sbin/anaconda --" label="unconfined")
daemon.info: dbus-daemon[15]: Successfully activated service 'org.fedoraproject.Anaconda.Boss'
daemon.info: dbus-daemon[15]: Activating service name='org.fedoraproject.Anaconda.Modules.Timezone' requested by ':1.1' (uid=0 pid=20 comm="/usr/libexec/platform-python -m pyanaconda.modules" label="unconfined")
daemon.info: dbus-daemon[15]: Activating service name='org.fedoraproject.Anaconda.Modules.Network' requested by ':1.1' (uid=0 pid=20 comm="/usr/libexec/platform-python -m pyanaconda.modules" label="unconfined")
daemon.info: dbus-daemon[15]: Activating service name='org.fedoraproject.Anaconda.Modules.Localization' requested by ':1.1' (uid=0 pid=20 comm="/usr/libexec/platform-python -m pyanaconda.modules" label="unconfined")
daemon.info: dbus-daemon[15]: Activating service name='org.fedoraproject.Anaconda.Modules.Security' requested by ':1.1' (uid=0 pid=20 comm="/usr/libexec/platform-python -m pyanaconda.modules" label="unconfined")
daemon.info: dbus-daemon[15]: Activating service name='org.fedoraproject.Anaconda.Modules.Users' requested by ':1.1' (uid=0 pid=20 comm="/usr/libexec/platform-python -m pyanaconda.modules" label="unconfined")
daemon.info: dbus-daemon[15]: Activating service name='org.fedoraproject.Anaconda.Modules.Payloads' requested by ':1.1' (uid=0 pid=20 comm="/usr/libexec/platform-python -m pyanaconda.modules" label="unconfined")
daemon.info: dbus-daemon[15]: Activating service name='org.fedoraproject.Anaconda.Modules.Storage' requested by ':1.1' (uid=0 pid=20 comm="/usr/libexec/platform-python -m pyanaconda.modules" label="unconfined")
daemon.info: dbus-daemon[15]: Activating service name='org.fedoraproject.Anaconda.Modules.Services' requested by ':1.1' (uid=0 pid=20 comm="/usr/libexec/platform-python -m pyanaconda.modules" label="unconfined")
daemon.info: dbus-daemon[15]: Successfully activated service 'org.fedoraproject.Anaconda.Modules.Security'
daemon.info: dbus-daemon[15]: Successfully activated service 'org.fedoraproject.Anaconda.Modules.Services'
daemon.info: dbus-daemon[15]: Successfully activated service 'org.fedoraproject.Anaconda.Modules.Payloads'
daemon.info: dbus-daemon[15]: Successfully activated service 'org.fedoraproject.Anaconda.Modules.Network'
daemon.info: dbus-daemon[15]: Successfully activated service 'org.fedoraproject.Anaconda.Modules.Users'
daemon.info: dbus-daemon[15]: Successfully activated service 'org.fedoraproject.Anaconda.Modules.Timezone'
daemon.info: dbus-daemon[15]: Successfully activated service 'org.fedoraproject.Anaconda.Modules.Localization'
```

We are missing full activation of `org.fedoraproject.Anaconda.Modules.Storage`

`strace` is my friend:

```bash
strace -f -s 1024 -e fork,execve,open,listen,connect livemedia-creator ...
```

On local:

```log
strace: Process 87 attached
daemon.info: dbus-daemon[19]: Successfully activated service 'org.fedoraproject.Anaconda.Modules.Storage'
```

On cloud build:

```log
strace: Process 86 attached
[pid    86] execve("/usr/local/sbin/modprobe", ["modprobe", "--dry-run", "xfs"], 0x7f911725fc90 /* 21 vars */) = -1 ENOENT (No such file or directory)
[pid    86] execve("/usr/local/bin/modprobe", ["modprobe", "--dry-run", "xfs"], 0x7f911725fc90 /* 21 vars */) = -1 ENOENT (No such file or directory)
[pid    86] execve("/usr/sbin/modprobe", ["modprobe", "--dry-run", "xfs"], 0x7f911725fc90 /* 21 vars */) = 0
[pid    86] +++ exited with 1 +++
```

I went back to the configuration for anaconda in `/etc/anaconda/product.d`. It wasn't entirely clear
to me which profile was being selected but I also saw the profiles were setting `file_system_type =
xfs`.

We don't have xfs in `/proc/filesystems` on the cloud builder but we do on local docker.

Finally just did a strace without filtering the syscalls and found this write call. Not sure why
this traceback wasn't making it all the way back to logs but it was definitely the culprit. I have
seen issues when doing multi-threaded python where logging to stdout/stderr gets lost.

```log
[pid    51] write(2, "Traceback (most recent call last):
  File \"/usr/lib64/python3.6/runpy.py\", line 193, in _run_module_as_main
    \"__main__\", mod_spec)
  File \"/usr/lib64/python3.6/runpy.py\", line 85, in _run_code
    exec(code, run_globals)
  File \"/usr/lib64/python3.6/site-packages/pyanaconda/modules/storage/__main__.py\", line 29, in <module>
    service = StorageService()
  File \"/usr/lib64/python3.6/site-packages/pyanaconda/modules/storage/storage.py\", line 138, in __init__
    self._set_storage(create_storage())
  File \"/usr/lib64/python3.6/site-packages/pyanaconda/modules/storage/devicetree/model.py\", line 50, in create_storage
    return InstallerStorage()
  File \"/usr/lib/python3.6/site-packages/blivet/threads.py\", line 53, in run_with_lock
    return m(*args, **kwargs)
  File \"/usr/lib64/python3.6/site-packages/pyanaconda/modules/storage/devicetree/model.py\", line 67, in __init__
    self.set_default_fstype(conf.storage.file_system_type or self.default_fstype)
  File \"/usr/lib/python3.6/site-packages/blivet/threads.py\", line 53, in run_with_lock
    return m(*args, **kwargs)
  File \"/usr/lib/python3.6/site-packages/blivet/blivet.py\", line 1069, in set_default_fstype
    self._check_valid_fstype(newtype)
  File \"/usr/lib/python3.6/site-packages/blivet/threads.py\", line 53, in run_with_lock
    return m(*args, **kwargs)
  File \"/usr/lib/python3.6/site-packages/blivet/blivet.py\", line 1054, in _check_valid_fstype
    raise ValueError(\"new value %s is not valid as a default fs type\" % newtype)
ValueError: new value tmpfs is not valid as a default fs type
", 1568) = 1568
```

After I changed the configuration `file_system_type = ext4` I was able to complete my build with Cloud
Build. Woo hoo!

## Conclusion

Overall Cloud Build is super easy to use and efficient for my use case. I wanted something that I
could kick off and just let run without having to keep my laptop open and awake. At ~40 min for a
build it was just too long to sit and wait for it to finish. Managing all of this in docker
containers is significantly simpler than managing spinning up VMs either in Vagrant (local or with
remote libvirt+ssh) or in Google Compute Engine (with Ansible).
