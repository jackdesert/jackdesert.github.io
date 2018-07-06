Adding SystemD to the Mix
=========================

This is part 4 of a 4-part series. Click xxx to see the first article.

Now that uWSGI is running your application, let's take the next logical
step, which is to get it to autostart via Emperor and Systemd.




Systemd Unit File
-----------------

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



---

Next Steps
----------

You've now complete all four articles in the uWSGI - Nginx - Emperor - Systemd train. Congratulations!

