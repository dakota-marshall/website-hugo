---
title: "Configure Apple Magic Trackpad 2 Gestures on Manjaro KDE"
date: 2020-06-10T12:12:00-04:00
draft: false
toc: false
images:
tags:
  - linux
  - kde
  - manjaro
  - Apple
---


### Intro  
  
I haven't seen any full-length tutorials for getting the Apple Magic Trackpad 2, or really any multi-touch track-pad, working in Manjaro, or more specifically KDE, yet so I figured I'd write down how I did it. Most of the info I got was from the libinput-gestures [Github page](https://github.com/bulletmark/libinput-gestures) if you would like to check it out. The device works out of the box with left, right, and middle-click as well as 2 finger scrolling. But to get the rest of the features you need to set a few things up. YMMV on whether or not everything in this works for you.  
  
---  
  
### Getting Started  
  
The very first thing you should know is that for fully supported drivers you need to be on at least Linux version *4.20*, I am running on *5.4.43-1* which is Manjaro's LTS kernel release.  
  
Once you are on a kernel version that supports the Trackpad drivers, plug in the device and it should immediately start working for the most part. However, the multi-finger gestures do not work, as nothing is currently bound to them. For that, we need *libinput-gestures*. I've seen a couple forum posts to suggest using other input libraries but the only one that works in this scenario I've seen is libinput.  
  
```sudo pacman -S libinput-gestures```  
  
Go ahead and install *libinput-gestures*. The first thing we should do is add our user to the *input* group.  
  
```sudo gpasswd -a $USER input```  
  
Form there we can run the following commands:  
  
```
libinput-gestures-setup autostart  
libinput-gestures-setup start  
```  
Alright awesome, now that its up and running we can go ahead and shut it back down and start building our config.  
  
```libinput-gestures-setup stop```  
  
Your config file exists in 2 places: `/etc/libinput-gestures.conf` and `~/.config/libinput-gestures.conf`. The former is for default gestures, all users on the system will share these gestures. The one in *.config* is personal to your user. I will be using the one in *.config* but it shouldn't matter.  
  
```nano ~/.config/libinput-gestures.conf```  
  
Here is a paste of my config file:  
  
```  
# Swipe threshold (0-100)  
swipe_threshold 0  
  
# Gestures  
gesture swipe up 3 konsole  
gesture swipe down 3 xdotool key Super_L+Page_Down  
gesture pinch out 3 xdotool key Ctrl+F9  
gesture swipe right 3 xdotool key Ctrl+Shift+F2  
gesture swipe left 3 xdotool key Ctrl+Shift+F1  
```  
  
The format for the gestures is pretty straight forward: `gesture` means just that, create a gesture. Then you define the gesture type. These can be `swipe` and `pinch`. Then specify the direction with `up` `down  
` `left` and `right`. Next you define the number of fingers to perform the gesture with. I have not testedwith any more than 3 fingers for gestures, I would imagine 4 works though I dont know about 5 and above.  
Finally you give it the command to run. For this I would suggest *xdotool* as you can pretty much do anything that the mouse and keyboard can do with that command. Mine just do keybinds though you can do some mo  
re complex macro'ing if youre into that sort of thing.  
  
Once you save your config file run:  
  
```libinput-gestures-setup start```  
  
Doing the defined gestures should run the command you specified. Should you notice that its not a quick way to ensure that libinput is running the command is to stop *libinput-gestures* again and then run:  
  
```  
libinput-gestures -d  
(Ctrl+C to stop)  
```  
This will show you the commands that would be ran once a recognizable gesture is detected. If all is good go ahead and start *libinput-gestures* again.  
  
### Is that it...?  
  
Well yes, but actually no... At this point the track-pad will work and you can bind gestures to your hearts content. However, you'll soon notice that if you connect the track-pad through Bluetooth, and have to reconnect the track-pad, that gestures stop registering. The gestures will also stop working sometimes once waking from a lock or sleep.  
  
The only way to fix this is by restarting the gesture service. You can do this by hand, but using *dbus-action* is a more elegant solution. Here is how to set that up.  
  
---  
### Configuring D-Bus Action

First things first is to install *dbus-action*. This is something that needs to be installed from the AUR, so do that however you would install any other AUR package. For me I use pamac:

```sudo pamac build dbus-action```

Once built we can start configuring the service. At this point we need to use the following command to find the message that is displayed on the D-Bus whenever we connect our track-pad. Run:

```dbus-action -m all```

From there you an try connecting and disconnecting your track-pad a couple times to see if you are able to find the message. For me it was `workingTouchpadFoundChanged` on my desktop but my laptop actually showed `mousePluggedInChanged` so you will need to see which one appears for you: 

```session org.kde.touchpad workingTouchpadFoundChanged: 1```

Now that we have that, we need to create the config file that will tell *dbus-action* to restart the gesture service once it detects a touchpad being plugged in. Create a file `~/.config/dus-action.conf` and paste the following:

```
triggers:   
 - bus: session  
  interface: org.kde.touchpad  
  member: workingTouchpadFoundChanged  
  values:  
   1: "libinput-gestures-setup restart"
```

Save that and then we need to run the same steps for this as we did for *libinput-gestures-setup*:

```
dbus-action-setup start
dbus-action-setup autostart
```

Once that is running it will begin restarting the service once it detects a track-pad has been plugged in. You can obviously test by re-plugging in your track-pad and ensure the gesture service restarts as expected. This should restart the gesture service when both plugged in via the lightning cable or when connected via Bluetooth.

At this point your track-pad should be completely good to go! You can bind the track-pad to issue whatever commands you are looking for. I have had really good luck with the responsiveness of the gestures and scrolling on the touch-pad. This allowed me to bind several workflow shortcuts I normally use to the track-pad instead of relying on the shortcut only.

If you feel like I missed anything or the setup didnt work for you, you can get in contact with me via my [Keybase](https://keybase.io/dmarshall549), email, or any of my attached accounts.