---
layout: post
title: "systemd as a CI system"
date: 2021-09-09 21:00:00 +0100
tags: tech coding
---

I make fairly extensive use of systemd for scripting tasks which I need to run repeatedly and possibly on schedule (or based on other triggers), and recording a log of the output. At work and among friends, people use Jenkins for the same purpose. This gave me a stupid idea - can I turn systemd into Jenkins, and use it to build Git codebases?

<!--more-->

*Disclaimer: This is purely a bit of "what if?" fun, not serious advice*

The basic idea is quite simple

1. Write a timer to pull a git repo
2. If a new change is found, trigger a oneshot service to build the project

Writing this for each project would get quite repetitive, so I'd like to generify it as follows

1. The timer will be parameterised with the git URL, so I can do `systemctl enable systemd-ci-poll@$REPO`
2. The builder service will expect an executable at `$REPO/.systemd-ci/build`

# Assumptions

Since this is just a proof of concept, I make the followingg assumptions

1. Every project has a name marching `[a-Z0-9]+`, to saves character escaping headaches
2. Every project is already cloned in the workspace dir, and we're only building the branch that's been checked out
3. I'm not including any sort of artefact archiving or workspace cleaning. That could be implemented if needed, but for this PoC I'll take the *laissez-faire* approach of leaving that to the build script
4. We can always fast-forward. No force pushes!

# System setup

I deployed a fresh Ubuntu 20.04 LTS Linux container on Proxmox and made the following changes

* Installed `git`
* Add a `systemd-ci` user and group with the normal minimal permissions expected of a service user
* Create the workspace dir `/var/lib/systemd-ci` with owner `systemd-ci:systemd-ci` and mode `770`


# The builder service

This service is quite simple - just run an executable

`/etc/systemd/system/systemd-ci-build@.service`

{% highlight systemd %}
[Unit]
Description=Build a systemd-ci project
After=multi-user.target

[Service]
Type=oneshot
WorkingDirectory=/var/lib/systemd-ci/%i
ExecStart=/var/lib/systemd-ci/%i/.systemd-ci/build
User=systemd-ci
{% endhighlight %}

We can test this in isolation, by just creating

`/var/lib/systemd-ci/test/.systemd-ci/build` (mode: `+x`)

{% highlight bash %}
#!/bin/bash -eux
touch success
echo it works
{% endhighlight %}

Then execute `systemctl start systemd-ci-build@test` (remembering to do a `daemon-reload` first). Logs can be found with `journalctl -u systemd-ci-build@test`

# The trigger service

So for the timer, we need to repeatedly pull the repository, and take note of when it changes. In accordance with assumption 4, let's be really hacky and just `grep Fast-forward` in the git output

`/etc/systemd/system/systemd-ci-poll@.service`

{% highlight systemd %}
[Unit]
Description=Poll a systemd-ci project repo and trigger a build if necessary
After=multi-user.target
OnSuccess=systemd-ci-build@%i

[Service]
Type=oneshot
WorkingDirectory=/var/lib/systemd-ci/%i
ExecStart=/bin/bash -c "git pull | grep Fast-forward"
User=systemd-ci
{% endhighlight %}

And here we run into a problem. `OnSuccess` was added in systemd 249, but this version of Ubuntu has 245. Luckily, bash has a negation operator, so I can use `OnFailure` and `ExecStart=/bin/bash -c "! git...`

# The polling timer

`/etc/systemd/system/systemd-ci-poll@.timer`

{% highlight systemd %}
[Unit]
Description=Run a systemd-ci poll every 15m

[Timer]
OnUnitInactiveSec=15m

[Install]
WantedBy=timers.target
{% endhighlight %}

As far as I understand you need to enable `systemd-ci-poll@.timer` at least once, in addition to `systemd-ci-poll@$REPO.timer`. Well, I guess that provides an easy way to turn the whole thing off if needed

# Project creation script

`systemd-ci-init` (mode `+x`)

{% highlight bash %}
#!/bin/bash -eux

# Syntax: sudo systemd-ci-init <repo url> <project name>

cd /var/lib/systemd-ci
git clone $1 $2
chown -R systemd-ci:systemd-ci $2
systemctl enable --now systemd-ci-poll@.timer
systemctl enable --now systemd-ci-poll@$2.timer
{% endhighlight %}

# Conclusion

Yes, you can do it, no, you probably shouldn't. The main problem is that you pollute the logs with "failed" polls, but I imagine this can be fixed if one is determined enough, but not on version 245