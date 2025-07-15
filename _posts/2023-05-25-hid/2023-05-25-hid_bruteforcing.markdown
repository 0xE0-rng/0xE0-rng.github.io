---
layout: single
toc: true
toc_label: false
toc_sticky: true
title:  "Bruteforcing Android Patterns Using HID Devices"
---

Unfortunately, my aunt died recently and left behind a perfectly good **Samsung Galaxy S7 FE** tablet. Unfortunately (or fortunately for you, the interested reader), we did not know the password for the device, which left us with one option: Factory Reset.

Sadly, all modern Android devices employ a feature Google calls **FRP** (Factory Reset Protection), which requires us to either:
- Enter the correct PIN / the correct Pattern
- or sign in with the connected Google Account.

Neither of those were available to me, so I had to find another solution:

## The Easy Way
Before going into the main topic of this blog post, I want to point out that there are much easier ways to perform what I did. There are still many different FRP bypasses available, and we might find a working one if searching for a while.
For this specific device and many others, Samsung was nice enough to add a small backdoor in their baseband which can be used to enable adb and use it to uninstall the app responsible for FRP. The flow goes something like this:
- Open the Dialer (e.g. through some talkback trickery)
- enter `*#0*#` in the Dialer to open the Samsung debug menu, which connects the phone's modem in serial mode to a USB-connected computer
- use specific AT commands to communicate with the modem and enable USB debugging
- we can then use the ADB shell to finish the setup manually (bypass FRP)

For more about this, I recommend this riskeco.com blog post: [^frp-bypass]. Note that for the tablet in question, another AT code than the one described in this blog might be needed.

## Human Interface Devices / Requirements on Linux
Human Interface Devices (HID) is a standard that devices can follow to communicate with your operating system to exchange information.
Without getting too deep into it [^hid-specs], the HID standard gives a common protocol for input devices.
Among others, this protocol is implemented by mice, keyboards or similar input devices.

At this point, I should mention that HID is already well known for malicious use cases known as HID-based attacks. Rubber Ducky is probably the most famous hardware device to inject keystrokes, and a 1 Euro ATTiny85 can replicate the same kind of attacks.

The Linux kernel provides the ability to emulate such a HID device using the **HID Gadget Driver** [^hid-gadget-driver].
Those gadget drivers can be enabled using the **configFS** kernel feature Linux provides. ConfigFS support must be enabled in the kernel, which has to be done at kernel build time.
For an example on Linux on how to set up a HID device, check out this repo [^hid-sample].
Some further reading on the USB Gadget API[^kernel-gadget-api].


### HID Gadget on Android and interacting with the gadget
On Android, configFS is enabled on most custom ROMs (e.g. LineageOS). For the configuration of the HID devices there is an app called **USB Gadget Tool**[^usb-gadget-tool] which makes configuring HID devices easier.

Once a HID gadget device is created and enabled, the `/dev/hidg0` device is available which we can use to write to connected HID devices.

The format to communicate with the HID Gadget is as follows:
We send three bytes
```
byte0 = 0x00 | 0x01 | 0x02 | 0x04  # to press no button, button 1, button 2 or 3
byte1 = deltaX                     # movement as a char with possible values between -256 and +255
byte2 = deltaY                     # y movement  -"-
```
Typical values for `deltaX` and `deltaY` are 1 or 2 for slow movement and 20 for very fast movement.

So after connecting our attacking Android device to a host device, we can test the HID gadget and write to the newly added device like so:
```
echo -ne \\x00\\x64\\x00 > /dev/hidg0
```
We should now see the mouse move by 10 units in the x direction on the host device.

## Introducing PatternBash
So what can we do with this knowledge? Inspired by the *Android Pin Bruteforcing Project*[^pin-bruteforce-repo], I have created a bash script that will bruteforce the pattern lock on an Android device. I called it **PatternBash** and it is available on its dedicated repo here [^patternbash-repo].

The problem with bruteforcing is the timeout between failed attempts grows exponentially. However, for the tablet in question (and many other devices), we can actually factory reset the device again, which will reset the timeout every time we do.
This is an issue that in a different form also existed on iPhones, which made bruteforcing PINs feasible for a while there too[^iphone-brute].

