---
title: Controlling your Windows 10 PC with HomeKit
date: 2020-01-25T00:00:00+01:00
draft: false
tags: 
    - home automation
    - homekit
category: home
keywords: 
    - home automation
    - homekit
---

Lately I've been going into a home automation spree, trying to automate my house as much as possible to avoid wasting time on repetitive stuff. While I am trying to not take it to [the extreme](https://github.com/NARKOZ/hacker-scripts), there's actually a lot of nifty stuff you can either buy or quickly "hack" with a Raspberry Pi to make your life much easier.

While I started by getting a cheap Echo Dot and running everything with Alexa, I quickly felt the system to be quite limited and messy - just like many other Amazon apps, the Alexa app has a very weird UI that seems to not have the main focus on home automation. Also, the routines you can build in there are quite limited. So I decided to look for alternatives:

* [Google's solutions](https://store.google.com/ie/category/connected_home) seem to be pretty good, but I don't want to wire tap my entire house for them.
* [Home Assistant](https://www.home-assistant.io/) and the rest of the "indie" solutions are very powerful, but a pain in the arse to set up and configure, and they also don't give me the confidence that a solution backed by a big company does. You also need to pay a monthly subscription to have access to everything.
* [Apple's HomeKit](https://developer.apple.com/homekit/) is on a very weird spot because of their greediness with the MFi program. While being a very robust and technically compliant solution, and being the only AAA option that seems to respect your privacy, there very few accessories that work with it and the ones that do tend to be quite expensive. That is, until you find [Homebridge](https://homebridge.io/).

Homebridge is a NodeJS solution that runs as a server and basically emulates a bridge (like the ones you get with stuff like Phillips Hue) through which you can add your own programmed accessories. You define your own scripts (that do things like interact with the REST API of a smart plug to control it), and Homebridge will expose them to HomeKit as HomeKit-defined accessories.

This means that anyone with enough skills can define Homekit accessories that not just control smart home stuff, but basically do anything. You can define a light switch in Homekit that instead of controlling a light, runs an SSH command. Or in the case of this blogpost, you can run a complex set of commands to turn on, monitor and turn off a Windows PC - like [Alex Gustaffson](https://github.com/AlexGustafsson) did in [homebridge-wol](https://www.npmjs.com/package/homebridge-wol).

Since Microsoft woke up and started doing interesting stuff with Windows 10 now [you can remotely control your PC via SSH](https://news.ycombinator.com/item?id=15904265), which means you can turn it on/off at will from something like a Raspberry Pi.

Let's get started.

# Getting the OpenSSH server in Windows 10

Microsoft prepared [very good docs for it](https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse), but basically:

1. Go to Windows 10 settings
2. Go to apps
3. Click in 'Optional features'
4. Click in 'Add a feature'
5. Select 'OpenSSH server'. You can install the client too for fun.
6. Reboot

Now there will be a new service for the SSH daemon, which you will have to configure to accept incoming connections. Since we are not idiots we'll also configure it to reject password authentication and only accept SSH keys. The configuration process for this is a pain in the arse due to the differences between Linux users and Windows ones, and user [n0rd](https://stackoverflow.com/users/31782/n0rd) made [a very good guide for it in StackOverflow](https://stackoverflow.com/a/50502015). Follow it to generate the keys and configure the service properly, then edit the `C:\ProgramData\ssh\sshd_config` file to set the `#PasswordAuthentication yes` to `PasswordAuthentication no`.

If you are wondering what your Windows username is (since Microsoft made a clusterfuck of it now that you login with your Microsoft account), you can retrieve it via the windows shell:

1. In Windows, press the Windows key + R
2. Type `cmd`, hit enter
3. In the command shell, use the `whoami` command to get your username. You'll get something like:

`DESKTOP\john`

The username is the 'john' part, 'DESKTOP' is your computer's network name which is not needed.

You will also have to configure [wake-on-lan](https://en.wikipedia.org/wiki/Wake-on-LAN) for your computer. There's a lot of tutorials online for it (like [this one](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=3&cad=rja&uact=8&ved=2ahUKEwiq6ej2_p7nAhW0QUEAHf-3BWIQFjACegQIDBAH&url=https%3A%2F%2Fwww.groovypost.com%2Fhowto%2Fenable-wake-on-lan-windows-10%2F&usg=AOvVaw2f4sTifO9oM4WmI7Rz4Qaw)) so I won't go through it, but keep in mind this is something you'll most likely also need to enable on the BIOS for it to work - and not all motherboards support it. Google your motherboard model plus 'wake on lan' for more info.

Now you can try to log into your computer and control it. Log into a second computer within the same LAN and ssh into your Windows computer:

`ssh -i <your_private_key_file> <username>@<ip_address>`

If it worked, you should see the Windows command shell. Run the following to initiate shutdown in 30 seconds:

`shutdown /a /t 30`

You should see a system notification in your Windows computer warning you of the remote-initiated shutdown, and the computer should automatically power off in 30 seconds. Now in order to wake it up, send a magic packet to the MAC address of your computer's network interface. If you are running this from a Raspberry Pi you'll probably have Raspbian which means you'll have the `wakeonlan` command already available, but if not a quick Google search should show you a thousand billion tools to do it:

`wakeonlan <your_mac_address>`

Your computer should now boot up if wake-on-lan is properly configured.

# Installing and configuring Homebridge

I would simply recommend Homebridge's own tutorials on how to install the software and run it as a service, they are very good. Again, you should probably run this on Raspbian so the guide would be [this one](https://github.com/nfarina/homebridge/wiki/Install-Homebridge-on-Raspbian). Once it's installed, install the `homebridge-wol` package through npm. Now there's a small modification you'll have to do to the process detailed in the Homebridge wiki: in order to run ssh commands, the `homebridge` service user you create in the guide will need a home directory to create the `.ssh` directory. You can do so by running:

`mkhomedir_helper homebridge`

 Keep in mind you'll also need to provide the private key to authenticate via SSH in a directory that is accesible by the `homebridge` user.

 Now you'll have to add your Windows laptop to Homebridge as an accessory. Here's a configuration sample, similar to the ones found in the `homebridge-wol` package information:

 ```
"accessories": [
  {
    "accessory": "NetworkDevice",
    "name": "Windows PC",
    "mac": "<your_mac_address>",
    "ip": "<your_ip_address>",
    "pingInterval": 30,
    "wakeGraceTime": 30,
    "shutdownGraceTime": 30,
    "shutdownCommand": "ssh -i <your_ssh_key> <username>@<your_ip_address> shutdown /s /t 15"
  }
]
 ```

Restart the Homebridge service and your Windows computer should show up as a lights switch in your iOS Home app.

Enjoy!
