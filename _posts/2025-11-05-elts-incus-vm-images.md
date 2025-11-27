---
layout: post
title: How to Build an Incus ELTS VM Image
tags: [misc, debian, elts, incus]
author: Carlos Henrique Lima Melara - Charles
date: 2025-10-09 00:01:09-03:00
---

In my last blog post, I've already explored why and how to create an container
image for incus using distrobuilder, so I won't repeat myself. Distrobuilder is
the default image builder for linux containers initiative and handles many
distros, but I am only interested in Debian - can I find a simpler way to build
VM images? Short answer is **yes**, long answer is you will have to read till
the end :p

## Preface

Reading incus documentation, one will learn the tool support two image formats.
One is the unified tarball image and the other is the split tarball. They are
equivalent, but the split tarball is easier to use with other image building
tools. It emcopasses a squashfs file for containers or a qcow2 disk image for
VMs and tarball containing metadata.

This is very promising since there is a myriad of ways to build disk images for
VMs and we probably can generate the metadata tarball somehow. So let's get to it!

## Generating the VM image

As I said before, there are way too many ways to generate a Debian VM image -
as is always the case in Debian - but we will focus on using debefivm-create
which is part of debvm. It creates the image for us using mmdebstrap and we
just need to pass the correct arguments for debefivm-create and mmdebstrap.

```bash
debefivm-create --architecture amd64 \
                --hostname buster-elts \
                --ssh-key ~/.ssh/id_ed25519.pub \
                --output debefivm-buster-elts.img \
                --imagesize 15G \
                --release buster \
                -- \
                --keyring=/usr/share/keyrings/debian-archive-removed-keys.gpg \
                --hook-dir=/usr/share/mmdebstrap/hooks/useradd \
                --aptopt='Apt::Install-Recommends "true"' \
                --keyring=/var/tmp/freexian-archive-key.gpg \
                --include=freexian-archive-keyring,linux-image-amd64,sudo,task-gnome-desktop \
                http://archive.debian.org/debian \
                "deb http://deb.freexian.com/extended-lts buster-lts main contrib non-free"

```

The debefivm-create arguments are pretty self-explanatory, but mmdebstrap's
ones deserve some explanation. For buster and older, we need
`debian-archive-removed-keys` and we need Freexian's keyring for elts, so make
sure to download it and place outside your home directory.

```
wcurl https://deb.freexian.com/extended-lts/archive-key.gpg -O /var/tmp/freexian-archive-key.gpg
```

