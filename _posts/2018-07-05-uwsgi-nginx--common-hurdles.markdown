Common Hurdles when Deploying a Python app with uWSGI and Nginx
===============================================================

So you wrote a shiny new WSGI application and you want to
put it out there in the world where it can do some good. Ideally, you want
it to autostart (systemd) so if you server reboots, your app comes up
automatically.

This multi-part tutorial will go over in detail the pieces that are
required in order to get all the pieces working end to end. Specifically,
those hurdles are:

1. uWSGI Invocation with Proper Entry Point and Callable
2. Make Python Packages Available to Masquerading User
3. Getting Nginx Write Access to the Unix Socket

We will also discuss common pitfalls for each hurdle and how to get past them.

The code used for this tutorial is available at https://github.com/jackdesert/simple-uwsgi-example.

Installation (The Easy Part)
----------------------------

This tutorial was run on an EC2 instance running Ubuntu 16.04.

Install uwsgi and the python(3) plugin we will need.

    sudo apt install -y uwsgi-core uwsgi-plugin-python3

Install nginx and pip

    sudo apt install -y nginx python3-pip

Now you have an executable called uwsgi

    which uwsgi # /usr/bin/uwsgi

Let's fetch the tutorial source code.

    cd
    git clone github.com/jackdesert/simple-uwsgi-example simple


---
---

Hurdle 1: uWSGI Invocation with Proper Entry Point and Callable
----------------------------------------------------------------

Now the fun begins. Take a look at simple/wsgi.py. This is our wsgi entry point.

    from myflask import application

    if __name__ == '__main__':
        application.run()


Notice the wsgi entry point (above) imports from myflask. Let's see what's in myflask.py. This is our flask application:

    from flask import Flask

    application = Flask(__name__)

    @application.route("/")
    def hello():
        return "<h1>Hello There!</h1>"

    if __name__ == "__main__":
        application.run(host='0.0.0.0')


At this point you could run either of these files directly:

    python3 wsgy.py

or
    python3 myflask.py

either of which will fire up flask with its built-in, non-production-grade webserver.


But instead, let's invoke the entry point from uWSGI. But first, let's create
the directory where we want the socket file to live.

    sudo mkdir /run/uwsgi
    sudo chown ubuntu:ubuntu /run/uwsgi

    sudo uwsgi --chmod-socket=020 --enable-threads --plugin=python3 -s /run/uwsgi/simple.sock --manage-script-name --mount /=wsgi:application --uid ubuntu --gid ubuntu


Let's go over the options in the invocation

  * sudo uwsgi           # Invoke as sudo so we can set --uid and --gid
  * --chmod-socket=020   # change the socket to chmod 020 after starting
  * --enable-threads
  * --plugin=python3     # use the python3 plugin
  * -s /path/to/my.sock  # Sets the unix socket. Must match nginx config
  * --manage-script-name
  * --mount /=wsgi:application   # "wsgi" means wsgi.py is the name of the ENTRY POINT
                                 # :application must match the name of the CALLABLE in ENTRY POINT
  * --uid ubuntu       # masquerade as the ubuntu user
  * --gid ubuntu       # masquerade as the ubuntu group


If you are lucky and this works on the first try, you will get this:

    mounting wsgi:application on /
    WSGI app 0 (mountpoint='/') ready in 1 seconds on interpreter 0x16e7e40 pid: 28127 (default app)
    *** uWSGI is running in multiple interpreter mode ***
    spawned uWSGI worker 1 (and the only) (pid: 28127, cores: 1)

If you get an error:

    ImportError: No module named 'flask'

Then skip ahead to Hurdle 2.

### Pitfall #1a: Entry Point Improperly Referenced

Remember this part of the invocation?
    --mount /=wsgi:application

There are multiple things going on in this invocation.
The first is the "wsgi" part. That is the entry point it will look for.
Meaning it expects a file named wsgi.py.

Of course, you can name your entry point (file) whatever you want,
as long as you reference it correctly in the invocation.

