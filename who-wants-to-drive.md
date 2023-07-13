Original post date: 2021-01-25

# Who wants to drive?

Driver assistance and self-driving car systems are in the beta stages as we speak. In the next decade, we'll more than likely see robo-taxis on the rise, nobody in the driver seat of 18-wheelers and maybe you'll even buy a Tesla for only a few thousands bucks. The catch for acquiring such a subsidized car, similar to the deals to get an iPhone so cheap, is that you agree that it can go uber people around instead of sitting "under-utilized" in your driveway or a parking garage for most of the day.

As a wise man once said, you can't have safety without security. There's some awesome start-ups out there doing these types of car + computer innovations with machine learning. Many are getting closer to something that your average consumer would see the value in (vs an expensive toy that only drives itself in controlled environments, under perfect conditions).

Comma.ai is one of those companies, founded by George Hotz. How to describe Geohot.. he's like the "The Kanye of Computers". We need more artists and visionaries like him. People like George have a knack for getting stuff done rather than just talking about it. During one of his talks on stage,  he goes on to say that he respects Tesla for actually shipping something.

If you're into security and technology in general, wouldn't you want to test drive of the latest tech? Figure out how this thing works? See if there are any good bugs there? Sure you would.

*All you need* is a [car](https://automobiles.honda.com/civic) and the [kit](https://comma.ai/shop/products/comma-two-devkit).

# Comma and Openpilot

## Prologue

This post is meant to provide an overview and details for a few different areas of the Comma device and Openpilot platform. It is not meant to be a full guide or deep dive in any way, just a walkthrough of different security and privacy aspects of using the tech. Hopefully this and further research will provide insights and data for the industry to continue work towards safe and secure driving systems.

*Disclaimer: DO NOT attempt to recreate or use any of this research on physical cars. The information here is for educational purposes only.*

## What is it and how does it work

Comma is a company that makes open source driver assist technology. No one is close to the dream of actual self-driving cars yet, but the road is paved farther and farther even year. Their hottest product at this time is the comma two, which is device you install in your car and is then setup with their openpilot software to make the device actually work.

Things it does well:
- Brakes and resumes for the driver in traffic
- Steers pretty well depending on your speed (no sharp curves)
- Simply signal, look and nudge it over for lane changing
- Limited yet impressive driving on roads without lines
- It requires you to pay attention to the road via active driver monitoring

Things that can be improved (as of late 2020):
- Sometimes it wants to take exits instead of keeping straight on the highway
- Exhibited limited visibility for seeing cars in front during rain (particularly white cars for some reason)

What it doesn’t do (yet):
- Doesn’t stop at red lights, stop signs, unusual road conditions, etc
- It can’t make full turns at intersections
- Parallel parking? Button to come pick me up at the door? - Go make me money?

**Being able to not touch the steering wheel for a half hour is kind of amazing**

## Open Source Driver Assist System

[Openpilot](https://github.com/commaai/openpilot) is the brains of the comma two. You can choose to give your car a "software upgrade" to perform automated functions better. It utilizes or improves upon the experience of Adaptive Cruise Control, Automated Lane Centering, Forward Collision Warning and Lane Departure Warning while also adding Driver Monitoring as a safety feature. This can compete with the likes of Tesla Autopilot and GM Super Cruise, except that it's open source software.

Openpilot deploys a bunch of local services written in Python to facilitate data going from the car to Comma devices and visa-versa. To get an in-depth view of how this works, refer to the excellent [article](https://desosa.nl/projects/openpilot/2020/03/11/from-vision-to-architecture.html) on [desosa.nl](desosa.nl) for a more complete explanation and [design diagram](https://desosa.nl/projects/openpilot/2020/03/11/images/openpilot/essay2/architecture-7.png).

# Security

The primary scope was on the *default* remote attack surface.

* "Could someone compromise the device over the network, via the Internet or WiFi?"

For example, is it possible to compromise the car via the Comma two after the initial setup, assuming the end user didn't change a thing.

**The answer was no**, as far as we could tell at this time.

After the first setup, there were no listening services or open ports found, nor does the UI lend itself to client-side attacks. One caveat which we'll expound upon an attack related to spoofing the installation URL, but that involves tricking the user into using that particular URL, which one may argue is hard to rationalize and defend against for "a dev kit".

The secondary scope was *any* remote attack surface as far as could be enabled by an end user via the UI.

* "Could someone compromise the device if we flip on WiFi, SSH, etc?"

**The answer was perhaps**, but by design. The default SSH [key](https://github.com/commaai/openpilot/blob/master/tools/ssh/id_rsa) is documented and for now... "its a dev kit", but continue into the research section for more details.

The last scope was the local attack surface.

* "Could someone who has compromised the device affect the car in a significant way?"

**The answer was probably not**, or at least not without substaintial effort. Also, the included [Panda](https://github.com/commaai/panda) device will enforce the [safety model](https://github.com/commaai/panda#safety-model) regardless of what funky code could possibly run on the Comma device itself, as quoting a conversation with a developer.

> There is still the panda between the C2 and the car which will always maintain the safety model no matter what code the C2 is running. At some point we will lock the bootloader in the panda allowing only signed code to run (requiring a physical paw/key to unlock the bootloader).

## Research

Knowing the CEO responsible for building this device had previously hacked the PS3, iPhone, Galaxy S5, etc, a reasonable assumption is that there wouldn't be an easy bugs or obvious security holes in the design. No way he would ship the product like that. But maybe they did make some trade-offs here and there, missed a few gotchas, so that was the thinking when starting the research.

Most of the suggestions for hardening against attacks revolves around how Openpilot does SSH and the local environment. Sorry, no default unauth RCEs today!

### Local System

**/dev/shm permissions**

/dev/shm is used to share memory and buffers between various openpilot components. This includes data from the can bus, gps sensor, car controls and state, etc. Openpilot uses this to make sure devices and sensors are operational and functioning properly.

When the device notices that car has been started, it creates many more files in the directory.

```
# ls
androidLog        driverState          gpsPlannerPoints       liveParameters      orbOdometry        trimbleGnss
applanixLocation  encodeIdx            health                 liveTracks          orbslamCorrection  ubloxGnss
applanixRaw       ethernetData         kalmanOdometry         logMessage          pathPlan           ubloxRaw
cameraOdometry    features             lidarPts               model               plan               uiLayoutState
can               frame                liveCalibration        modelV2             procLog            uiNavigationEvent
carControl        frontEncodeIdx       liveLocation           navStatus           qcomGnss           wideEncodeIdx
carEvents         frontFrame           liveLocationCorrected  navUpdate           radarState         wideFrame
carParams         gpsLocation          liveLocationKalman     offroadLayout       sendcan
carState          gpsLocationExternal  liveLocationTiming     orbFeaturesSummary  sensorEvents
clocks            gpsLocationTrimble   liveLongitudinalMpc    orbKeyFrame         thermal
controlsState     gpsNMEA              liveMapData            orbLocation         thumbnail
dMonitoringState  gpsPlannerPlan       liveMpc                orbObservation      trafficEvents
```

Permissions for this directory are too open though with it being RWX for everyone (`chmod 777`). It's recommended to be either 700 or 770 to have some sort of defense in depth. It makes sense because poking at stuff in /dev/shm seems like one of the things we’d only want comma components / the root user to do. That way, if some random service did get compromised on the device and one was able to execute code as a lower privileged user, there would be much less to worry about up to that point.

There's better tools for this, but lets try to take a quick peek on what some of the data looks like in `androidLog`.

```
# xxd androidLog | head
00000000: 0100 0000 0000 0000 605a 2600 0000 0000  ........`Z&.....
00000010: 3c0a 0000 66fa 5d80 605a 2600 0000 0000  <...f.].`Z&.....
00000020: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000030: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000040: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000050: 0000 0000 0000 0000 0100 0000 0000 0000  ................
00000060: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000070: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000080: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000090: 0000 0000 0000 0000 260f 0000 daa9 02db  ........&.......
```

As we were pretty curious about "what happens if you actually poke at stuff in there?", there was a simple test (while the device is ON, but **NOT WHILE DRIVING**): `echo 0 > /dev/shm/SOMETHING` and check how the device responds.

* can

`CAN Error: Check Connections on UI`

* health

`UI reverts from driving mode to main screen, shows "No vehicle" on left side`

* sensorEvents

```
No Data from Device Sensors
Reboot your Device
```

* driverState

```
Camera Malfunction
Contact Support
```

* controlsState

```
(Red screen with loud constant beeping)

TAKE CONTROL IMMEDIATELY
Controls Unresponsive
```

Whew, that was sort of exciting, but at the same time seeing the (lack of) reaction from the car, it felt quiet safe. Comma seems to be handling rudimentary bad state conditions pretty well. And the conclusion was that while none of these seemed to have an affect on the car itself, many of them could at least disable the device and/or annoy the driver.

This again is a reason why local hardening such as least privilege permissions, especially on important areas of the filesystem. During testing, we of course used the default root account to perform the demos only because there wasn't another available user on the system (yet) to test with, but the open permissions are clear. If someone did manage to get a non-privileged shell on the device, hardening improvements such as no RWX for everyone means they could not directly affect that part of the system.  They'd need +1 privilege escalation bug or to take an indirect route, such as getting a local service to ferry malformed data over for them, in order to have an impact.

**Processes Should Run As Non-root**

Processes on the device are running as various users such as `root, system, nobody, sensors, shell, drm, media, media_rw, rfs, radio, net_admin, gps, wifi, u0_a[NN]`, but pretty much all Comma processes run as root.

Dropping privileges after start or architecting the system in a way that it doesn't need high privileges to run is something that the industry has been getting much better at these days. It's an effective strategy because it prevents "insta-root" upon compromise. If there are ever bugs in these processes, they don’t automatically lead to a root shell. An attacker with only user privileges is an attacker that now needs another bug to do real damage.

Adding or reusing a local sandboxed user and doing a little work to the local services to make sure everybody is still communicating correctly would be the easiest approach. Maybe one way to run processes as another user instead of root would involve modifying `manager.py` and move `os.setpgrp() in preexec_fn=` to a new function that also calls setuid/setgid for the target account.

Although this Android device is "different" in that there are not just a bunch of apps running on it like your phone, but instead of ton of Python scripts. So the target account would also need python binary execution permissions so that the user could run it. Ran through a few small tests to see if it would work, but it seems like it needs a few more environmental changes to make it function properly within the platform.

Taking a look at Comma's code, the services seem to be written defensively and simple enough code to audit. Though whilst no good bugs were discovered or exploited in the local services, assuming that there may be a bug or two one day, running the services as non-root is an easy win for security. This would limit the blast radius and minimize the attacker's options if bugs were found and had reliable exploits.

### SSH

SSH is disabled by default, but can be turned on via the UI. It listens on port `8022`. There are a few official ways to access the device over SSH once enabled. 

1) Connect to a WiFi access point (or hotspot) and SSH over the local network
2) Turn on the WiFi hotspot and connect to the device's network and SSH in
3) Enable Comma Prime so the device uses LTE and SSH in [over the Internet](https://ssh.comma.ai)

At least for the first two scenarios, the device is configured to use a default key for authentication unless configured otherwise. The thing is, the default private key is [public](https://github.com/commaai/openpilot/blob/master/tools/ssh/id_rsa). This means that if you setup the device, connect it to a network and just flip SSH on, then anyone who can connect to the device with the key and drop into a root shell.

```
$ ssh root@10.1.1.14 -p 8022 -i id_rsa
root@localhost:/data/openpilot$

root@localhost:/data/openpilot$ ls
CONTRIBUTING.md  RELEASES.md  apk     installer            launch_env.sh        opendbc    prebuilt  scripts
LICENSE          SAFETY.md    cereal  launch.sh            launch_openpilot.sh  panda      pyextra   selfdrive
README.md        SConstruct   common  launch_chffrplus.sh  models               phonelibs  rednose   site_scons
root@localhost:/data/openpilot$
```

Now of course they'd need to be on the same network as the device, maybe scanning for the non-standard port 8022, and have the comma github key on hand to try it, but it would be pretty straightforward to put together a script that scans and reports back if happens to finds something.

If you don't want to use Comma prime, you could also expose the device to the Internet using [ngrok](https://ngrok.com) or something similar. Change the SSH key to something not public, remount `/system` to add some name servers to `/system/etc/resolv.conf` and copy the arm64 ngrok binary onto the device to get started.

```
mount -o rw,remount /system
...
HOME=/tmp ./ngrok authtoken 1zfaqEiDPCdDV8… && ./ngrok tcp 8022
```

Could you just scan the Internet for port 8022 and find a bunch of cars? That's unlikely due to additional SSH hardening which restricts connections to be from RFC1918 ranges, aka private networks only. Smart!

```
$ cat /data/params/d/GithubSshKeys
from="10.0.0.0/8,172.16.0.0/12,192.168.0.0/16" ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC+iXXq30Tq+J5NKat3KWHCzcmwZ55nG.....
```

So use the default key + open IP range + ngrok only if you want to play a game of Russian Roulette with your car. "Don't do that" :>

You can also enter your GitHub username in the UI to lock SSH to your GitHub SSH key. Read more on this commasupport [article](https://commasupport.zendesk.com/hc/en-us/articles/360037705013-What-is-comma-prime-) and the comma ssh [docs](https://ssh.comma.ai/).

And kudos for being cool on some privacy aspects. Please continue :-)

> comma does not have and does not want remote access to your device (unlike certain electric car companies), because your device is yours. You alone control your private key for remote access. Our security model is the same as SSH, so a compromise would likely mean someone has gained access to your private key (or an OpenSSH 0-day just compromised 80% of the internet!). comma does not have your private key and a comma employee will never ask you for your private key.

#### Defenses

Change the default key.

We'd expect as the Comma products mature that there will not be public default keys. But in future devices, for the Comma 3 perhaps, the SSH key should not be public.

Picking the SSH key off of the openpilot github is great for easy mode, but it probably should be randomly generated either 1) per device or 2) per openpilot install or 3) a user-provided key is mandated (right now github key is optional) so you’re less scared of connecting this thing to the wifis.

