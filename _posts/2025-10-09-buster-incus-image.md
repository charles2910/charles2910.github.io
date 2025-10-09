---
layout: post
title: How to Build an Incus Buster Image
tags: [misc, debian, elts, incus]
author: Carlos Henrique Lima Melara - Charles
date: 2025-10-09 00:01:09-03:00
---

It's always nice to have container images of Debian releases to test things,
run applications or explore a bit without polluting your host machine. From
some Brazilian friends (you know who you are ;-), I've learned the best way to
debug a problem or test a fix is spinning up an incus container, getting to it
and finding the minimum reproducer. So the combination incus + Debian is
something that I'm very used to, but the problem is there are no images for
Debian ELTS and testing security fixes to see if they actually fix the
vulnerability and don't break anything else is very important.

Well, the regular images don't materialize out of thin air, right? So we can
learn how they are made and try to generate ELTS images in the same way -
shouldn't be that difficult, right? Well, kinda ;-)

The images available by default in incus come from
[images.linuxcontainers.org](https://images.linuxcontainers.org/) and are built
by Jenkins using distrobuilder. If you follow the links, you will find the
repository containing the yaml image definitions used by distrobuilder at
[github.com/lxc/lxc-ci](https://github.com/lxc/lxc-ci). With a bit of
investigation work, a [fork](https://github.com/charles2910/lxc-ci), an incus
VM with distrobuilder installed and some magic (also called trial and error) I
was able to build a buster image! Whooray, but VM and stretch images are still
work in progress.

Anyway, I wanted to share how you can build your images and document this
process so I don't forget, so here we are...

## Building Instructions

We will use an incus trixie VM to perform the build so we don't clutter our own
machine.

```bash
incus launch images:debian/trixie <instance-name> --vm
```

Then let's hop into the machine and install the dependencies.

```bash
incus shell <instance-name>
```

And...

```bash
apt install git distrobuilder
```

Let's clone the repository with the yaml definition to build a buster
container.

```bash
git clone --branch support-debian-buster https://github.com/charles2910/lxc-ci.git
# and cd into it
cd lxc-ci
```

Then all we need is to pass the correct arguments to distrobuilder so it can build
the image. It can output the image in the current directory or in a pre-defined
place, so let's create an easy place for the images.

```bash
mkdir -p /tmp/images/buster/container
# and perform the build
distrobuilder build-incus images/debian.yaml /tmp/images/buster/container/ \
            -o image.architecture=amd64 \
            -o image.release=buster \
            -o image.variant=default  \
            -o source.url="http://archive.debian.org/debian"
```

It requires a build definition written in yaml format to perform the build. If
you are curious, check the `images/` subdir.

If all worked correctly, you should have two files in your pre-defined target
directory. In our case, `/tmp/images/buster/container/` contains:

```bash
incus.tar.xz  rootfs.squashfs
```

Let's copy it to our host so we can add the image to our incus server.

```bash
incus file pull <instance-name>/tmp/images/buster/container/incus.tar.xz .
incus file pull <instance-name>/tmp/images/buster/container/rootfs.squashfs .
# and import it as debian/10
incus image import incus.tar.xz rootfs.squashfs --alias debian/10
```

If we are lucky, we can run our Debian buster container now!

```bash
incus launch local:debian/10 <debian-buster-instance>
incus shell <debian-buster-instance>
```

Well, now all that is left is to [install Freexian's ELTS package
repository](https://www.freexian.com/lts/extended/docs/how-to-use-extended-lts/)
and update the image to get a lot of CVE fixes.

```bash
apt install --assume-yes wget
wget https://deb.freexian.com/extended-lts/archive-key.gpg -O /etc/apt/trusted.gpg.d/freexian-archive-extended-lts.gpg
cat <<EOF >/etc/apt/sources.list.d/extended-lts.list
deb http://deb.freexian.com/extended-lts buster-lts main contrib non-free
EOF
apt update
apt --assume-yes upgrade
```