### Features
PatternBash tool has the following features:
- A resume function that allows you to pick up between attempts
- A list of Android patterns based on the distance between the pattern points, which is available here on its dedicated repo on GitHub [^all-pattern-repo]
- A tool to create optimized bruteforce patterns based on the paper [^pattern-frequency]
- An installer script to just push and run the APK
- A usage menu to set up the location of the dots on the screen
- A backup and resume feature to pause the bruteforcing
- Auto backup of the current progress

### Challenges and Known Issues
When writing PatternBash, I came across a couple of challenges which I will describe in the following.

#### Skipping Inputs
I have realized that sometimes the device skips over certain HID inputs, which throws off the whole coordinates. I investigated this issue a lot and actually rewrote the tool to use the HID gadget binary provided by the Linux team instead of direct bash-to-device communication.
Unfortunately, this was also unsuccessful, so my only fix was to go slower and limit the inputs.

#### Mouse Positioning
There is actually no way to know where the mouse is currently located on the device. The solution to work around this is to move the mouse to the very top left and call this location (0,0). From there on out, we can access different locations using coordinates.

#### Pathfinding
To move the mouse between points, we need pathfinding. Because this isn't trivial in bash, I implemented a simple approach:
```bash
    while [ "$CURRENT_X" -ne "$JUMP_X" ] || [ "$CURRENT_Y" -ne "$JUMP_Y" ]; do
        if [[ $CURRENT_X -lt $JUMP_X ]]; then
            move_right $MOVE_DISTANCE
        fi
        if [[ $CURRENT_X -gt $JUMP_X ]]; then
            move_left $MOVE_DISTANCE
        fi
        if [[ $CURRENT_Y -lt $JUMP_Y ]]; then
            move_down $MOVE_DISTANCE
        fi
        if [[ $CURRENT_Y -gt $JUMP_Y ]]; then
            move_up $MOVE_DISTANCE
        fi
    done
```
This works but breaks for complex patterns like connecting 1 and 8, potentially touching other points.

#### Random Phone Resets
Some devices reboot during bruteforcing. My tool handles this by pausing at a set attempt count using the `SHUTDOWN_AT` variable.

#### Feedback Limitations
We can't get feedback through HID alone to know when we've succeeded. Existing products use a sensor taped onto the device to measure brightness. Other solutions would be to just film the device or create a picture after each attempt and process the image.

### Future Work
Raspbian actually also has **ConfigFS** support, so my script can be run on a Raspberry Pi. This same Raspberry Pi can be used to drive stepper motors that automatically perform a factory reset on the device. I have experimented on this but did not create a running prototype. Maybe in another blog post...

## Sources
[^frp-bypass]: [Riskeco: Analysis of a Samsung FRP Bypass](https://blog-cyber.riskeco.com/en/analysis-of-samsung-frp-bypass/)
[^pattern-frequency]: [Loge, Marte et al: On User Choice for Android Unlock Patterns](https://web.archive.org/web/20230527233502/http://www.usablesecurity.net/USEC/NDSS/wp-content/uploads/2018/03/01-on-user-choice-for-android-unlock-patterns.pdf)
[^pin-bruteforce-repo]: [The Android PIN Bruteforce Github Repo](https://github.com/urbanadventurer/Android-PIN-Bruteforce)
[^patternbash-repo]: [PatternBash Github Repo](https://github.com/0xE0-rng/PatternBash)
[^usb-gadget-tool]: [Android USB Gadget Github Repo](https://github.com/tejado/android-usb-gadget)
[^all-pattern-repo]: [My AndroidPatternLock Fork](https://github.com/0xE0-rng/AndroidPatternLock/tree/50eac641818bfb03b1a102c39b4e3c71dafac1bd)
[^kernel-gadget-api]: [kernel.org USB Gadget Driver Documentation](https://www.kernel.org/doc/html/v5.0/driver-api/usb/gadget.html)
[^hid-gadget-driver]: [kernel.org HID Gadget Documentation](https://www.kernel.org/doc/html/latest/usb/gadget_hid.html)
[^hid-specs]: [usb.org HID Protocol Specification](https://usb.org/sites/default/files/hut1_4.pdf)
[^hid-sample]: [HID Keyboard Gadget Sample by dlyoung](https://github.com/qlyoung/keyboard-gadget/blob/master/gadget-setup.sh)
[^iphone-brute]: [Mitre CVE-2014-4451](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-4451)
[^iphone-burte2]: [Blackbox Bruteforcing](https://insinuator.net/2015/04/apple-ios-pin-bruteforce/)