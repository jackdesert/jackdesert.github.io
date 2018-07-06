Bare Bones Hiera 
================

Here's the least you can do to get Hiera running.


Installation
------------

On Ubuntu 14.04 the `sudo puppet resource package hiera ensure=installed` command didn't work, so I
installed the gem. 

    $ sudo gem install hiera


Configuration
-------------

Hiera needs to know where to look for your data, and what format to expect it in.

Create an empty file at /etc/hiera.yaml (note the 'a' in yaml). Finding an empty 
configuration file, Hiera will assume these defaults:

    # Hiera Defaults
    :backends: yaml
    :yaml:
      :datadir: /var/lib/hiera
    :hierarchy: common
    :logger: console

Once Hiera has a configuration file (even our empty one), you can ask 
it for data, and it will (successfully) return nil

    $ hiera trusty
    #=> nil


Add Some Data
-------------

Hiera is going to look in <datadir>/<hierarchy>.<backend> for your data. 
Based on the Hiera defaults listed above, this is 
/var/lib/hiera/common.yaml. (Note the 'a' in yaml). So let's put some data in that file.

    # /var/lib/hiera/common.yaml
    trusty: 'steed'
    family: 'heirloom'

(If you want this file in a more convenient location, just create a symlink from /var/lib/hiera to a 
folder of your choosing.)


Getting Data Out
----------------

Now from the command line:

    $ hiera trusty
    #=> steed
    $ hiera family 
    #=> heirloom 
