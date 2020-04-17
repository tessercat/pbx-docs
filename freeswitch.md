Documentation to build
FreeSWITCH binaries
for the verto project.


# Check release notes

    https://github.com/signalwire/freeswitch/releases


# Update and reboot build host

    ssh root@<build-host> -i <host-private-key>

    apt -y update
    apt -y full-upgrade && reboot


# Prep to build

Install stuff.

    apt -y install git


Configure stuff.

    update-alternatives --config editor

Set version vars.

    export CURRENT=v1.10.X
    export VERSION=v1.10.Y

and do one of the following.

## Prep to build amd64

From
[Debian 10 Buster](https://freeswitch.org/confluence/display/FREESWITCH/Debian+10+Buster).

    export ARCH=amd64
    apt -y install gnupg2 wget lsb-release


    wget -O - https://files.freeswitch.org/repo/deb/debian-release/fsstretch-archive-keyring.asc | apt-key add -
    echo "deb http://files.freeswitch.org/repo/deb/debian-release/ `lsb_release -sc` main" > /etc/apt/sources.list.d/freeswitch.list
    echo "deb-src http://files.freeswitch.org/repo/deb/debian-release/ `lsb_release -sc` main" >> /etc/apt/sources.list.d/freeswitch.list

## Prep to build arm32

From
[Raspberry Pi](https://freeswitch.org/confluence/display/FREESWITCH/Raspberry+Pi).

    export ARCH=arm32
    apt -y install gnupg2 wget lsb-release


    wget -O - https://files.freeswitch.org/repo/deb/rpi/debian-dev/freeswitch_archive_g0.pub | apt-key add -
    echo "deb http://files.freeswitch.org/repo/deb/rpi/debian-dev/ `lsb_release -sc` main" > /etc/apt/sources.list.d/freeswitch.list
    echo "deb-src http://files.freeswitch.org/repo/deb/rpi/debian-dev/ `lsb_release -sc` main" >> /etc/apt/sources.list.d/freeswitch.list


# Build FreeSWITCH

## Clone current binaries

    git clone -b $CURRENT --depth=1 https://github.com/tessercat/verto-$ARCH.git /opt/verto/freeswitch

## Install build deps

    apt -y update
    apt -y build-dep freeswitch

## Clone source

    cd /usr/local/src
    git clone --depth=1 https://github.com/signalwire/freeswitch.git -bv1.10 freeswitch
    cd freeswitch
    git log

## Compile and install

    cp /opt/verto/freeswitch/conf/modules.conf .
    ./bootstrap.sh -j
    ./configure --prefix=/opt/verto/freeswitch --disable-fhs
    make
    make install


# Snapshot changes

Inspect changes to `/opt/verto/freeswitch`
and create and push a new version branch.

    cd /opt/verto/freeswitch
    git checkout -b $VERSION
    git add .
    git commit -m "Update to $VERSION."
    git push origin $VERSION


# Test changes


# Release changes
