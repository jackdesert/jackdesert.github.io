Common Hurdles to Deploying uWSGI Apps (Part 3: Emperor)
========================================================

This is article 1 of a 4-part-series. Here are the links to the other articles: [1. Invocation][1], [2. Nginx][2], [3. Emperor][3], and [4. Systemd][4].


Now that uWSGI is running your application, let's take the next logical
step, which is to get it to autostart via Emperor and Systemd.

<a name='install'></a>
Installation
------------

If you already have the uWSGI executable, there is nothing more to install.

    which uwsgi

The config files we will use for this article are in the same
git repo as the first article, https: https://github.com/jackdesert/simple-uwsgi-nginx-tutorial



<a name='step-1'></a>
First Step: Run uWSGI Emperor from Command Line
-----------------------------------------------

<a name='emperor-ini'></a>
### Emperor .ini file

https://stackoverflow.com/questions/41038159/cant-run-uwsgi-ini-file-with-systemd-emperor

First create a couple directories we will need

    sudo mkdir /etc/uwsgi
    sudo mkdir /var/log/uwsgi
    sudo chown ubuntu:www-data /var/log/uwsgi

Then let's add a symbolic link to the emperor.ini file from the git repo

    cd /etc/uwsgi
    sudo ln -s ~/simple/config/emperor.ini



Here are the contents of emperor.ini:

    # /etc/uwsgi/emperor.ini
    [uwsgi]
    emperor = /etc/uwsgi/vassals
    uid = ubuntu
    gid = www-data
    limit-as = 1024
    logto = /var/log/uwsgi/emperor.log


<a name='invoke-emperor'></a>
### Invoke uWSGI Emperor

To invoke the emperor from command line:

    sudo uwsgi --ini /etc/uwsgi/emperor.ini

Note that just as in the previous article, we invoke uwsgi with `sudo`, because
otherwise our uid and gid settings in emperor.ini will be ignored.

This doesn't give us much output, but we can cat or tail the log to see more.
Note the location of "logto" in emperor.ini.

    cat /var/log/uwsgi/emperor.log


    *** Starting uWSGI 2.0.12-debian (64bit) on [Fri Jul  6 12:58:42 2018] ***
    compiled with version: 5.4.0 20160609 on 31 August 2017 21:02:04
    os: Linux-4.4.0-1049-aws #58-Ubuntu SMP Fri Jan 12 23:17:09 UTC 2018
    nodename: ip-172-31-14-5
    machine: x86_64
    clock source: unix
    pcre jit disabled
    detected number of CPU cores: 1
    current working directory: /home/ubuntu
    detected binary path: /usr/bin/uwsgi-core
    *** WARNING: you are running uWSGI without its master process manager ***
    your processes number limit is 3900
    limiting address space of processes...
    your process address space limit is 1073741824 bytes (1024 MB)
    your memory page size is 4096 bytes
    detected max file descriptor number: 1024
    *** starting uWSGI Emperor ***



<a name='step-2'></a>
Second Step: Add Your WSGI App as a Vassal
------------------------------------------

> Vassal: (in the feudal system) a person granted the use of land, in return for rendering homage, fealty, and usually military service or its equivalent to a lord or other superior; feudal tenant. -- dictionary.com




Let's make a symbolic link from the vassals directory to our simple.ini file.

    sudo mkdir /etc/uwsgi/vassals
    cd /etc/uwsgi/vassals
    sudo ln -s ~/simple/config/simple.ini


Here are the contents of simple.ini. Notice that many of the options
are the same ones that we used when calling uWSGI from command line in the
last episode. Also note that "%n" will be converted to the string "simple".

    [uwsgi]
    chdir = /home/ubuntu/simple
    wsgi-file = wsgi.py
    callable = application
    processes = 4
    threads = 2
    offload-threads = 2
    stats =  127.0.0.1:9191
    max-requests = 5000
    master = True
    vacuum = True
    enable-threads = true
    harakiri = 60
    logto = /var/log/uwsgi/%n.log
    chmod-socket = 020
    plugin = python3
    runtime_dir = /run/uwsgi
    pidfile=%(runtime_dir)/%n.pid
    socket = %(runtime_dir)/%n.sock



Emperor will start the vassal as soon as its .ini file is added to
/etc/uwsgi/vassals. We can see the results of this in two files.
First we'll look in the emperor log:

    cat /var/log/uwsgi/emperor.log

    [uWSGI] getting INI configuration from simple.ini
    Fri Jul  6 13:22:44 2018 - [emperor] vassal simple.ini has been spawned
    Fri Jul  6 13:22:44 2018 - [emperor] vassal simple.ini is ready to accept requests