To simulate the first type of error, let's rename wsgi.py to wsgi_original.py
and see what happens.

    *** Operational MODE: single process ***
    mounting wsgi2:application on /
    ImportError: No module named 'wsgi'
    unable to load app 0 (mountpoint='/') (callable not found or import error)
    *** no app loaded. going in full dynamic mode ***


### Pitfall #1b: Callable Not Found


Remember this part of the invocation?
    --mount /=wsgi:application

The second piece is the "application" part. That is the name of the callable.
Remember this line from wsgi.py:

    # wsgi.py
    from myflask import application
    ...


and this line from myflask.py:

    # myflask.py
    ...
    application = Flask(__name__)
    ...

There are four pieces that must align. In our case they are as follows:

  1. the "application" part (after the colon) in the mount option
  2. "application" is imported in wsgi.py
  3. "application" is defined in myflask.py

Of course you can name these something else other than "application",
but they must all align.

To show what happens if I change the line from wsgi.py to the following:

    # wsgi.py
    from myflask import some_other_app

Then I get this error:

    *** Operational MODE: single process ***
    mounting wsgi:application on /
    Traceback (most recent call last):
      File "./wsgi.py", line 1, in <module>
        from myflask import some_other_app
    ImportError: cannot import name 'some_other_app'
    unable to load app 0 (mountpoint='/') (callable not found or import error)
    *** no app loaded. going in full dynamic mode ***

OR if I change the line from myflask.py to the following:

    # myflask.py
    ...
    application_53 = Flask(__name__)
    ...


then I get this error:

    *** Operational MODE: single process ***
    mounting wsgi:application on /
    Traceback (most recent call last):
      File "./wsgi.py", line 1, in <module>
        from myflask import application
      File "./myflask.py", line 5, in <module>
        @application.route("/")
    NameError: name 'application' is not defined
    unable to load app 0 (mountpoint='/') (callable not found or import error)
    *** no app loaded. going in full dynamic mode ***


OR if I rename myflask.py to myotherflask.py:

    *** Operational MODE: single process ***
    mounting wsgi:application on /
    Traceback (most recent call last):
      File "./wsgi.py", line 1, in <module>
        from myflask import application
    ImportError: No module named 'myflask'
    unable to load app 0 (mountpoint='/') (callable not found or import error)
    *** no app loaded. going in full dynamic mode ***





---
---







Hurdle #2: Make Python Packages Available to Masquerading User
--------------------------------------------------------------

You may encounter this error, which simply means that the 'flask'
package is unavailable:

    mounting wsgi:application on /
    Traceback (most recent call last):
      File "./wsgi.py", line 1, in <module>
        from myflask import application
      File "./myflask.py", line 1, in <module>
        from flask import Flask
    ImportError: No module named 'flask'
    unable to load app 0 (mountpoint='/') (callable not found or import error)
    *** no app loaded. going in full dynamic mode ***


No problem, right? Let's install flask via pip.

    python3 -m pip install --user flask

If you are lucky, that's all it will take to get past this hurdle.


### Pitfall 2a: Python Package is Installed, but Unavailable to (Regular) User X

Remember these options from the invocation:

  * --uid ubuntu       # masquerade as the ubuntu user
  * --gid ubuntu       # masquerade as the ubuntu group


This example uses the easy version. That is, uwsgi masquerades as ubuntu,
which is the the same user that normally logs in. This makes it painless
to install the flask package (or whatever packages your app requires)
and still have them available when the app is run via uwsgi.

However, let's say you want your app masquerade as a different user than
ubuntu.

The not-too-hard version of this is if the user want the app to masquerade
as is a regular user, meaning one who can log in. Just log in as that user,
then install your packages.

    sudo su <some_other_regular_user>
    python3 -m pip install --user flask


### Pitfall 2b: Python Package is Installed, but Unavailable to (NONRegular) User Y


Let's say that you want your app to masquerade as the www-data user.
The www-data user is a NONRegular user. That is, it is a real user, but
it has certain restrictions on it, like it cannot actually log in.
And it doesn't actually own its home directory.

Your best option for installing python packages may be to install them
using the system package manager, which installs the packages for all users.

