# crossbuilder
A debian package cross building tool using LXD

Crossbuilder aims at making cross compiling code and deploying projects to devices simple, reliable and most importantly fast. It uses LXD containers and ccache by default.

Initially developed in: https://launchpad.net/crossbuilder

To use it, clone the repository and just run it from there:
```bash
cd crossbuilder
./crossbuilder help
```

To build and deploy your project on the device connected to your computer all in one go change into the project folder an run crossbuilder (common parameters applied, change as needed):
```bash
cd yourproject/
crossbuilder --arch=arm64 --lxd-image=ubuntu:24.04 --password=????
```

## architecture

Crossbuilder by default builds for armhf architecture. If a device is connected, it should detect the devices architecture. In case for some reason the architecture needs to be explicitely specified, it can be set like this:
```bash
crossbuilder --architecture=arm64
```

## device password

To be able to install the deb on the device after building, the devices sudo password needs to be provided. The build-in parameter option (`crossbuilder --password=PASSWORD`) is NOT RECOMMENDED since this command - including your password - will stay in your shell history.

Rather use env vars either for providing the password or to read it from a file. Crossbuilder supports `DEVICE_PASSWORD` and `DEVICE_PASSWORD_FILE` for this purpose.

```bash
# provide password for single command
DEVICE_PASSWORD=0000 crossbuilder

# provide password for shell session
export DEVICE_PASSWORD=0000
crossbuilder
```

The default path, where crossbuilder expects the password file is under `~/.config/crossbuilder/device_password`. You can create a file named `device_password` there with the content `DEVICE_PASSWORD=0000`. Or specify a folder of your choice like this:

```bash
DEVICE_PASSWORD_FILE=path_to_file crossbuilder
```

## lxd-image

We can specify specific lxd images for the build process. This can be useful e.g. when building for specific target releases like 24.04. 
```bash
crossbuilder --lxd-image=ubuntu:24.04
```

Change a line of code and type crossbuilder again to re-build and re-deploy.

## no deb build

To go even faster, bypass building Debian packages with:
```bash
crossbuilder --no-deb
```

## setup lxd

For an even faster LXD setup, resetup LXD using ZFS:
```bash
crossbuilder setup-lxd
```

## lxd shell

To enter the LXD container used to build:
```bash
crossbuilder shell
```

## ssh

To use ssh instead of adb to deploy, use the ```--ssh``` option. If your device has ssh enabled on address let's say 192.168.0.5, use:
```bash
crossbuilder --ssh=phablet@192.168.0.5
```

# Troubleshooting

If crossbuilder fails, check the following:

## branch
Do you use the correct branch? `main` is generally targeting the latest OS version, you might be trying to build for an older version.

## install on device fails
Did you provide the devices lock screen password?

## changes do not appear on my device
In order to see the changes a restart of Lomiri might be needed.

## docker
If docker is running on your system, that may configure iptables forward policy to drop forwards of other devices.
You can check with:
`sudo iptables -S FORWARD | head -5`
If the output is
```
-P FORWARD DROP
-A FORWARD -j DOCKER-USER
-A FORWARD -j DOCKER-FORWARD
```
then you need to fix that. For solutions please see https://canonical.com/lxd/docs/default/howto/network_bridge_firewalld/#prevent-connectivity-issues-with-lxd-and-docker.

## PACKAGENAME: command not found
The lxc image might be missing the named package.
Check for the package in the container with
`lxc exec CONTAINERNAME -- dpkg -l PACKAGENAME`
If its missing, you can install it with:
```
lxc exec CONTAINERNAME -- apt update
lxc exec CONTAINERNAME -- apt install -y PACKAGENAME
```
## get logs
Crossbuilder doesn't have a verbose parameter, so output isn't always sufficient.
Some logs can be created like this:
`bash -x $(which crossbuilder) --lxd-image=ubuntu:24.04 --architecture=arm64 --password=???? 2>&1 | tee crossbuilder.log`
(adjust parameters according to your command)

## arch missing for dpgk
If the build brings this error:
```
The package has been created.
dpkg: error processing archive CONTAINERNAME-cross-build-deps_1.0_arm64.deb (--unpack):
 package architecture (arm64) does not match system (amd64)
Errors were encountered while processing:
 CONTAINERNAME-cross-build-deps_1.0_arm64.deb
mk-build-deps: dpkg --unpack failed
```
likely dpgk is not yet configured to build for cross architecture. You can fix that like this:
```
lxc exec CONTAINERNAME -- dpkg --add-architecture arm64
lxc exec CONTAINERNAME -- apt-get update
```

## new keyboard doesn't get installed
Crossbuilder by default runs `apt-get dist-upgrade` for the build packages. This means only existing packages will be updated with the changes you may have applied.
When adding for instance an entirely new keyboard, its .deb needs to be installed manually once. Then on recurrent builds it will be updated as well.