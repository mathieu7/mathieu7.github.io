---
layout: post
title:  "Automated testing and development in Android Part II"
date:   2014-09-18 23:18:51
categories: android
disqus: true
---

In my past post a number of months ago, I went over the basics of automating your
android testing environment. In this post, I want to expand on that even further.

The scripts I had outlined previously only made use of usb connected devices.
So in essence, every device you want to test on had to be plugged into your computer
before you could run the tests and gather results.

Unfortunately that isn't very scalable. If you work in a large office with a
multitude of android devices, testing in this way is pretty difficult.

However, ADB has an option to connect to your devices over your WiFi network.
If we connect our devices to adb over wifi, we can unplug them and run our tests
without having to connect every device to our computer.

#### What we'll need
<ul>
	<li>Wifi network</li>
	<li>Android devices</li>
	<li>All devices connected to this wifi network</li>
</ul>

Unfortunately, unless your device is rooted, we'll have to connect our
device through USB then initiate our ADB tcp connection. Then we can remove
from USB and still be connected to our main computer.

I've added a script that will handle and store all device connections.

{% highlight bash %}
#!/bin/bash
#************************************************#
#              setup_device_wifi.sh              #
#           written by Matt Miller               #
#                Sept 16th, 2014                 #
#                                                #
#           Fetch connected device ips           #  
#	         and connect over ADB wifi           #
#************************************************#
usage() {
	echo "Usage: $0"
	echo "Simple script to fetch ips from any connected devices in your LAN
		and update your config file with a list of possible device ips"
	echo "-h : Display usage"
}

# Function to connect device to adb over wifi
function  getIP {
	device=$1
	index=$2

	ip=$(adb -s $device shell netcfg | grep wlan0 | awk '{ gsub("(//*.*)","",$3); print $3}')

	if [ -z  "$ip" ]; then
		echo "No netcfg output"
   	else
   		echo "$device ip = $ip"
   		cmd="adb -s $device connect $ip:$port"
   		echo $cmd
   		porti=$(($port + $index))
   		adb -s $device tcpip $porti
   		res=$(adb -s $device connect "$ip:$porti")
		echo $res
		if ! [[ $res =~ *'unable to connect'* ]]
		then
			#write the ip:port combination to the file.
			echo "$ip:$porti/n" >> $device_ip_file
		fi
   	fi
}

function checkDependencies {
	# Dependency ADB (Android Debug Bridge)
	if [ -z $(which adb) ] ; then
		echo "adb binary not in your user's path. It is a dependency of this script."
		exit
	fi
}

# Iterate over a list of saved ips. And also connect to them. If it's already connected, nothing happens.
# This mitigates a tiny issue with some phones disconnecting from wifi connection after unplugging the usb.
function checkIPList {
 	while read ip; do
		adb connect $ip
	done <$device_ip_file
}

# source the config file
source test_config

# parse cmd line args
while getopts :p:aet: opt; do
	case $opt in
		h)
			usage
			exit
			;;
	esac
done

checkDependencies

# Get all devices currently running on adb and store their ids into an array
output=$(adb devices)
devices=( $(printf "%s","$output" | awk '{if (NR!=1){print $1}}') )

# iterate over each device, running tests we've specified.
index=0

checkIPList

for device in "${devices[@]}"
do
    rm  $device_ip_file
	check="\:"
	if [[ $device =~ $check ]];
	then
		echo "Skipping wifi device $device"
		echo "$device/n" >> $device_ip_file
	else
		echo "Fetching ip for device $device"
		getIP $device $index
		index=$(( $index + 1 ))
	fi
done
echo "Finished."
{% endhighlight %}

#### The Gist

The script calls adb to fetch your connected devices. If your devices aren't connected
over wifi, it will fetch your device's network ip and connect ADB to that ip.

Once connected, it will save all connections into a text file.
This script requires a config file called test_config.

Inside are some basic environment vars:

{% highlight bash %}
#!/usr/bin/env bash
apk_location=
test_apk_location=
timeout=
port=5555
device_ip_file=device_ips.txt
{% endhighlight %}

The ones of importance for this script are device_ip_file and port.
Port represents the default port number you will connect to your device's ip
over (e.g. 127.0.0.1:5555). The port number increases per connection.

<i>Imagine your network of connected devices:</i>
<ol>
   <li> 127.0.0.1:5555 -> will connect to the ADB on port 5555</li>
   <li> 127.0.0.2:5556 -> will connect to ADB on port 5556</li>
</ol>
... and so on for each device.

$device_ip_file is the location of the file you will write connected devices' ip:port
combination to.

So to setup for wifi ADB connections, plug in your device(s) and run
{% highlight bash %}
$ ./setup_wifi_device.sh
{% endhighlight %}

You can do this as many times as you want and the devices will remain connected,
until you explicitly disconnect them, lose network connection or restart the ADB
instance.

From there, just run your test_automator.sh script from the previous post, and
you're all set!