In our case, that is:

    sudo apt install python3-flask


You may also be able to install packages using your normal way (Be that venv
or anaconda or whatever), and afterwards go in and grant the appropriate
group permissions such that the NONRegular user can access them.

Here's how you can test whether the user you want to masquerade as
has access to the python packages:

    sudo -u <some_other_user> python3
    >>> import flask

That is not the definitive answer, though, as some of my experiments
in this vein:

    # Note the "-H" flag
    sudo -H -u www-data python3 -m install flask

resulted in being able to import flask if running python as www-data thusly:

    # Note the "-H" flag
    sudo -H -u www-data python3
    >>> import flask

I may have had to disable pip cache along the way. But the flask package
was still unavailable when the app was run via uWSGI, masquerading as www-data.






---
---




Hurdle #3: Getting Nginx to Read/Write to the Unix Socket
---------------------------------------------------------

Now that you have crossed Hurdles 1 and 2, your app is being run by uWSGI
and the required modules are accessible to your app. You are ready to connect
your app to Nginx.

Let's tell Nginx to act as a reverse proxy for our app:

    cd /etc/nginx/sites-enabled
    sudo rm default
    sudo ln -s /home/ubuntu/simple/config/simple-nginx.conf
    sudo nginx -s reload

Here is the nginx config file:

    # simple-nginx.conf
    server {
        listen 80;
        server_name _;

        location / {
            include uwsgi_params;
            uwsgi_pass unix:/run/uwsgi/simple.sock;
        }
    }

Note that it contains the same path to the unix socket file that you used
when invoking uWSGI.


### Pitfall 3a: Nginx Cannot Find Socket File

To simulate this error, let's change the uwsgi_pass line in simple-nginx.conf
so that nginx looks in the wrong place for the socket:

    # simple-nginx.conf
    ...
    uwsgi_pass unix:/run/uwsgi/not-so-simple.sock;
    ...


And reload nginx

    sudo nginx -s reload

Nginx will not complain initially, as nginx is expected to fire up even if your
application is not yet running. But let's tail the nginx logs and
send a request to your app.

    cd /var/log/nginx
    tail -f access.log error.log

In another window, make sure your app is running using the uWSGI invocation
from Hurdle #1.

In a third window, make a request from your app:


    curl localhost

You will get a 502 Bad Gateway error:

    $ curl localhost
    <html>
    <head><title>502 Bad Gateway</title></head>
    <body bgcolor="white">
    <center><h1>502 Bad Gateway</h1></center>
    <hr><center>nginx/1.10.3 (Ubuntu)</center>
    </body>
    </html>


And in the window where you are tailing the nginx logs, you will see

    ==> error.log <==
    2018/07/05 17:26:44 [notice] 28601#28601: signal process started
    2018/07/05 17:26:48 [crit] 28602#28602: *298 connect() to unix:/run/uwsgi/not-so-simple.sock failed (2: No such file or directory) while connecting to upstream, client: 127.0.0.1, server: _, request: "GET / HTTP/1.1", upstream: "uwsgi://unix:/run/uwsgi/not-so-simple.sock:", host: "localhost"

    ==> access.log <==
    127.0.0.1 - - [05/Jul/2018:17:26:48 +0000] "GET / HTTP/1.1" 502 182 "-" "curl/7.47.0"


Notice that is is saying "No such file or directory" regarding the file "/run/uwsgi/not-so-simple.sock".

Once you get nginx and your uWSGI app pointed at the same socket file, this error
will go away. If you changed your nginx config file, you can revert it now

    cd simple
    git checkout config/simple-nginx.conf
    sudo nginx -s reload



### Pitfall 3b: Permission Denied when Nginx Connects to Socket

#### A Bit of Background on uWSGI and Nginx
There are two users of importance in this equation. One is the user that
runs nginx (www-data in my case). The other is the user your app masquerades as
via uWSGI. (ubuntu in my case).

uWSGI will be the one to create the socket file, and it will set certain
permissions on the file. Obviously since uWSGI created the file, it will
have no trouble accessing it.

