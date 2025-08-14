---
title: Using camera modules on a Raspberry Pi running Ubuntu 25.04 (Plucky Puffin)
date: 2025-04-23
---

With the release of Ubuntu 25.04 Plucky Puffin, the camera stack now works on a Raspberry Pi running Ubuntu. The main changes were adding the new PiSP driver to `libcamera` and adding the `rpicam-apps` and `picamera2` userland packages (along with their dependencies).

We also tried to support IMX500's (Raspberry Pi AI camera) AI features (except the Hailo preprocessing models), but that comes with a catch described later.

We have made the usage of the camera stack as close to Raspberry Pi OS as possible, so you can use your existing setup and scripts and run them on Ubuntu as is.  

![This is an image I took after making changes to `libcamera` using `qcam`](https://i.imgur.com/VgVzS3W.png)

This is an image I took after making changes to `libcamera` using `qcam`

![This is an image I took after making changes to `rpicam-apps` using `rpicam-still`](https://i.imgur.com/8hVya4y.png)

This is an image I took after making changes to `rpicam-apps` using `rpicam-still`

![This is a minimal example of object detection using the Raspberry Pi AI camera and `rpicam-hello`](https://i.imgur.com/zuYRy4o.png)

This is a demo of object detection using the Raspberry Pi AI camera and `rpicam-hello`

Detailed instructions on using the camera stack are given in the [Ubuntu Boards Documentation](https://canonical-ubuntu-boards.readthedocs-hosted.com/en/latest/how-to/rpi-camera/). If you find any bugs, please report them on Launchpad under [libcamera](https://bugs.launchpad.net/ubuntu/+source/libcamera) or [rpicam-apps](https://bugs.launchpad.net/ubuntu/+source/rpicam-apps).

### The AI camera stuff
While I could get the things described in the [AI camera guide](https://www.raspberrypi.com/documentation/accessories/ai-camera.html) to work, there is a critical catch - the `imx500-firmware` required for enabling these features does not have a redistributable license. We are communicating with Raspberry Pi to resolve this, but we cannot distribute `imx500-firmware` in Ubuntu due to this issue.

However, as this is a deb package, you can manually install the `.deb` package and then proceed to install the packages from [this PPA](https://launchpad.net/~r41k0u/+archive/ubuntu/imx500-picam), and it should work. The steps are mentioned at the end of the [Ubuntu Boards Documentation](https://canonical-ubuntu-boards.readthedocs-hosted.com/en/latest/how-to/rpi-camera/) 

But again, this is just a workaround, which we don't want our end users to do. We hope that this gets resolved and we can provide the packages (with their AI camera features) in the Ubuntu archive for the next release (This helps us with achieving parity with Raspberry Pi's documentation, and the end users can use most of their stuff "as-is" between Raspberry Pi OS and Ubuntu).
