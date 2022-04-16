+++
title = "Mirroring All the Nim Packages"
date = "2022-04-16"
author = "James Conroy"
cover = ""
tags = ["nim", "scripting", "apache", "git"]
showFullContent = false
+++

# The Problem

At [$DAY_JOB]({{< ref "about" >}} "About") we like to keep our development environment isolated from the internet[^1].
This means we have to mirror stuff like the Ubuntu and CentOS package repositories, so they are available in our dev environment.

[^1]: So the programmers don't spend *all* their time watching cat videos.

Recently my company has decided to try and develop some tools in [Nim](https://nim-lang.org), meaning we now had to mirror any an all `nim` packages so that Developers could install them with `nimble`[^2].

[^2]: The `nim` package manager: [github.com/nim-lang/nimble](https://github.com/nim-lang/nimble)


# Getting the Packages

Unlike a lot of languages `nim` doesn't have a centralised package repository we can just `rsync` and serve.
Instead it fetches the packages source code from a URL defined in a [packages.json](https://github.com/nim-lang/packages/blob/master/packages.json) file, and compiles it locally.
That JSON file contains around 2000 entries so I am not going to download all of them manually.

I'm going to turn to `sh` to save me from that repetitive task, but first I need all the package URLs.

Thankfully `jq` already exists.

``` bash
jq '.[].url' packages.json > urls
sed -i 's/"//g' urls # to get rid of the quotes
```

Now all I need is someway to clone them all, preferably in parallel.

``` bash
#!/bin/sh

while read u ; do
  GIT_TERMINAL_PROMPT=0 git clone "${u}" &
done < "${1}

wait
```

That script will take a list of URLs from a file (the script's arg 1) and clone them all with git in the background[^3].
I set `GIT_TERMINAL_PROMPT=0` so that git will fail silently instead of prompting me for a username/password.

[^3]: well at least all the ones served by git

# Serving the Packages

Now I could sit back and just put all these repos on a network share,
telling the devs to grab the files they need and install locally,
but that sounds like a recipe for constant maintenance.
I want to make the process of installing packages as smooth as possible.
That means serving the packages in such a way that `nimble` knows how to grab,
and telling `nimble` where to find them.

## Apache Configuration

Since the existing infrastructure was set up with apache[^4] I went with it.

[^4]: and because git has [examples](https://git-scm.com/docs/git-http-backend) on how to set this up

The configuration snippet to serve the packages is:

``` apache
SetEnv GIT_PROJECT_ROOT /var/www/git
SetEnv GIT_HTTP_EXPORT_ALL
ScriptAlias /git/ /usr/lib/git-core/git-http-backend/

SetEnvIf Git-Protocol ".*" GIT_PROTOCOL=$0

<LocationMatch "^/git/.git-receive-pack$>
    AuthType Basic
    AuthName "Git Access"
</LocationMatch>
```

> Note: The documentation for `git-http-backend` seems to be written for RH distros.
> Debian's man pages for `git-http-backend` have the wrong paths.
> Both documents say to set the `ScriptAlias` to `/usr/libexec/git-core/git-http-backend/`
> instead of the correct `/usr/lib/git-core/git-http-backend/` path.
> Make sure to find the correct path for the bin, and don't forget the trailing `/` like I did

Now that we are serving git we have to let `nimble` know about them.
Remember that `packages.json` file we got all those URLs from, `nimble` uses that to figure out how to get the packages.
So we just have to place a copy with all our packages in `~/.nimble/packages_official.json`.

First step is to replace all the URLs with our new domain.

``` bash
sed -i'' 's|"url": "https://.*/|"url": "https://<new.domain.name>/' packages.json
```

## Self Service

I don't want to [have](have) to keep answering questions about how to get this working.
So I made a webpage to explain how[^5].

[^5]: I am by no means a webdev so it is very simple

``` html
<!DOCTYPE html>
<html>
<head>
    <title>Nim Package Mirror</title>
<style>
    ul{
        column-count: auto;
        column-width: 15em;
    }
</style>
</head>
<body>
    <h1>Our Nim Package Mirror</h1>
    <ol>
        <li>Install nimble (Ex <code>apt install nim</code></li>
        <li>Place this <a href="./packages_official.json">file</a> in your .nimble dir.</li>
        <li>Install a package as normal (Ex <code>nimble install about</code></li>
    </ol>
    <ul>
        <!--#include file="./packages.html" -->
    </ul>
</html>
```

{{< image src="/img/nim-web.png" alt="Instruction on how to use the nim package server and a list of the available packages" position="center" style="border-radius: 8px;" >}}

That `<!--#include file="./packages.html" -->` line is so I can display a list of all the packages available.
You just need tell apache to allow [includes](https://httpd.apache.org/docs/2.4/mod/mod_include.html)[^6]

[^6]: props to [css-tricks](https://css-tricks.com/the-simplest-ways-to-handle-html-includes/) for the how to.

``` apache
<directory "/var/www/html/nim/">
    options includes
    addtype text/html .html
    addoutputfilter includes .html
</directory>
```

And a simple script to populate the `packages.html` file:

``` bash
> /var/www/html/nim/packages.html # ensure the file exists and is empty
for d in /var/nim/git/* ; do
  printf "<li>%s<li>\n" "${d}" >> /var/www/html/nim/packages.html
done
```

> I will probably generate the list, sort it and replace the file if they differ.

The last step is to tell the devs about it and see if they break anything.