By default, uWSGI creates the socket file with 755 permissions.
With those permissions, the user of nginx are not be able to access the file.

#### On to the Pitfall and its Solution
Once you've made sure both uWSGI and Nginx are pointed at the same socket file,
Nginx may still give you a Permission Denied error when it goes to access the socket.

To simulate this error, I'm going to remove the "chmod-socket=020" part of the uWSGI invocation, so my uWSGI invocation now looks like this:

    sudo uwsgi --enable-threads --plugin=python3 -s /run/uwsgi/simple.sock --manage-script-name --mount /=wsgi:application --uid ubuntu --gid www-data

This means the uWSGI starts, the socket file will have 755 permissions,
which means the group (www-data) will not be able to write to it.


Let's tail the nginx logs again:

    cd /var/log/nginx
    tail -f access.log error.log

Start uWSGI using the invocation above (without chmod-socket).

Here is the error we get:


    ==> error.log <==
    2018/07/05 17:30:59 [crit] 28610#28610: *302 connect() to unix:/run/uwsgi/simple.sock failed (13: Permission denied) while connecting to upstream, client: 127.0.0.1, server: _, request: "GET / HTTP/1.1", upstream: "uwsgi://unix:/run/uwsgi/simple.sock:", host: "localhost"

    ==> access.log <==
    127.0.0.1 - - [05/Jul/2018:17:30:59 +0000] "GET / HTTP/1.1" 502 182 "-" "curl/7.47.0"


Note that it is pointed to the correct socket file, but that is has a "Permission denied" error. uWSGI is running as ubuntu:www-data, which means when uWSGI creates the socket file, the socket file is owned by ubuntu:www-data. However, as mentioned above, by default
uWSGI sets the permissions on the socket file to 755, which means the group
does not have write permissions. That's why this fails.

This is the most challenging pitfall in getting uWSGI and Nginx to talk
to each other. Using the --chmod-socket option is the best way I've found to
make it work, and that's why it's used in the original invocation shown
at the top of this article.

It's interesting to note that using "--chmod-socket=020" is enough to make this work.
Offering it more permissions than this will also work, but in my case 020 was enough
so I left it at that. If 020 is not enough on your system, see if it works with
permissions set to 777, then gradually reduce permissions until you figure out
how much are needed. (Leaving it at 777 leaves it exposed to all users.)


the permissions of "020" ma
--chmod-socket=020
the only thingIn the working version from the beginning of this articleIf you are invoking uWSGI with a different --uid or a different --git



#### Other (Working) uWSGI / Nginx Configurations

First Working Alternate (no sudo, but open to all users):


Invoke uWSGI without sudo and without --uid / --gid, but with chmod-socket set to 666

    uwsgi --chmod-socket=666 --enable-threads --plugin=python3 -s ~/simple/tmp/simple.sock --manage-script-name --mount /=wsgi:application


Second Working Alternate (no sudo, but requires ACL. Not recommended, as it's more complex.):

This also works without sudo iff you use setfacl to set who owns the socket file.

The weird part is that you are setting up setfacl with USER permissions,
but it's the group permission that you are chmodding to.

    # Specify that the www-data user has full access to any new files
    # that are created in /run/uwsgi
    setfacl -Rm d:u:www-data:rwx,u:www-data:rwx /run/uwsgi
    #
    uwsgi --chmod-socket=020 --enable-threads --plugin=python3 -s ~/simple/tmp/simple.sock --manage-script-name --mount /=wsgi:application


#### Non-Working Nginx Configurations

Invoking uWSGI with --uid=ubuntu / --gid=ubuntu

This strategy worked sometimes, but not always. Using --gid=www-data makes more sense
anyway.


#### As A Last Resort

If you are still getting "Permission denied" error in the Nginx error.log, try manually changing the permissions on the socket file _after uWSGI is already running_.

    chmod 777 /run/uwsgi/simple.sock

This will at least give you a small win. It's not a permanent solution
because each time uWSGI starts up it will reset the permissions
on the socket.



---
---

Next Steps
----------

The next article will be how to add Emperor and Systemd to this mix. Stay tuned.

