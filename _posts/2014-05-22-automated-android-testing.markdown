---
layout: post
title:  "Automated testing and development in Android"
date:   2014-05-22 22:13:00
categories: android
disqus: true
---

Working with the Android ecosystem, you want to have tools to make your development
life easier.

I've decided to write weekly development blogs on some topic that I either
tackled that week at work or something I've been working on. I don't write much since
graduating college, and that's a big reason for doing this blog. This will
also be a good opportunity to reflect on my own progress with software engineering, and if I could
help anyone out, that's just bonus points.

I spend a lot of my time with the Android SDK and all the fun that comes with it.
Android development can be difficult because of its nature: open source platform forked by
many manufacturers and other businesses, along with numerous devices with different hardware and API versions.
A good development environment can ease the pain that one will endure at times in your app development.

#### Step 1: Make sure your SDK installation is up to date as much as possible.

Why not automate that?

Here's a little script that can do that for you:

{% highlight bash %}
#!/bin/bash
#/usr/bin/expect
if [ -z $(which android) ] ; then
	echo "android binary not in your user's path."
	exit
fi
expect -c '
set timeout -1 ;
spawn android update sdk --no-ui;
expect {
    "Do you accept the license" { exp_send "y\r" ; exp_continue }
    eof
}
'
{% endhighlight %}

The most important line in this bash script is
`android update sdk --no-ui`, as this is the binary
that will update your sdk installation.
Unfortunately, this will require you to accept the licenses for all the new updates
so we'll use the expect utility to automate our "yes" responses for each prompt.

Run this in, say, cron, and enjoy your automatic updates.

#### Step 2: Automate your testing

The app I develop at work is supported on over 5000 devices (from api level 2.2 to 4.4.2, as of the time of this writing).
Testing new code and refactors on different devices/emulators/VMs takes time, so having a good test suite/continuous integration can facilitate development.

I will not get into how to go about writing/using tests for your app, there are many good tutorials.
For reference, check the Android developer site [here](http://developer.android.com/training/activity-testing/activity-basic-testing.html).

For the app I work on, I use the Robotium framework for its ease of use, minimal development time, and quick execution.
Remember I have a big range of devices properties to test against! Check out [robotium](https://code.google.com/p/robotium/).

Using Robotium, I have a nice collection of essential UI/integration tests, and unit tests to ensure
the experience is consistent from one device to the next, from one user to the next.

Despite the urge to buy a bunch of devices, I have to settle with emulators and virtual machines to span the properties of devices I normally
could not physically get my hands on (like API version, screen size, etc.). I have a significant amount of virtual device configurations, as the app can be supported on almost every kind of device.

Ideally I would love to automate this too! Get a coffee and have all the tests run on every emulator and device,
find the failing tests and get to work fixing any issues.

So I wrote a bash script to do that for me, partially because I liked the idea and mostly because I want to get better at shell scripting. It's not perfect, but it serves its intended purpose. Feel free to fork it and make it work for you, if so desired.

{% highlight bash %}
#!/bin/bash
#************************************************#
#              test_automator.sh                 #
#           written by Matt Miller               #
#                May 9, 2014                     #
#                                                #
#          Run robotium tests on connected       #
#	         android devices and emulators         #
#************************************************#

usage() {
	echo "Usage: $0 -p package-name [-t testcase] [-a] [-e]"
	echo "-a Run all tests in package"
	echo "-t testcase or testcases delimited by commas"
	echo "-p required: package name of your application's tests"
	echo "-e start emulators"
}

# Function to start avds we have created
function startEmulators {
	echo "Looking for AVDs..."
	avds=( $(android list avd | awk '/Name:/ {print $2}') )

	echo "Available AVDs:"

	for avd in "${avds[@]}"
	do
		echo "Starting emulator ", $avd
		emulator -avd $avd &
		avd_processes+=($!)
	done

	#wait just a bit for ADB to catch up
	sleep 5

	# Get all emulator ids that are waiting to boot
	# We must wait until the lock screen is showing to begin the testing.
	avd_to_boot=( $(adb devices | awk '/emulator-/ {print $1}') )

	#set the timeout form the config
	if [ -z $timeout ]; then
		echo "Timeout config not set, defaulting to 120 second boot timeout"
		TIMEOUT=120 #2 minute boot time for the emulators
	else
		TIMEOUT=$timeout
	fi

	#If we are booted add to the current booted devices
	declare -A booted_devices

	starttime=$(date +%s)

	while [ "${#booted_devices[@]}" -lt "${#avd_to_boot[@]}" ]; do
		currenttime=$(date +%s)
		diff=`expr $currenttime - $starttime`
		echo "Diff = $diff"
		if [ $diff -ge $TIMEOUT ]; then
			break
		fi
		for avd in "${avd_to_boot[@]}"
		do
			ret=$(adb -s $avd shell getprop init.svc.bootanim)
			echo "here: $ret"
			if [[ "$ret" == *"stopped"* ]]; then
				echo "$avd is booted!"
				booted_devices[$avd]=$avd
			fi
		done
		sleep 15
	done

	#kill the rest of the devices that didn't boot
	for avd in "${avd_to_boot[@]}"
	do
		if [ -z "$booted_devices[$avd]" ]; then
			adb -s $avd emu kill
		fi
	done
}

# Function to execute test on a device
function executeTest {
	device=$1

	# Make sure the app APK and the test APK are installed
	# And install them if not from the config or args

	res=$(adb -s $device shell pm list instrumentation $package)
	if [ -z  "$res" ]; then
		echo "Test apk not installed, installing from config"
		adb -s $device install $test_apk_location
   	fi

	res=$(adb -s $device shell pm list packages $package)
	if [ -z "$res" ]; then
		echo "App Apk not installed, installing from config"
		adb -s $device install $apk_location
	fi

	if [ "$runall" = true ]; then
		echo "Running all tests for device $device"
		#run all tests
		cmd="adb -s $1 shell am instrument -w -e package $package.test $package.test/android.test.InstrumentationTestRunner"
		echo $cmd
		adb -s $device shell am instrument -w -e package $package.test $package.test/android.test.InstrumentationTestRunner > $device.results &
	else
		echo "Running testcases $testcases for device $device"
		#run all tests
		cmd="adb -s $1 shell am instrument -w -e class $testcases $package.test/android.test.InstrumentationTestRunner"
		adb -s $device shell am instrument -w -e class $testcases $package.test/android.test.InstrumentationTestRunner >> $device.results &
	fi
}

function checkDependencies {
	# Dependency ADB (Android Debug Bridge)
	if [ -z $(which adb) ] ; then
		echo "adb binary not in your user's path. It is a dependency of this script."
		exit
	fi

	if [ -z $(which android) ] ; then
		echo "android binary not in your user's path. It is a dependency of this script."
		exit
	fi
}

# Catch ctrl-c and kill all subprocesses that the script started.
trap ctrl_c INT
function ctrl_c() {
	echo "Script terminating..."
	pkill -P $$
	exit
}

# our vars
package=
runall=true
testcases=
avd_processes=()

source test_config
echo "APK location = $apk_location"
echo "Test APK location = $test_apk_location"
# parse cmd line args
while getopts :p:aet: opt; do
	case $opt in
		p)
			package=$OPTARG
			;;
		a)
			runall=true
			;;
		t)
			runall=false
			testcases=$OPTARG
			;;
		e)
			startEmulators
			;;
		*)
			usage
			exit
			;;
	esac