I don't really need the smallest possible image, so let's be sure to install
Recommends using `--aptopt` option. Also, I want a container with a graphical
user interface, so let's include `task-gnome-desktop` and create a user by
default using the `useradd` hook. This hook creates a new user called `user`,
copies the root's authorized keys and add it to the sudo groups in case it
exists (that's why we are also including sudo). Adding the
freexian-archive-keyring is an easy way to not need to copy the previously
downloaded keyring to the image, it's handled by the package.

If everything goes well, we will have a debefivm-buster-elts.img file in the
current directory, but incus requires a qcow2 image, so we need to convert.

```bash
qemu-img convert -f raw -O qcow2  debefivm-buster-elts.{img,qcow2}
```

## Incus Metadata file

Now we have to generate a metadata compressed archive for incus. The archive should contain a metadata.yml file with a few required keys. Luckly, incus-extras package contains a tool to generate it for us, so let's use it.

```bash
$ incus-simplestreams generate-metadata incus-buster-elts-vm.tar.xz
Operating system name: Debian
Release name: buster
Variant name [default="default"]:
Architecture name: amd64
Description [default="Debian buster (default) (amd64) (202511040104)"]:
```

There we go!

## Importing and Running the Image

With both requirements ready, we can import the split tarball image giving a good alias for our comfort.

```bash
incus image import --alias debian/buster incus-buster-elts-vm.tar.xz debefivm-buster-elts.qcow2
```

And launch it with extra RAM - after all we are running a full-blown Desktop Environment -, a vga console to interact with the DE and Secure Boot disabled for reasons I'm not smart to explain (but it works ;-)

```bash
incus launch --vm \
             --config limits.memory=3GiB \
             --config security.secureboot=false \
             --console=vga \
             local:debian/buster debian-buster-elts-vm
```

## The Annoying "Error: VM agent isn't currently running"

If you try to `incus shell` or `incus file` the instance, you will get "Error: VM agent isn't currently running". That happens because distrobuilder embeds a systemd service and a script to mount, copy and run an incus agent automagically in every VM or container, but we don't have that because we are to hardcore (irony disclaimer) to use distrobuilder. So we need to add both by hand:

```bash
$ cat <<EOF >/etc/systemd/system/incus-agent.service
[Unit]
Description=Incus - agent
Documentation=https://linuxcontainers.org/incus/docs/main/
Before=multi-user.target cloud-init.target cloud-init.service cloud-init-local.service
DefaultDependencies=no

[Service]
Type=notify
WorkingDirectory=-/run/incus_agent
ExecStartPre=/lib/systemd/incus-agent-setup
ExecStart=/run/incus_agent/incus-agent
Restart=on-failure
RestartSec=5s
StartLimitInterval=60
StartLimitBurst=10
EOF
$ cat <<EOF >/lib/systemd/incus-agent-setup
#!/bin/sh
set -eu
PREFIX="/run/incus_agent"
CDROM="/dev/disk/by-id/scsi-0QEMU_QEMU_CD-ROM_incus_agent"

# Functions.
mount_virtiofs() {
    mount -t virtiofs config "${PREFIX}.mnt" >/dev/null 2>&1
}

mount_9p() {
    modprobe 9pnet_virtio >/dev/null 2>&1 || true
    mount -t 9p config "${PREFIX}.mnt" -o access=0,trans=virtio,size=1048576 >/dev/null 2>&1
}

mount_cdrom() {
    mount "${CDROM}" "${PREFIX}.mnt" >/dev/null 2>&1
}

fail() {
    # Check if we already have an agent in place.
    # This will typically be true during restart in the case of a cdrom-based setup.
    if [ -x "${PREFIX}/incus-agent" ]; then
        echo "${1}, reusing existing agent"
        exit 0
    fi

    # Cleanup and fail.
    umount -l "${PREFIX}" >/dev/null 2>&1 || true
    eject "${CDROM}" >/dev/null 2>&1 || true
    rmdir "${PREFIX}" >/dev/null 2>&1 || true
    echo "${1}, failing"

    exit 1
}

# Try getting an agent drive.
mkdir -p "${PREFIX}.mnt"
mount_9p || mount_virtiofs || mount_cdrom || fail "Couldn't mount 9p or cdrom"

# Setup the mount target.
umount -l "${PREFIX}" >/dev/null 2>&1 || true
mkdir -p "${PREFIX}"
mount -t tmpfs tmpfs "${PREFIX}" -o mode=0700,size=50M

# Copy the data.
cp -Ra "${PREFIX}.mnt/"* "${PREFIX}"

# Unmount the temporary mount.
umount "${PREFIX}.mnt"
rmdir "${PREFIX}.mnt"

# Eject the cdrom in case it's present.
eject "${CDROM}" >/dev/null 2>&1 || true

# Fix up permissions.
chown -R root:root "${PREFIX}"

# Attempt to restore SELinux labels.
restorecon -R "${PREFIX}" >/dev/null 2>&1 || true

exit 0
EOF
$ chmod u+x /lib/systemd/incus-agent-setup
$ systemctl start incus-agent
```

## Conclusion

Turns out it's not that difficult to create ELTS VM images for incus, it just requires reading and exploring. But I said **images**, right?! Well, `s/buster/stretch/` in the commands above and you will be ready to go.
