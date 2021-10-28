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

## Packaging these applications

For the example applications in this article there's an [example recipe](./snap/snapcraft.yaml). Before we look at that in detail we'll simply use it to package these applications.

    sudo snap install snapcraft --classic
    git clone https://github.com/AlanGriffiths/iot-gui-snaps.git
    cd iot-gui-snaps
    snapcraft

When you first run `snapcraft` you will be asked "Support for 'multipass' needs to be set up. Would you like to do it now? [y/N]:", answer "yes".

_[Note: some development environments (e.g. containers) do not support multipass, it is possible to use snapcraft with LXD (`snapcraft --use-lxd`), remote building on Launchpad (`snapcraft remote-build`) or running natively `snapcraft --destructive-mode`. The latter will potentially change and, as the name suggests, break the environment it runs in.]_

After a few minutes the snap will be built with a message like:

    Snapped fork-and-rename-me_0+git.5fcc9fb_amd64.snap

You can then install and run the snap:

    sudo snap install --dangerous *.snap
    snap run fork-and-rename-me.neverputt

But you are likely to see a warning:

    WARNING: wayland interface not connected! Please run: /snap/fork-and-rename-me/current/bin/setup.sh
    WARNING: hardware-observe interface not connected! Please run: /snap/fork-and-rename-me/current/bin/setup.sh
    WARNING: joystick interface not connected! Please run: /snap/fork-and-rename-me/current/bin/setup.sh
    Failure to initialize SDL (No available video device)

Run the setup script to connect the missing interfaces, and try again:

    /snap/fork-and-rename-me/current/bin/setup.sh
    snap run fork-and-rename-me.neverputt

You can also run the other applications in the snap:

    snap run fork-and-rename-me.mastermind
    snap run fork-and-rename-me.bomber

This confirms the application runs successfully with Ubuntu Frame in the development environment before deploying onto a device. While developing a snap this is the best way to ensure that the snap contains everything necessary and is configured correctly.

## Building for and installing on a device

A lot of devices are not using the "amd64" archictecture typical of development machines. The simplest way to build your snap for other architectures is:  

    snapcraft remote-build

This uses the launchpad build farm to build each of the architectures supported by the snap. (This can take some time if the farm is busy, requires you to have a Launchpad account and to be happy uploading your snap source to a public location.)

Once the build is complete you can `scp` the `.snap` file to your device and install using `--dangerous`.

For the sake of these notes I'm using a VM set up using the approach described in [Ubuntu Core: Preparing a virtual machine with graphics support](https://ubuntu.com/tutorials/ubuntu-core-preparing-a-virtual-machine-with-graphics-support). Apart from the address used for scp and ssh this is the same as any other "device". 

    scp -P 10022 *.snap <username>@<hostname>:~
    ssh -p 10022  <username>@<hostname>
    snap install ubuntu-frame
    snap install --dangerous *.snap

You'll see the Ubuntu Frame greyscale background, but you won't see any of the games. If you check the logs you'll see why:

    $ snap logs -n 30 fork-and-rename-me
    2021-10-28T14:39:20Z fork-and-rename-me.mastermind[6712]: WARNING: hardware-observe interface not connected! Please run: /snap/fork-and-rename-me/current/bin/setup.sh
    2021-10-28T14:39:20Z fork-and-rename-me.neverputt[6714]: WARNING: hardware-observe interface not connected! Please run: /snap/fork-and-rename-me/current/bin/setup.sh
    2021-10-28T14:39:20Z fork-and-rename-me.bomber[6732]: WARNING: audio-playback interface not connected! Please run: /snap/fork-and-rename-me/current/bin/setup.sh
    2021-10-28T14:39:20Z fork-and-rename-me.neverputt[6714]: WARNING: audio-playback interface not connected! Please run: /snap/fork-and-rename-me/current/bin/setup.sh
    2021-10-28T14:39:20Z fork-and-rename-me.bomber[6732]: WARNING: joystick interface not connected! Please run: /snap/fork-and-rename-me/current/bin/setup.sh
    2021-10-28T14:39:20Z fork-and-rename-me.mastermind[6712]: WARNING: audio-playback interface not connected! Please run: /snap/fork-and-rename-me/current/bin/setup.sh
    2021-10-28T14:39:21Z fork-and-rename-me.mastermind[6712]: WARNING: joystick interface not connected! Please run: /snap/fork-and-rename-me/current/bin/setup.sh
    2021-10-28T14:39:21Z fork-and-rename-me.neverputt[6714]: WARNING: joystick interface not connected! Please run: /snap/fork-and-rename-me/current/bin/setup.sh
    2021-10-28T14:39:21Z fork-and-rename-me.neverputt[6714]: ALSA lib conf.c:4120:(snd_config_update_r) Cannot access file /usr/share/alsa/alsa.conf
    2021-10-28T14:39:21Z fork-and-rename-me.neverputt[6714]: Failure to initialize SDL (Could not initialize UDEV)
    ...

All these WARNING message give the clue:

    /snap/fork-and-rename-me/current/bin/setup.sh

After which you should see the "Bomber" start. You can configure the active game using the `daemon` snap configuration:

    snap set fork-and-rename-me daemon=neverputt
    snap set fork-and-rename-me daemon=mastermind

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

