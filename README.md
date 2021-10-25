# Working with Ubuntu Frame - the developer experience

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

## Snap Packaging for the Internet of Things

Much of what you find online about packaging GUI applications as a snap refers to packaging for "desktop". Some of that doesn't apply to the "Internet of Things" as Ubuntu Core and Ubuntu Server do not include everything a desktop installation does and the snaps need to run as "daemons" instead of being launched in a user session.

