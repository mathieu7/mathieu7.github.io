---
layout: post
title:  "Playing with the Myo Armband"
date:   2014-10-16 12:18:51
categories: myo armband lua
disqus: true
---

Hi guys, I wanted to share a pretty cool gadget with you. It's called the Myo Armband, developed by Thalmic Labs.

I preordered the armband back in March and it's finally come in the mail! The past couple days I've been playing
with it and am really looking forward to working with it for some personal projects and just doing some cool things.

#### First thing: WTF is the Myo?

The Myo is an armband complete with electromagnetic sensors that determine various gestures you
do with your arm(s). It also has an accelerometer integrated to detect your arm's relative positioning.
All of this information is communicated via Bluetooth to your device of choice.

Here is a preview video:

<iframe width="600" height="400" src="https://www.youtube.com/embed/oWu9TFJjHaM" frameborder="0" allowfullscreen></iframe>

The Myo is in a developer beta, and is very much a work in progress in terms of support
and how well the armband works (how accurate it is, etc.). However, the developers at
Thalmic Labs are improving the firmware constantly, so it's just a matter of time
before it's the real deal.

Despite a few hiccups, the Myo comes equipped with an SDK to integrate this device into
any software/platform you can imagine! I'm having a hard time seeing many people building
super meaningful experiences with it unless the device really catches on. But for personal
uses, it couldn't be easier.

#### Sample Myo Usage: Switching Chrome Tabs with gestures

The Myo comes with a piece of software called the Myo Connect. Put simply, it's
the software that integrates with the bluetooth USB adapter you plug into your
computer, and processes inputs from the Myo itself. It is the middleman between
your Myo and your OS.

The Myo SDK comes with a Lua scripting framework for creating "Myo Scripts".
These scripts turn into processes when imported into the Myo Connect's Script Manager.

Here's a small demo video of it in action, interfacing with Google Chrome.

<video width="600" height="400" controls src="/assets/myovideo.mp4">
  Your browser does not support HTML5 video.
</video>

#### Behind the scenes

Getting a simple demo of your own Myo functionality is pretty straightforward.
Here is the Lua script you can import into Myo Connect and it just <em>works</em>
when you switch to Chrome.

{% highlight lua %}
scriptId = 'com.thalmic.examples.outputeverything'

-- A callback to whenever the Myo detects a gesture.
-- We do an action depending on the pose.
function onPoseEdge(pose, edge)
    myo.debug("onPoseEdge: " .. pose .. ", " .. edge)
    if pose == "waveOut" and edge == "on" then
    	myo.vibrate("short")
    	myo.keyboard("tab", "press", "control")
    	myo.debug("keyboard waveOut")
    end

    if pose == "waveIn" and edge == "on" then
    	myo.vibrate("short")
    	myo.keyboard("tab", "press", "control","shift")
   	end
end

function onPeriodic()
end

-- Figure out if Chrome is running and is the active application.
-- When that is the case, our Myo's extra functionality is turned "on".
function onForegroundWindowChange(app, title)
	  local wantActive = false
	  activeApp = ""
    myo.debug("onForegroundWindowChange: " .. app .. ", " .. title)
    wantActive = string.find(title, "Chrome") ~= nil
    if wantActive == true then
    	  activeApp = "Chrome"
        myo.debug("Active app: " .. activeApp)
    end
    return wantActive
end

function activeAppName()
    return activeApp
end

-- Detect a change in the active application.
function onActiveChange(isActive)
    myo.debug("onActiveChange")
end
{% endhighlight %}

For Lua documentation and tutorials, check [here](http://www.lua.org/pil/contents.html).

The script maps two gestures to two Chrome keyboard shortcuts
and executes that shortcut when the Myo detects that motion.
So if you wave your hand to the right, chrome switches to the tab on the right.
If you wave to the left, the previous tab becomes active.

This is really simple, and took only a few minutes to write!

#### Potential Improvements

One thing I noticed was the lack of accuracy in gesture detection.
This could be improved on, and I have some ideas! Hopefully it'll come to fruition in
the next installment.
