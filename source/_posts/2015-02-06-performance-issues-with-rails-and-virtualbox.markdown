---
layout: post
title: "Performance issues with Rails and VirtualBox"
date: 2015-02-06 12:02:50 +0100
author: ifosch
comments: true
categories: 
- Infra

---

Two weeks ago, we noticed some performance issues with Rails in our development setup, while all our other environments, some much less powerful, were working with much better performance. After checking there was no recent change expected to cause this, and running some diagnostics and measurements to record the performance in some point, this took us to a small trip into some Ruby on Rails debugging on a VirtualBox.

<!-- more -->

## Our development setup

Most of us at Devex, we use a specific setup of our apps into a [VirtualBox](https://www.virtualbox.org) machine so we can hold a local development version of our site and check how the new features integrate before to give the work as done. In some cases, for feature availability, performance, and workstation power reasons, these apps, and their components run completely on this environment, but in other cases, we don't run all the components in the virtual machine and we do use some components from the staging (we call it develop) environment, which is quite approximate to what we need. So this development environment's architecture and infrastructure is quite different from the ones we have in staging, preproduction, and, of course, production.
Not only that, in all our other environments, we run the apps using [Unicorn](http://unicorn.bogomips.org/) as a daemon, with more or less workers, while in this development environment, we run [WEBrick](http://www.ruby-doc.org/stdlib-1.9.3/libdoc/webrick/rdoc/WEBrick.html) within a screen session, just to simplify the load.

## The problem arises

The problem arised when we noticed Rails was underperforming when running in the local development setup. Some measurements were taken showing up that the problem seemed to be located in our front end application, since the backend was running properly:

    $ time wget -pq --no-cache --delete-after http://localhost:3002/apps/front_end/api/system/health
    real    0m0.523s
    user    0m0.000s
    sys 0m0.000s

    $ time wget -pq --no-cache --delete-after http://localhost:3004/public/system/health
    real    0m0.019s
    user    0m0.000s
    sys 0m0.000s

In that case, these two URIs correspond to the same content, but for our front end and for our back end applications, respectively, and this content is mostly text, JSON formatted. So, another measurement was done, which consisted into measuring the time to serve one complete page:

    $ time wget -pq --no-cache --delete-after http://localhost:3002/people
    real    0m13.393s
    user    0m0.008s
    sys 0m0.032s

Looking for similar cases on Internet, we found some links related to [WEBrick performance improvements](http://stackoverflow.com/questions/1156759/webrick-is-very-slow-to-respond-how-to-speed-it-up), [Rails performance within VirtualBox](http://stackoverflow.com/questions/8670080/rails-3-1-on-ubuntu-11-10-under-virtualbox-very-slow), and [WEBrick reverse lookups](http://www.visionfactory.com.au/blog/rails_dev_with_webrick_really_slow_in_a_).

None of the solutions we tried from these links helped to find the solution. We then tried doubling processors and memory available in the VM configuration, but it didn't solved the issue. Measuring with `atop` showed no bottleneck. Reproducibility of the issue in other computers and setups was also checked, so it was not something related to the specific setup or the hardware on which it was relying.

## What's going on then?

Then, we started analizing the issue in some more depth, first taking a look on what was going on when the browser issued a request. So, using the Network view in the web inspector, we saw very long waiting times for static content, but the rest of the times were ok, no execution issues or so.

Next step was taking a closer look to the log to see what would be happening when Rails received the static requests. We could confirm that the logs were written much more slowly when serving static files. So we decided to trace the WEBrick process.

When we `strace`'d the WEBrick process, we couldn't see anything meaningful, but then we learned WEBrick is threaded, so the actual requests where being attended by threads, which traces were not showed in the `strace` output.

The trick to [identify the thread TIDs](http://superuser.com/questions/80556/how-do-you-view-all-threads-running-on-linux) is to run the `ps` command with the `-T` option, which lists also the threads as processes, with the corresponding id. Then, you can [run `strace` on those TIDs](http://stackoverflow.com/questions/7698209/tracing-pthreads-in-linux). So, finally started finding errors on unavailable resources. We searched again and [found out](http://mitchellh.com/comparing-filesystem-performance-in-virtual-machines) that VirtualBox uses a specific filesystem for the shared folders, with some performance problems, and that NFS is one of the fastest ones you can use.

## The fix

Some of us, simply copied the shared folder content to be used to a regular directory in the VM disk. This fixes the issue, but introduces the need to keep copying the content when it gets updated.

Some others of us, decided to go for NFS, which has a couple of drawbacks:

* It is not compliant with having a Windows host, but this is not our case.
* It requires a LAN to be setup between the host and the guest, but we solved the whole problem by adding the following to our Vagrantfile:

      ip = ENV['VMSETUP_IP'] || `vboxmanage list hostonlyifs | grep IPAddress | cut -d: -f2 | tr -d ' '`.to_s.tr("\n", "") + '0'
      [...]
      Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
        [...]
        config.vm.network :private_network, ip: "#{ip}"
        [...]
        config.vm.synced_folder ".", "/vagrant", type: "nfs"
        [...]
      end