And we will also look in the log for this particular vassal:

    cat /var/log/uwsgi/simple.log

    *** Starting uWSGI 2.0.12-debian (64bit) on [Fri Jul  6 13:22:44 2018] ***
    compiled with version: 5.4.0 20160609 on 31 August 2017 21:02:04
    os: Linux-4.4.0-1049-aws #58-Ubuntu SMP Fri Jan 12 23:17:09 UTC 2018
    nodename: ip-172-31-14-5
    machine: x86_64
    clock source: unix
    pcre jit disabled
    detected number of CPU cores: 1
    current working directory: /etc/uwsgi/vassals
    writing pidfile to /run/uwsgi/simple.pid
    detected binary path: /usr/bin/uwsgi-core
    chdir() to /home/ubuntu/simple
    your processes number limit is 3900
    your process address space limit is 1073741824 bytes (1024 MB)
    your memory page size is 4096 bytes
     *** WARNING: you have enabled harakiri without post buffering. Slow upload could be rejected on post-unbuffered webservers ***
    detected max file descriptor number: 1024
    lock engine: pthread robust mutexes
    thunder lock: disabled (you can enable it with --thunder-lock)
    uwsgi socket 0 bound to UNIX address /run/uwsgi/simple.sock fd 3
    Python version: 3.5.2 (default, Nov 23 2017, 16:37:01)  [GCC 5.4.0 20160609]
    Python main interpreter initialized at 0x1c5e1e0
    python threads support enabled
    your server socket listen backlog is limited to 100 connections
    your mercy for graceful operations on workers is 60 seconds
    mapped 415360 bytes (405 KB) for 8 cores
    *** Operational MODE: preforking+threaded ***
    WSGI app 0 (mountpoint='') ready in 0 seconds on interpreter 0x1c5e1e0 pid: 30789 (default app)
    *** uWSGI is running in multiple interpreter mode ***
    spawned uWSGI master process (pid: 30789)
    spawned uWSGI worker 1 (pid: 30791, cores: 2)
    spawned uWSGI worker 2 (pid: 30792, cores: 2)
    spawned uWSGI worker 3 (pid: 30793, cores: 2)
    spawned uWSGI worker 4 (pid: 30794, cores: 2)
    *** Stats server enabled on 127.0.0.1:9191 fd: 16 ***
    spawned 2 offload threads for uWSGI worker 4
    spawned 2 offload threads for uWSGI worker 3
    spawned 2 offload threads for uWSGI worker 2
    spawned 2 offload threads for uWSGI worker 1




<a name='step-3'></a>
Third Step: Verify that Nginx is communicating with your Vassal
---------------------------------------------------------------
And to verify that it's actually serving requests, let's curl localhost:

    curl localhost

If you get

<a name='hurdle-1'></a>
### Hurdle #1: Allow Nginx to Access the Socket

Just as in the previous article, you may run into this issue again.
The key is to verify the ownership and permissions on the socket file.

    ls -al /run/uwsgi/simple.sock
    # s----w---- 1 ubuntu www-data 0 Jul  6 13:48 simple.sock

We expect the owner to be ubuntu, group to be www-data.
If instead you see that both owner and group are ubuntu (your normal user),
verify that you are calling uwsgi with sudo, lest your uid and gid
settings be ignored.


We expect the group "write" bit to be set, since in simple.ini specified chmod-socket 020.


<a name='chmod-socket'></a>
### Changing chmod-socket

Touching the vassal .ini file is enough to cause the vassal to reload.
However, it appears that if you want to test different settings for
chmod-socket within your vassal .ini file, you need to:

1. Stop the vassal
2. Change the chmod-socket setting in the .ini file
3. Remove the socket file manually
4. Start the vassal

If emperor was still running when you made the symlink to simple.ini,

As soon as you made the symlink to simple.ini



<a name='debugging'></a>
### Additional Debugging

There are other issues you may run into. Luckily, the error messages
provided in the emperor log file and the vassal log file
are helpful in figuring out what is needed.

    cd /var/log/uwsgi
    tail -f emperor.ini simple.ini

Sometimes it is also helpful to truncate one or both of these log files
so you can see the entire log from a single startup.

    cd /var/log/uwsgi
    echo '' > emperor.log
    echo '' > simple.log
    # Then start the Emperor
    # See entire emperor.log
    cat emperor.log
    # See entire simp
    cat simple.log


https://stackoverflow.com/questions/41038159/cant-run-uwsgi-ini-file-with-systemd-emperor#answer-42393135
  * Run ExecStart command as non-root
  * If you previously ran ExecStart command as root, you will need to
    remove or chown the logfile, etc to make it writable by non-root again


To make sure emperor is running and not running into errors:

    tail -f  /tmp/uwsgi.log

To make sure your app is running and not running into errors:

    tail -f  /var/log/simple.log

Also useful is to stop the emperor, truncate simple.log, start the emperor.
That way you can see an entire fresh run.

If things go well, you will see this in the uwsgi.log:

  [uWSGI] getting INI configuration from simple.ini
  Thu Jul  5 13:47:00 2018 - [emperor] vassal simple.ini has been spawned
  Thu Jul  5 13:47:00 2018 - [emperor] vassal simple.ini is ready to accept requests


---

Next Steps
----------


The [next article][4] shows how to add Systemd to the mix.


[1]: {% link _posts/2018-07-05-common-hurdles-to-deploying-uwsgi-apps-part-1.md %}
[2]: {% link _posts/2018-07-05-common-hurdles-to-deploying-uwsgi-apps-part-2.md %}
[3]: {% link _posts/2018-07-06-common-hurdles-to-deploying-uwsgi-apps-part-3.md %}
[4]: {% link _posts/2018-07-06-common-hurdles-to-deploying-uwsgi-apps-part-4.md %}