done

#required
if [ -z "$package" ]; then
	echo "Missing package."
	usage
	exit
fi

checkDependencies

#Get all devices currently running on adb and store their ids into an array
output=$(adb devices)
devices=( $(printf "%s","$output" | awk '{if (NR!=1){print $1}}') )

#iterate over each device, running tests we've specified.
for device in "${devices[@]}"
do
	echo "Running test on device $device"
	executeTest $device
done
wait
echo
echo "Tests finished for all devices"
{% endhighlight %}

It's 200 lines of fun, so let me explain the gist of it.

#### Basic usage

{% highlight bash %}
usage() {
	echo "Usage: $0 -p package-name [-t testcase] [-a] [-e]"
	echo "-a Run all tests in package"
	echo "-t testcase or testcases delimited by commas"
	echo "-p required: package name of your application's tests"
	echo "-e start emulators"
}
{% endhighlight %}

This is relatively self-explanatory. Given the package name of your app,
(e.g. com.example.myapp), your tests are located in <i>package_name</i>/test.

You can run all the test cases or one particular test case, using the -a or -t param flags.

By default, the tests will run on all physical devices you have connected to your computer, interfacing through
ADB (the android debug bridge).

If you want to run the test(s) on your emulators also, use the -e flag.

#### Dependencies
{% highlight bash %}
function checkDependencies {
	# Dependency ADB (Android Debug Bridge)
	if [ -z $(which adb) ] ; then
		echo "adb binary not in your user's path. It is a dependency of this script."
		exit
	fi

	if [ -z $(which android) ] ; then
		echo "android binary not in your user's path. It is a dependency of this script."
		exit
	fi
}
{% endhighlight %}

The *android* and *adb* binaries must be in your PATH environment variable.
We use the android binary to install your app's APK on each device, as they may not have it,
and adb to run your tests on each device.

In part 2, I'll explain the rest of the code.
