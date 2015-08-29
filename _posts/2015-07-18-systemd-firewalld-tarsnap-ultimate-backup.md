---
layout: post
title: "Backups without Worries: Tarsnap, Systemd and Firewalld"
comments: true
excerpt_separator: <!--more-->

---

For years, I've been looking for the perfect backup solution. I thought I had
found it for some time in Dropbox, but I recently abandoned that due to
concerns around privacy. I looked around, and I believe that I have found
something (much) better now. And the great thing? It was actually pretty easy,
by combining some recent developments in Linux desktop land.

<!--more-->

What are my requirements? Well, there's quite a few:

* I am using Fedora as my primary workstation and so I need it work work there.
* I want privacy for my backups. This means client-side encryption.
* I want backups to be non-tamperable. So they should be signed.
* I want the backup client to be verifiable. So I need source code.
* I want my backups to be hassle free. They should run in the background,
  automatically and reliably.
* The backup process should be efficient. This means incremental backups
  and deduplication of data.
* The backups should be timely. I can live with the loss of an hour of work,
  but not more.
* Finally, I travel a lot. It would be nice if a backup would not run over an
  in-flight wireless network or a super slow conference wifi.

I thought I'd never find my perfect solution, but I believe I now have. Here's
what I have been using for the last few months:

As the backup tool, I use [Tarsnap](http://www.tarsnap.com). Tarsnap gives me
most of what I need. It brands itself as *backups for the truly paranoid*. I
don't believe I fit the label, but I appreciate the positioning and I'll gladly
use the tool anyway. Tarsnap is written by Colin Percival who is well known and
well respected in security circles. It comes with source code that you compile
yourself (note that Tarsnap is not [Open Source](http://opensource.org/) as per
the [OSI Definition](http://opensource.org/definition), but that is not one of
my requirements). It comes with extensive crypto including client-side
encryption and signatures. It also performs extensive deduplication.

Tarsnap is just half of the solution though. As anybody that has ever written
their own backup script knows, there's a whole bunch of hairy issues regarding
how to execute the backup job periodically. The standard answer would be to use
[cron(8)](http://man7.org/linux/man-pages/man8/cron.8.html). However, this
solution has some serious non-trivial issues:

* You need to take care of job exclusion yourself. Jobs can take a long time
  and so a future job could be started before the current job finishes. This
  requires some kind of reliable locking.
* Debugging is a pain. Cron will mail the output of your script. Local mail is
  not something that I want to deal with anymore in the twenty-teens.
* Cron has no notion of a logged on user. Cron jobs always run, even when not
  logged in. This is not what you want for a backup job.
* Modern laptops sleep all the time, and may only be active for short periods
  of time between sleeps. Cron has no notion of this. Jobs could be delayed for
  a long amount of time if the system sleeps at the time the job is due.

Enter `systemd --user`! My Fedora 22 system runs a [user mode systemd](
https://wiki.archlinux.org/index.php/Systemd/User) for every logged on user by
default. The process is fully reliable. Using control groups and other tricks,
systemd makes sure there is always exactly one systemd user instance running
per logged on user. Systemd also takes care of the following:

* If a previous job has not completed, a new one will not be started up
  concurrently.
* It is possible to run a job at regular intervals after it was first
  activated, ignoring time that the computer went to sleep between jobs.
* It provides great logging and status monitoring. It can show you the output
  of your job, when it was last run, and when it will next run.

The last problem is the travel issue. Ideally I'd like my backups to run
automatically when I'm at home and when I'm at work, but not elsewhere.

Enter `firewalld`! The command `firewall-cmd --get-active-zones` will give you
the curently active networking zones. Using this it was easy to write a script
that will only kick off tarsnap if I am at home or in the office.

That's it! The total number of lines of configuration and scripts that I needed
is a whopping 36. I believe that's safely below the "will I be able to maintain
this?" threshold.

Files are below. First, the systemd timer file. Store this in
`~/.config/systemd/user/tarsnap.timer`.

{% highlight ini linenos %}
[Unit]
Description=Run tarsnap hourly

[Timer]
OnCalendar=*-*-* *:15:00

[Install]
WantedBy=default.target
{% endhighlight %}

The timer kicks off a service file with the same name. Store the file below in
`~/.config/systemd/user/tarsnap.service`. Note how this uses `/bin/sh` to
execute the backup script. This way it can use the `$HOME` environment
variable:

{% highlight ini linenos %}
[Unit]
Description=Run tarsnap backup

[Service]
Type=oneshot
ExecStart=/bin/sh ${HOME}/bin/run-tarsnap
{% endhighlight %}

Finally the backup script. Store it in `~/bin/run-tarsnap`. The script is
pretty straightforward. As an optimization, it uses a timestamp in $HOME to
keep the time of the last backup. If no files in the backup locations have a
date more recent than the timestamp, and a backup has run in the last 24 hours,
then no backup is performed. This is done to prevent creating too many
identical tarsnap archives.

{% highlight sh linenos %}
zone="`firewall-cmd --get-active-zones | head -1`"
echo "Current network zone: $zone"

test "$zone" != "home" -a "$zone" != "work"  && {
    echo "Not running backup in this zone"
    exit 1
}

echo "Running backup in this zone"

cd "$HOME"
files="Documents Projects"  # configure this

test -f ~/.backup-timestamp &&
  test "`find ~/.backup-timestamp -mmin -1440 | wc -l`" -eq 1 && \
    test "`find $files -newer ~/.backup-timestamp | wc -l`" -eq 0  && \
      { echo "Last backup < 24 hours and no changes, exiting."; sleep 1; exit 0; }

label="`date +'%Y%m%d-%H%M'`"
tarsnap -c -f "$USER-$label" $files
touch ~/.backup-timestamp
sleep 1  # Without this, stdout sometimes does not get into journal.
{% endhighlight %}

Final thoughts
--------------

The solution has been working great for me so far over the last couple of
months. So, do I store all my files in Tarsnap? No, it is too expensive for
that. At $0.25 per GB per month it is much more expensive than e.g. S3 which is
just $0.03 per GB per month. The way I backup up my files is as follows:

* All my "work" (documents, code, presentations, etc) is backed up by Tarsnap.
* Media files like family pictures, music and videos go onto Google Drive.
  I don't care much about privacy here. On Google Drive I can store these for
  the impressive price of $0.01 per GB per month (for the 1TB plan). And it
  also lets me share and synchronize these files with my family. I'm using the
  great [Insync](https://www.insynchq.com) client for this.

The final final thought: back up your keys! Without your tarsnap private key
you cannot recover. I have a `~/.pki` directory that holds all my private keys
including the one for tarsnap. This directory is occasionally copied to
multiple USB storage devices that are encrypted using dm-crypt/LUKS with a
6-word [Diceware](http://world.std.com/~reinhold/diceware.html) passphrase.
