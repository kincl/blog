---
title: "Reviving a Legacy HDHomeRun Device"
date: 2023-09-01
draft: true
---

HDHomeRun is a network-attached TV tuner built by a company called SiliconDust. A friend recently gave me an old HDHR-US, one of the first models the company made. I was using it in conjunction with my Plex server to get over-the-air TV channels but Plex suddenly stopped supporting the older models of HDHomerun so I set out to find a way to do it and not let these great devices become e-waste.

<!--more-->

{{< figure
src="hdhomerun-hdhr-us.jpg"
caption="HDHR-US model"
class="float-image-right"
>}}

## Initial Discovery

The basic idea is that it is a digital TV tuner that you can connect to over your local network. SiliconDust offers a client application for MacOS that can *discover the tuner over the network* which I found very interesting and got me wondering how this device works. It has a simple web interface with a link to download the firmware upgrade utility for MacOS.

{{< figure
src="hdhr-web-interface.png"
caption="web interface"
class="mid-image"
>}}

## Debugging the Installer

The installer is primarily used to install the HDHomerun.app to Applications/ but it also puts an executable into place and then executes a post-install shell script that updates the firmware on the device.

{{< highlight bash >}}
$ pkgutil --payload-files /Volumes/HDHomeRun\ Installer\ 20231214/HDHomeRun\ Installer.pkg
.
./usr
./usr/local
./usr/local/bin
./usr/local/bin/hdhomerun_config
{{< /highlight >}}

### The firmware update script

I was curious about what this firmware update script was doing. I couldn't find a way to dump something akin to post-install scripts like with RPMs so I instead opted for the foolproof way of: "run ps while the script is running to get the path and cat it before the installer cleans up its own tmp files"

You can see the full script [here](https://github.com/kincl/hdhr-legacy-proxy/blob/main/doc/firmware-upgrade-script.sh) which gave me an idea of how to use the `hdhomerun_config` utility

## How does it work


