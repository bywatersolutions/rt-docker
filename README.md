# Request Tracker for Kubernetes

Docker image of [Request Tracker](http://bestpractical.com/rt/) setup to be
run under Kubernetes, though it should work fine under "just docker", too.

It includes RT::Extension::REST2, RT::Authen::Token and RT::Extension::MergeUsers.

A prebuild version is available on [quay.io/abh/rt](https://quay.io/abh/rt/).

    docker pull quay.io/abh/rt:latest

## How to configure RT

Make a volume mounted into `/opt/rt/etc/RT_SiteConfig.d/` with one or more
[configuration files](https://docs.bestpractical.com/rt/4.4.2/RT_Config.html)
(file extension `.pm`).

## How to run

To test under docker, something like

    mkdir RT_SiteConfig.d
    vi RT_SiteConfig.d/myrt.pm
    docker run -i --rm \
        -v `pwd`/RT_SiteConfig.d:/opt/rt/etc/RT_SiteConfig.d/ \
        -p 8000:8000 \
        quay.io/abh/rt:latest

Then access the RT installation on port 8000.

For development you might want to leave out `--rm`. I like including
it to not have a bunch of "dead" containers around and to discourage
editing anything in a non-reproducible way.

## How to configure incoming mail

You need your mailhost to run
[rt-mailgate](https://docs.bestpractical.com/rt/4.4.2/rt-mailgate.html)
to submit mail into the system, or a variation that does the same.

I use another kubernetes pod running
[sparkpost-rt](https://github.com/abh/sparkpost-rt) to process mail
handled by [SparkPost](https://www.sparkpost.com).

If your mail goes into something like gmail it should be reasonably
straightforward to setup a container with a pop or imap client to do
this, too.

## How to configure outgoing mail

The docker container sends mail via "localhost port 25". In kubernetes
you can run a second container in the pod listening on port 25. I use
[namshi/smtp](https://hub.docker.com/r/namshi/smtp/) for this,
relaying to a "real" mailhost. It's easy to configure it to use gmail
or Amazon SES. I again use SparkPost as a "generic SMTP relay", any
SMTP service should work.

RT really wants to use a program to send mail. The busybox `sendmail`
regularly (but not always?) quit with some error or didn't like the
parameters RT provided. mini-sendmail requires the `-f` parameter and
the sender address to not have a space between them.

Instead I patched RT to allow sending mail via SMTP. Use 'smtp' as
the configured MailCommand.

## How to make a custom build

You can make a custom build based on this with your own Dockerfile,
along the lines of:


```
FROM quay.io/abh/rt:latest

# install a module from CPAN
RUN cpanm RT::Extension::Demo

# Install a module from the current directory might work, too. Adding a .tar.gz would for sure. 
ADD Local-Module /tmp/
RUN cpanm /tmp/Local-Module/

# Add some file
ADD something.html /opt/rt/share/html/something.html

