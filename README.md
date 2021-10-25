# IOT GUI snaps

## Introducing Ubuntu Frame

Ubuntu Frame provides a way to embed your application into a kiosk style, embedded or digital signage solution.

To do this there are several steps to go through and multiple ways to achieve each step. The end goal of this an application snap and the infrastructure to deploy and configure it.

The process of developing an application isn't specific to Ubuntu Frame and nor is the building of custom Ubuntu Core images with "gadget snaps" to configure a system with specific snaps and provide configuration to them. There's plenty of good documentation for this elsewhere.

Here we will concentrate on the steps in between: taking an application, ensuring it works with Ubuntu Frame, and packaging it as a snap.

## Setting up a development environment

This is simplest if you have a Wayland based application that you can test on your desktop. In this case you simply need to install and run Ubuntu Frame on your desktop and run your application with it.

The key thing to know about connecting a Wayland app to Ubuntu Frame is that the connection is controlled by the `WAYLAND_DISPLAY` environment variable. We need to set that to the same thing for both processes and, because you may be using a Wayland based desktop environment (which uses the default of `wayland-0`) we'll set it to `wayland-99`

So open a terminal window and type:

    sudo snap install ubuntu-frame
    export WAYLAND_DISPLAY=wayland-99
    ubuntu-frame&

You should see a "Mir on X" window containing a graduated grey background.

_[Note: it is possible to set up a development environment in a container, VM or even on another computer. To do this you need to use ssh X-forwarding and set the `XAUTHORITY` environment variable. For this you start your terminal session with (before the above commands):_

    ssh -CX <hostname>
    export XAUTHORITY=~/.Xauthority

_You can then continue as above.]_

## Checking applications work with Ubuntu Frame

Now instead of whatever application you really want to use I'm going to use some examples. Now, still in the same terminal window:

### GTK: Mastermind

    sudo apt install gnome-mastermind
    gnome-mastermind

Now Frame's "Mir on X" should contain the "Mastermind" game.

Close that (Ctrl-C) and try the next example...

### Qt: Bomber

    sudo apt install bomber
    QT_QPA_PLATFORM=wayland bomber

Now Frame's "Mir on X" should contain the "Bomber" game. Note that Qt does not default to using Wayland and that `QT_QPA_PLATFORM` needs to be set to get that behaviour.

Close that (Ctrl-C) and try the next example...

### SDL2: Neverputt

    sudo apt install neverputt
    SDL_VIDEODRIVER=wayland neverputt

Now Frame's "Mir on X" should contain the "Bomber" game. Note that SDL2 does not default to using Wayland and that `SDL_VIDEODRIVER` needs to be set to get that behaviour.

Close that (Ctrl-C) and try the next example...

Actually, that's enough examples. You can see how to prove that an application is able to work with Ubuntu Frame. The next step is to use snap packaging to prepare the application for use on an "Internet of Things" device.

## Snap Packaging GUIs for the Internet of Things

There's a lot of information about packaging snaps online, and the purpose here is not to teach about the snapcraft packaging tool or the Snap store. There are good resources for that elsewhere online. We instead focus on the things that are special to IoT graphics.

Much of what you find online about packaging GUI applications as a snap refers to packaging for "desktop". Some of that doesn't apply to the "Internet of Things" as Ubuntu Core and Ubuntu Server do not include everything a desktop installation does and the snaps need to run as "daemons" instead of being launched in a user session. In particular, there are various Snapcraft "extensions" that help writing snap recipes that integrate with the Desktop Environment (e.g. using the correct theme).

Writing Snap recipes without these extensions is the main thing you need to understand. You need to do some setup for your particular program and the GUI toolkit it uses.

For the example applications in this article there's an [example recipe](./snap/snapcraft.yaml) that we will look through.

Before we explore the individual applications we'll first mention a few things that are universal.

### The Header

This you should adapt for your own snap.

```yaml
name: fork-and-rename-me # you probably want to 'snapcraft register <name>'
version: git # just for humans, typically '1.2+git' or '1.3.2'
summary: Single-line elevator pitch for your amazing snap # 79 char long summary
description: |
  This is my-snap's description. You have a paragraph or two to tell the
  most important story about your snap. Keep it under 100 words though,
  we live in tweetspace and your description wants to look good in the snap
  store.
confinement: strict
compression: lzo
grade: stable
base: core20
```

### The graphics-core20 plug and environment

There are three snippets of the recipe that relate to providing the userspace graphics needed by your application. What they do is use a "content interface" to include (by default) the Mesa open drivers. You can treat this as "magic" so long as you don't need to make changes. On the Mir discourse forum there's a lot more detail on [the graphics-core20 Snap interface](https://discourse.ubuntu.com/t/the-graphics-core20-snap-interface/23000) and it's use.

```yaml
plugs:
  graphics-core20:
    interface: content
    target: $SNAP/graphics
    default-provider: mesa-core20

environment:
  LD_LIBRARY_PATH:    $SNAP/graphics/lib:${SNAP}/usr/lib/${SNAPCRAFT_ARCH_TRIPLET}/pulseaudio
  LIBGL_DRIVERS_PATH: $SNAP/graphics/dri
  LIBVA_DRIVERS_PATH: $SNAP/graphics/dri
  __EGL_VENDOR_LIBRARY_DIRS: $SNAP/graphics/glvnd/egl_vendor.d
```

```yaml
layout:
  /usr/share/libdrm:  # Needed by mesa-core20 on AMD GPUs
    bind: $SNAP/graphics/libdrm
  /usr/share/drirc.d:  # Used by mesa-core20 for app specific workarounds
    bind: $SNAP/graphics/drirc.d
```

```yaml
  cleanup:
    after:
      - neverputt
      - mastermind
      - bomber
      - mir-kiosk-snap-launch
    plugin: nil
    build-snaps: [ mesa-core20 ]
    override-prime: |
      set -eux
      cd /snap/mesa-core20/current/egl/lib
      find . -type f,l -exec rm -f $SNAPCRAFT_PRIME/usr/lib/${SNAPCRAFT_ARCH_TRIPLET}/{} \;
      rm -fr "$SNAPCRAFT_PRIME/usr/lib/${SNAPCRAFT_ARCH_TRIPLET}/dri"
      for CRUFT in bug drirc.d glvnd libdrm lintian man; do
        rm -rf "$SNAPCRAFT_PRIME/usr/share/$CRUFT"
      done
```

### The layout

The `layout` ensures that files can be found by applications where they are expected by the toolkit or application.

This is a rather messy section in the example as it contains the requirements of several applications and toolkits. If you're packaging a single application, can be simplified.  