Because there is only one `layout` stanza this contain a mixture of generally useful paths and those specific to individual applications and toolkits. If you're packaging a single application, can be simplified.

```yaml
layout:
  /usr/share/libdrm:  # Needed by mesa-core20 on AMD GPUs
    bind: $SNAP/graphics/libdrm
  /usr/share/drirc.d:  # Used by mesa-core20 for app specific workarounds
    bind: $SNAP/graphics/drirc.d
  # Generally useful
  /usr/share/fonts:
    bind: $SNAP/usr/share/fonts
  /usr/share/icons:
    bind: $SNAP/usr/share/icons
  /usr/share/sounds:
    bind: $SNAP/usr/share/sounds
  /etc/fonts:
    bind: $SNAP/etc/fonts
  # neverputt
  /usr/share/games/neverball:
    bind: $SNAP/usr/share/games/neverball
  # gnome-mastermind & GTK
  /usr/share/gnome-mastermind:
    bind: $SNAP/usr/share/gnome-mastermind
  /usr/lib/$SNAPCRAFT_ARCH_TRIPLET/gdk-pixbuf-2.0:
    bind: $SNAP/usr/lib/$SNAPCRAFT_ARCH_TRIPLET/gdk-pixbuf-2.0
  /usr/share/mime:
    bind: $SNAP/usr/share/mime
  /etc/gtk-3.0:
    bind: $SNAP/etc/gtk-3.0
  # bomber
  /usr/share/bomber:
    bind: $SNAP/usr/share/bomber
```

### The applications

There are two parts of the [snapcraft.yaml](./snap/snapcraft.yaml) that relate to each application and possibly some additional files.

The `apps` stanzas are all pretty similar:

```yaml
apps:
  neverputt:
    command-chain:
      - bin/wayland-launch
    command: usr/games/neverputt
    daemon: simple
    restart-condition: always
    plugs:
      - opengl
      - wayland
      - hardware-observe
      - audio-playback
      - joystick
```

The `command-chain` lists some setup scripts, the `command` is the program to run and the `plugs` are the Snapd interfaces the application needs access to to work.

The `daemon` and `restart-condition` are informing snapd that the app runs as a daemon (which it needs to do for IoT).

#### The Neverputt part 

```yaml
parts:
  neverputt:
    plugin: dump
    source: neverputt
    stage-packages:
      - neverputt
      - libsdl2-2.0-0
      - libsdl2-image-2.0-0
      - libsdl2-mixer-2.0-0
      - libsdl2-net-2.0-0
```

Applications are installed from repositories (such as the Ubuntu Archive) are controlled using `stage-packages:` which lists the neverputt package and SDL2 toolkit libraries it needs.

Using the `plugin: dump` means the `source: neverputt` directory is copied into the snap. For Neverputt this contains a configuration file that sets the application to fullscreen.

#### The Mastermind part 

```yaml
  mastermind:
    plugin: nil
    build-packages:
      - libgdk-pixbuf2.0-0
      - shared-mime-info
    override-build: |
      # Update mime database
      update-mime-database ${SNAPCRAFT_PART_INSTALL}/usr/share/mime
    stage-packages:
      - gnome-mastermind
      - librsvg2-common
      - gsettings-desktop-schemas
      - libglib2.0-bin
    override-prime: |
      snapcraftctl prime
      # Compile the gsettings schemas
      /usr/lib/${SNAPCRAFT_ARCH_TRIPLET}/glib-2.0/glib-compile-schemas "$SNAPCRAFT_PRIME/usr/share/glib-2.0/schemas"
      # Index the pixbuf loaders
      LOADERS_PATH=$(echo ${SNAPCRAFT_PRIME}/usr/lib/${SNAPCRAFT_ARCH_TRIPLET}/gdk-pixbuf-2.0/*/loaders)
      QUERY_LOADERS=/usr/lib/${SNAPCRAFT_ARCH_TRIPLET}/gdk-pixbuf-2.0/gdk-pixbuf-query-loaders
      GDK_PIXBUF_MODULEDIR=${LOADERS_PATH} ${QUERY_LOADERS} > ${LOADERS_PATH}/../loaders.cache
      sed s!$SNAPCRAFT_PRIME!!g --in-place ${LOADERS_PATH}/../loaders.cache
```

Applications are installed from repositories (such as the Ubuntu Archive) are controlled using `stage-packages:` which lists the gnome-mastermind package and the SVG, Gsettings and GLib libraries it needs.

The `build-packages`, `override-build` and `override-prime` stanzas are there to incorporate the mime database and pixbuf loaders into the snap. This setup of mime database and pixbuf loaders is done differently for IoT as the snap cannot rely on getting these from the host environment.

#### The Bomber Part

```yaml
  bomber:
    plugin: dump
    source: bomber
    stage-packages:
      - bomber
      - qtwayland5
      - dbus
    override-prime: |
      snapcraftctl prime
      # replace the SNAP_NAME placeholder with our actual project name
      sed -i "s/SNAP_NAME/$SNAPCRAFT_PROJECT_NAME/" $SNAPCRAFT_PRIME/etc/dbus-1/session.conf
```

Because Bomber won't run without a session dbus, and this isn't available to daemons on core (there is no "session") we need to include a dbus session in the snap. The `source: bomber` contains a script and configuration file for running `dbus-run-session`.  