### Device Setup

Putting an installation warning on URLs other than openpilot official is a good start.

Measures should be taken to warn users in case they typo or are otherwise led to use a URL that looks or sounds like the official openpilot one (we all know the comma two is meant to use openpilot.comma.ai, it can still be non-obvious just with an addition confirmation from the other if they want to really install something else).

One typo or bad URL shouldn’t be a straight shot to running a backdoored image. For example, a GitHub repo that does something as seemingly innocent as enabling SSH by default opens up the device for compromise if you're on the same LAN. Anything is possible with a few more keystrokes, though.

# Misc

## Comma Connect

The Comma Connect [app](https://play.google.com/store/apps/details?id=ai.comma.connect&hl=en_US&gl=US) lets you replay driving videos, turn on cameras, buy Comma Prime, etc. You can dump the JavaScript and get an idea of how it works, which Comma server and cloud providers (Azure) are involved and the APIs it calls. Simply grepping through the code (amass + subfinder for even more) can give you insight into a lot of different pieces of their infrastructure. Of course you can understand how the app works by looking through the various functions as well.

```
fetchDevice, fetchDevices, fetchDeviceLocations, fetchDeviceStats, unpairDevice, takeDeviceSnapshot, fetchDeviceCarHealth, setDeviceAlias, fetchSimInfo, fetchLocation, fetchVehicles, setDeviceVehicleId, grantDeviceReadPermission, removeDeviceReadPermission, fetchDeviceOwner, fetchRoutes, getSegmentMetadata, getRouteInfo, getShareSignature, getRouteSegments, listRoutes, pilotPair, unpair, fetchMakes, fetchModels, fetchVehicle, getSubscription, payForPrime, getPaymentMethod, updatePaymentMethod, cancelPrime, ...
```

There's also the comma API [spec](https://api.comma.ai/#comma-api-spec) which details how it does authentication with JWT tokens, how each dongle has a unique ID, lots of interesting stuff. Everything seemed to make sense there from a security perspective, pretty well thought out and designed.

The openpilot repo also has some [tools](https://github.com/commaai/openpilot/blob/master/tools/lib/api.py) for authenticating to APIs as well.

## Frida

You can also setup Frida on the device which can help give you insight into various processes and things happening on the system. It's something we wish there was more time to explore in this particular environment as it's a very powerful tool to weild, especially for security research on embedded systems. Just remount `/system` as RW, grab frida for [arm64](https://github.com/frida/frida/releases/download/14.0.8/frida-server-14.0.8-android-arm64.xz), copy it over to the device and expose it to the network.

```
# mount -o remount,rw /system
...
# ./frida-server -l 0.0.0.0 -D
```

Then install frida tools and you can now use the tool suite.

```
$ pip install frida-tools

$ frida-ps -H 10.1.1.14
 PID  Name
----  -----------------------------
2098  _sensord
2087  _ui
3370  bash
2258  boardd
.....

$ frida-trace -H 10.1.1.14 -i "open*" [selfdrive process maybe]
```

# Privacy

As much as we'd like more privacy with our devices these days, adding more cameras and sensors to your environment is just going to increase data collection, at least until some real innovations make market sense and are demanded by consumers. There are real trade-offs with advances in technology and *hopefully* we'll continue to decide whats ok and what's not far into the future.

That being said, there are road facing cameras (necessary to assist) and driver facing (for safety features like driver monitoring) infrared cameras on the Comma two. As far as we could tell, you cannot turn off recording and uploading trips on the road camera. This makes one a little nervous (if at all concerned about privacy) as it at least in the US, it records everything in front of, not just the road, but people and places too. The driver facing infrared camera is off by default (as of now) and can be turned on in the UI.

Of course there's a good reason to record this data: They use it to improve the systems capabilities and learning models. Though it would be nice in the future if they could e.g. *blur* out everything that isn't the road or otherwise anything that isn't quality feedback, while still ensure the data is useful for training their models.

There's also a connection between the device and Comma servers. Other than some plain-text HTTP health checks containing some metadata, most of the communication is at least encrypted.

One could do things such as null routing specific Comma server or local services with `route add xx.xx.xx.xx gw 127.0.0.1 lo`, but that could affect the driving experience and the device may function or may not, and this could change over time as infrastructure grows. Best case, all your pretty driving stats go away and the UI gets funky. Also if the video recordings bother you, there are some hacky things around like messing with the `ROOT` directory in `loggerd/config.py` that can help disable the storage and therefore auto-uploading of data.

Media and logs are stored in `/data/media/0/realdata/` and the structure looks like this.

- fcamera.hevc

High quality driving video clip

- qcamera.ts

mpeg2 quality driving video clip

- qlog.bz

Compressed file of binary route data?

- rlog.bz

Compressed file for binary route data?

Comma doesn't actually exercise that much control over your experience (yet). Besides "need to update" stuff, there was no remote "kill switch" or other things one would imagine other car makers will only be implementing more of.

So will it ever be possible to go "ghost driving" with autonomous cars? Manufacturers will do their best to make sure the answer is no, but hackers gonna hack. 

# Conclusion

It's a great driver assist system, super easy to install and for a shoe string budget. It definitely is killing it. What's better right now, other than Tesla's Autopilot? All from a small start-up in SoCal.

**Did it feel safe driving with it installed?**

Yes, it was pretty clear they take driver safety very seriously and the system they've created automates a lot of human functions (probably driving better than many drivers).

**Is the device secure?**

Well, if you think its a “dev kit”, of course it's not going to be super locked down. But security doesn’t need to get in the way of innovation, seriously. I'm sure they'll spend a ton more resources on hardening once they get 10, 50 and 100k cars on the road. Just know what it is and what it isn’t.

**Does it convert your car into a textbook self-driving car?**

No, but its still cool and open source, and it shipped!!!1 which deserves a lot of respect. It does indeed "Make Driving Chill" (as their motto goes).

Hopefully this research is helpful to Comma and also gets more folks interested in car hacking related stuff. If you're in the latter camp, Charlie and Chris's years worth of documenting and presenting their tinkering around is a [must read](http://illmatics.com/carhacking.html).

Looking at the security model of the device, taking into account that its a v2 likely of many more to come, the security model has been pretty well thought out. Sure, you can modify it as you wish, but it would take some effort to make it a vulnerable mess honestly. Comma on the network doesn't have any weird ports listening for connections, only SSH on a non-standard port and IF you explicitly turn it on. Of course there's plenty of hardening to do and hopefully it will make sense for them to put more resources on that once they start making production devices.

As they say, you can't have safety without security. But as a security nerd, it took a long time for me to accept this: Security is obviously very important, but it's not #1. The product can be the most secure thing in the world, but if it never ships, it doesn't exist. You don't have to choose one, but you do have to strike a keen balance if you want to produce value for society. So far so good for Comma.

You stay classy, [San Diego](https://medium.com/@comma_ai/comma-ai-moves-to-san-diego-7db472a417ed)!

# References

- http://illmatics.com/carhacking.html
- https://ssh.comma.ai
- https://github.com/commaai/openpilot
- https://github.com/commaai/openpilot/blob/master/tools/ssh/id_rsa
- https://github.com/commaai/openpilot/blob/devel/SAFETY.md
- https://commasupport.zendesk.com/hc/en-us/articles/360037705013-What-is-comma-prime-
- https://desosa.nl/projects/openpilot/2020/03/11/from-vision-to-architecture.html
