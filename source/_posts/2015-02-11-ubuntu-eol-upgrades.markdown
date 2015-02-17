---
layout: post
title: "How to upgrade when stuck with unsupported Ubuntu Releases"
date: 2015-02-16 16:02:50 +0100
author: ifosch
comments: true
categories: 
- Infra

---

Have you ever updated your servers?
I trust most of you have.
Anyway, from time to time one finds an unsupported release, running on some forgotten server.
In some cases, it's an unused service or something to be decommissioned, but sometimes, we should update it.
At [Devex](https://www.devex.com), found this situation recently.
You'll find in this article how we upgraded some cases we couldn't decommission.

<!-- more -->

## Common aspects

Our infrastructure is running on AWS EC2, so we take a lot of profit from the ability to create AMI images from running instances.
With this approach, we could use those images to try out the upgrade without affecting the instances.
Of course, this implies that data might be outdated, which can be fixed easily by reloading a backup from the correct instance, or by using the same procedure on the running instance later.

Also, the usage of ephemeral drive mounted on `/mnt` is a good place to store backups and files we would need to save during the process.

## Upgrading from Intrepid

Our first case was upgrade an Intrepid Ubuntu server, running Postgres 8.3, to Lucid.
We chose to stay in Lucid because this application is going to be decommissioned soon, anyway.
However, this decommission won't happen immediately, since these features must be implemented in newer applications.
So, the first step was getting a backup from the PostgreSQL database, running these commands:

    sudo mkdir -p /mnt/postgresql
    sudo chown -R postgres:postgres /mnt/postgresql
    sudo -u postgres pg_dumpall > /mnt/postgresql/backup

Once the backup was saved, we upgraded intrepid packages.
Of course the original Intrepid package repositories are down already, but we could take use of the `old-release` repositories.
To select them, we used the following commands:

    sudo cat >/etc/apt/sources.list <<EOL
    deb http://old-releases.ubuntu.com/ubuntu/ intrepid main restricted universe multiverse
    deb http://old-releases.ubuntu.com/ubuntu/ intrepid-updates main restricted universe multiverse
    deb http://old-releases.ubuntu.com/ubuntu/ intrepid-security main restricted universe multiverse
    deb http://old-releases.ubuntu.com/ubuntu/ intrepid-backports main restricted universe multiverse
    deb http://old-releases.ubuntu.com/ubuntu/ intrepid-proposed main restricted universe multiverse
    EOL
    sudo apt-get update
    sudo apt-get upgrade

With these commands, we got an updated Intrepid release, so our following step was upgrade to Jaunty, by issuing the following commands:

    sudo perl -p -i.intrepid -e 's/intrepid/jaunty/' /etc/apt/sources.list
    sudo apt-get update
    sudo apt-get install update-manager-core
    sudo apt-get upgrade
    sudo reboot

After the reboot, we got a Jaunty, so next step is upgrading to Karmic, and Lucid:

    sudo perl -p -i.jaunty -e 's/jaunty/karmic/' /etc/apt/sources.list
    sudo apt-get update
    sudo apt-get upgrade
    sudo do-release-upgrade -f DistUpgradeViewNonInteractive
    sudo reboot

Karmic was the first release on which we could use `do-release-upgrade`.
In that case, we used the `-f DistUpgradeViewNonInteractive` option to have the release upgrade done without user intervention.
After that reboot, the OpenSSH server upgrade updated the host fingerprints, so we needed to change our `.ssh/known_hosts` removing the old one, before reconnecting.
Then, the final step was resetting the PostgreSQL cluster, issuing the following commands as the `postgres` user:

    pg_dropcluster --stop 8.4 main
    pg_createcluster -u postgres -d /data/postgresql/8.4/main 8.4 main
    service postgresql-8.4 start
    psql -f /data/postgresql/backup postgres

Finally, after checking everything is in place, we just could remove the old cluster and setup:

    sudo rm -rf /data/postgresql/8.3 /etc/postgresql/8.3

## Upgrading from Maverick

In that case we could upgrade to Precise, since Trusty just broke the boot process.
The first step was pretty similar, since Maverick package repositories are also outdated:

    sudo bash -c "cat >/etc/apt/sources.list <<EOF
    deb http://old-releases.ubuntu.com/ubuntu/ maverick main restricted universe multiverse
    deb http://old-releases.ubuntu.com/ubuntu/ maverick-updates main restricted universe multiverse
    deb http://old-releases.ubuntu.com/ubuntu/ maverick-security main restricted universe multiverse
    deb http://old-releases.ubuntu.com/ubuntu/ maverick-backports main restricted universe multiverse
    deb http://old-releases.ubuntu.com/ubuntu/ maverick-proposed main restricted universe multiverse
    EOF
    "
    sudo apt-get update
    sudo apt-get upgrade
    sudo do-release-upgrade -f DistUpgradeViewNonInteractive
    sudo reboot

After the reboot, we got a Natty release running, so updating to Oneiric was the next step. Pretty much the same:

    sudo do-release-upgrade -f DistUpgradeViewNonInteractive
    sudo reboot

In our case, the application running here was a little bit sensible to automatic removal of packages for Precise upgrade.
So in the following step, we couldn't use the unattended mode of `do-release-upgrade`:

    sudo do-release-upgrade

When manually running the `do-release-upgrade`, we just ommitted the old package removal part.
After the reboot, issued from the `do-release-upgrade`, we got a working Precise release running.

## Details

In first place, we'd like to point the great importance of having backups and images for the instances.
With the backups, preferably dumps, you'll be able to recover the data for your databases.
Of course, if the application running on the affected instances are not in the same server, then you'll need to face the client libraries updating and possible version differences that might appear.
Here's where having the server images might help.
We took static images for having the servers upgrade procedures ready.
