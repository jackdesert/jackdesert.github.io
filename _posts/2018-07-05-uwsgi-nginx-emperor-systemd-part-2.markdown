Common Hurdles when Deploying a Python App with uWSGI and Nginx
===============================================================


Hurdle #3: Getting Nginx to Read/Write to the Unix Socket
---------------------------------------------------------
![Image](https://thumbs.dreamstime.com/z/hurdle-barrier-28185719.jpg)

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

Next Steps
----------

The next article shows how to add Emperor to this mix.

