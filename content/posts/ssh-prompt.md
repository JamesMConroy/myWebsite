+++
title = "Custom Prompt on Remote Servers"
date = "2020-05-02"
author = "James Conroy"
cover = ""
tags = ["ssh", "scripting"]
keywords = ["", ""]
description = ""
showFullContent = false
+++

# The Insanity

In my job I have to ssh into a lot of servers to do preventive and routine maintenance.
These servers tend to have a patch work of settings and configurations.
During a shift I can touch Redhat, AIX, and HP-UX servers.
These servers do not have a consistent prompt.
Some show `user@hostname $`, others just display `bash-#.#.#`, and several are just a lonely `$` devoid of any context to keep you from bricking the server or deleting HIPPA records.

To make matters worse each of these servers use a different default shell.
The Redhat boxes run bash, AIX uses ksh by default, and the HP-UX servers have a strictly POSIX compliant version of ksh.
Usually when presented with systems using different defaults I'd let them be and try and learn in case it came in handy latter, but IBM's version of ksh does not have tab completion.

Let me repeat that.

> In k-shell tab completion does not exist

To get completion in ksh you either have to mess arround with settings or use the default keybinding ESC+\ *shudder*.
So say good bye to all that muscle memory when you're cding into a directory or typing out a long file name; because, in IBM that muscle memory gets you a tab.
Oh and backspace doesn't work on HP-UX, so have fun fixing that tab you just inserted by accident.

I could of course learn to live with the quirks of ksh and get into the habit of checking your assumptions on unfamiliar servers, but this is IT where the creatively lazy rule.

# Automation !!!

Turns out ssh is useful for more than just opining interactive sessions on remote servers.
Secure Shell can run any simgle command, or a combination of commands on a remote host and return their output into your stdout.
You can also take some enviroment varables along with you when you ssh into another host.
To keep myself same I created a bash function to be a wrapper for ssh.
The function gather's facts about the remote server you are trying to access, and then launches you into an remote interactive session with some appropriate environment variables.

``` bash
ssh () {
	# gather facts about the remote host
	local remoteFacts=$(command ssh -t username@"$@" "bash --version | head -n1 ; uname")

	# if remoteFacts does not contain not (ie the server has bash) launch an interactive bash session with these env varables
	if [[ ! remoteFacts ~= 'not' ]]
	then
		command ssh -t "username@"$@" "export PS1='set your prompt string here' ; bash -i"
	# if the serrver does not have bash
	elfi [[ remoteFacts ~= 'aix' ]]
	then
		# Same thing but with AIX settings
	elif [[ remoteFacts ~= 'HP-Uz' ]]
	then
		# Same thing but with HP-UX settings
	else 
		# default incase no other logic is correct
	fi
```

This simple wrapper lets me use bash when available, set up a useful prompt string, and fix the various quirks of ksh when it's unavoidable.

# Further Reading

Tom Ryder has two excellent posts on [custom commands](https://sanctum.geek.nz/arabesque/custom-commands/) on Unix and your [prompt string](https://sanctum.geek.nz/arabesque/bash-prompts/)
