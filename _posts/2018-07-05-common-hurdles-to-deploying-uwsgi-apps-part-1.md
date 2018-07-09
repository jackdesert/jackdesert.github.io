Common Hurdles to Deploying uWSGI Apps (Part 1: Invocation)
===========================================================

So you wrote a shiny new WSGI application and you want to
put it out there in the world where it can do some good. Ideally, you want
it to autostart (systemd) so if you server reboots, your app comes up
automatically.

This is article 1 of a 4-part-series. Here are the links to the other articles: [1. Invocation][1], [2. Nginx][2], [3. Emperor][3], and [4. Systemd][4].

This article details the hurdles that
you must cross to deploy your WSGI Python app using uWSGI and Nginx.

These be the hurdles:

1. uWSGI Invocation with Proper Entry Point and Callable
2. Make Python Packages Available to Masquerading User
3. Getting Nginx Write Access to the Unix Socket

We will also discuss common pitfalls for each hurdle and how to get past them.

The code used for this tutorial is available at https://github.com/jackdesert/simple-uwsgi-nginx-tutorial.

---

<a name='install'></a>Installation (The Easy Part)
--------------------------------------------------

This tutorial was run on an t2.micro EC2 instance running Ubuntu 16.04.
If you are using something that is not an ubuntu derivative, you will
need to change the `apt install` to something appropriate for your OS.


Log in to the box where you want to run this tutorial.

    ssh ubuntu@some-amazon-instance

Install uwsgi and the python(3) plugin we will need.

    sudo apt install -y uwsgi-core uwsgi-plugin-python3

Install nginx and pip

    sudo apt install -y nginx python3-pip

Now you have an executable called uwsgi

    which uwsgi # /usr/bin/uwsgi

Let's fetch the tutorial source code.

    cd
    git clone github.com/jackdesert:jackdesert/simple-uwsgi-nginx-tutorial


---

<a name='hurdle-1'></a>Hurdle 1: uWSGI Invocation with Proper Entry Point and Callable
---------------------------------------------------------------------------------------
![Image](https://thumbs.dreamstime.com/z/hurdle-barrier-28185719.jpg)

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

    sudo uwsgi --chmod-socket=020 --enable-threads --plugin=python3 -s /run/uwsgi/simple.sock --manage-script-name --mount /=wsgi:application --uid ubuntu --gid www-data


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
  * --gid www-data     # masquerade as the www-data group


If you are lucky and this works on the first try, you will get this:

    mounting wsgi:application on /
    WSGI app 0 (mountpoint='/') ready in 1 seconds on interpreter 0x16e7e40 pid: 28127 (default app)
    *** uWSGI is running in multiple interpreter mode ***
    spawned uWSGI worker 1 (and the only) (pid: 28127, cores: 1)

If you get an error:

    ImportError: No module named 'flask'

Then skip ahead to Hurdle 2.



<a name='pitfall-1a'></a>
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


<a name='pitfall-1b'></a>
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

There are three pieces that must align. In our case they are as follows:

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







<a name='hurdle-2'></a>
Hurdle #2: Make Python Packages Available to Masquerading User
--------------------------------------------------------------
![Image](https://thumbs.dreamstime.com/z/hurdle-barrier-28185719.jpg)

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


<a name='pitfall-2a'></a>
### Pitfall 2a: Python Package is Installed, but Unavailable to (Regular) User X

If you're here, it means that the standard way of installing packages
via pip did not make those packages available to your masquerading user.


Remember these options from the invocation:

  * --uid ubuntu       # masquerade as the ubuntu   user
  * --gid www-data     # masquerade as the www-data group


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


<a name='pitfall-2b'></a>
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

Next Steps
----------

The [next article][2] will be how to add Nginx to this mix.


[1]: {% link _posts/2018-07-05-common-hurdles-to-deploying-uwsgi-apps-part-1.md %}
[2]: {% link _posts/2018-07-05-common-hurdles-to-deploying-uwsgi-apps-part-2.md %}
[3]: {% link _posts/2018-07-06-common-hurdles-to-deploying-uwsgi-apps-part-3.md %}
[4]: {% link _posts/2018-07-06-common-hurdles-to-deploying-uwsgi-apps-part-4.md %}
