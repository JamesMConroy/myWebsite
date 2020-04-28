---
title: "Setting up Zowe on Linux"
author: James Conroy
date: 2020-04-20T21:20:53-05:00
draft: true
---

I have been interested in learning COBOl and mainframes for a couple of months now, but I've been stymied by the difficulty of setting up a good mainframe emulator.
The Hercules mainframe emulator has a lot of conflicting install instructions ([and the top search result was last updated in 2010](http://www.hercules-390.org/));
so I was excited when the global pandemic highlighted the need for engineers who understood these systems.
In the past month IBM has started the [Open Mainframe Project](https://www.openmainframeproject.org/) with the intent of filling the skills gap for mainframes.
As part of the Open Mainframe project IBM has released [cobol programming course](https://github.com/openmainframeproject/cobol-programming-course), along with "COBOL Programming with VSCode" labs.
Once registered in this course you get access to a real life big iron mainframe so you don't have to worry about mucking around with emulators.

There is just one catch.
As the name implies the course uses VSCode relying on the Zowe Explorer plugin to grant you access to the mainframe.
If VSCode is not your preferred editor and you want to take the course, you're out of luck.
Or are you? The plugin is based on the Zowe project which has also created [zowe-cli](https://github.com/zowe/zowe-cli).
Using zowe-cli you should be able to perform all the labs using your preferred terminal and editor of choice.

Here's how to get it running on Arch Linux. If your roll with a different distro the steps should be similar.

## Installing Zowe
1) install npm you can use your distro's package manager of choice
2) install zowe though npm
``` bash
npm install @zowe/cli@zowe-v1-lts
```
On my system zowe was installed to `$HOME/.node_modules/bin/`, so I added that folder to my PATH variable. That way I don't need to type in the path to zowe every time I want to run a command.

## Register a Profile
Once you've registered for the IBM class IBM will sent you an email containing login information, and a warning that you're account will lock if not activated within 14 days.
Strangely the email has 3 IP address, and no instructions on how to activate your account.
The pdf distributed with the course also does not describe how to activate your account, in VSCode or otherwise.

To activate your account though the command line you can create a profile on your system that zowe can reference as default.
``` bash
zowe profiles create zosmf-profile zoweCLI --host https://IP.ADDRESS --port 000 --user username --pass password --reject-unauthorized false
```
Replace the IP address, user name and password with the ones provided in the email from IBM.

Now you can test your connection to the mainframe using your profile with:
``` bash
zowe zosmf check status
```

Alright now it's time to party like it's 1959.

-------------

## References

https://docs.zowe.org/stable/user-guide/cli-configuringcli.html#testing-zowe-cli-connection-to-z-osmf
