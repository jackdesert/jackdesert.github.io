Common Hurdles to Deploying uWSGI Apps (Part 4: Systemd)
========================================================

This is article 1 of a 4-part-series. Here are the links to the other articles: [1. Invocation][1], [2. Nginx][2], [3. Emperor][3], and [4. Systemd][4].

Now that you have been able to start the uWSGI Emperor and have it start
your app using .ini files, let's take the next logical
step, which is to get the Emperor to autostart via Systemd.


<a name='systemd-unit'></a>
Systemd Unit File
-----------------

Let's add a symlink to our emperor's systemd unit file:

    cd /etc/systemd/system
    sudo ln -s ~/simple/config/emperor.uwsgi.service


Here is the unit file:

    # /etc/systemd/system/emperor.uwsgi.service
    [Unit]
    Description=uWSGI Emperor
    After=syslog.target

    [Service]
    ExecStart=/usr/bin/uwsgi --ini /etc/uwsgi/emperor.ini
    # Requires systemd version 211 or newer
    RuntimeDirectory=uwsgi
    Restart=always
    KillSignal=SIGQUIT
    Type=notify
    StandardError=syslog
    NotifyAccess=all

    [Install]
    WantedBy=multi-user.target


Note that the ExecStart option is exactly the same as the command line
invocation of uWSGI Emperoror in the last article.


<a name='start-emperor'></a>
Start The uWSGI Emperor
-----------------------

Now we are going to start the emperor via systemd. First let's start
tailing the system logs so we can see the status.

    sudo journalctl -f

Also make sure you have stopped the command-line-invoked emperor from the last article.

Then in another window, start the emperor with systemctl

    sudo systemctl start emperor.uwsgi.service

These are the system logs for a successful start.

    Jul 06 14:45:49 ip-172-31-14-5 sudo[31457]:   ubuntu : TTY=pts/4 ; PWD=/etc/systemd/system ; USER=root ; COMMAND=/bin/systemctl start emperor.uwsgi
    Jul 06 14:45:49 ip-172-31-14-5 sudo[31457]: pam_unix(sudo:session): session opened for user root by ubuntu(uid=0)
    Jul 06 14:45:49 ip-172-31-14-5 systemd[1]: Starting uWSGI Emperor...
    Jul 06 14:45:49 ip-172-31-14-5 uwsgi[31460]: [uWSGI] getting INI configuration from /etc/uwsgi/emperor.ini
    Jul 06 14:45:49 ip-172-31-14-5 systemd[1]: Started uWSGI Emperor.
    Jul 06 14:45:49 ip-172-31-14-5 sudo[31457]: pam_unix(sudo:session): session closed for user root


<a name='verify-nginx'></a>
Verify that Nginx Connects
--------------------------

As in the second and third articles, we want to make sure that we can curl localhost
and get a valid response back from Nginx and your app.

    curl localhost

If you get an error when curling localhost, use some of the tools you learned in articles 2 and 3. That is, verify the owner, group, and permissions of the socket file.

<a name='debuggin'></a>
Additional Debugging
--------------------

At this point we expect the following things to be working. In parentheses are the responsible config files foe each item.

1. uWSGI Emperor is running (as root) via Systemd (emperor.ini, emperor.uwsgi.service)
2. Your app is running as a vassal under the uWSGI Emperor (simple.ini)
3. Your app is masquerading as the user "ubuntu" and the group "www-data" (emperor.ini)
4. The socket file is owned by the user "ubuntu" and the group "www-data" (emperor.ini)
4. The socket file has permissions 020 which allows group writes (simple.ini)
5. Nginx is running as the user "www-data" and the group "www-data" (this is the default)
6. Nginx knows where the socket file is located (simple-nginx.conf)
7. Nginx has write access to the socket because nginx runs as group "www-data"
8. When you curl localhost, You get the appropriate response back from Nginx + your app

If these are not all working for you, revisit the section that introduced them. It is also useful to tail the various log files:

    # Tailing the emperor log and the vassal log
    cd /var/log/uwsgi
    tail -f *


    # tailing the nginx access and error logs
    cd /var/log/nginx
    tail -f access.log error.log


    # tailing the system logs
    sudo journalctl -f




<a name='autostart'></a>
Autostart on Reboot
-------------------

Assuming System started for you, go ahead and enable the service so it
will start automatically whenever your system boots up.

    sudo systemctl enable emperor.uwsgi.service



---

Next Steps
----------

You've now complete all four articles in the uWSGI - Nginx - Emperor - Systemd train. Congratulations!

[Click here][1] to start back at the beginning.


[1]: {% link _posts/2018-07-05-common-hurdles-to-deploying-uwsgi-apps-part-1.md %}
[2]: {% link _posts/2018-07-05-common-hurdles-to-deploying-uwsgi-apps-part-2.md %}
[3]: {% link _posts/2018-07-06-common-hurdles-to-deploying-uwsgi-apps-part-3.md %}
[4]: {% link _posts/2018-07-06-common-hurdles-to-deploying-uwsgi-apps-part-4.md %}

