SSH Forwarding with Vagrant
===========================

If you do most of your development work within a VM, it is convenient to have ssh forwarding
enabled. Three things need to happen in order to use ssh forwarding with Vagrant.

1. Edit your Vagrantfile
------------------------

Uncomment this line from your Vagrantfile

    config.ssh.forward_agent = true

And run `vagrant reload` so the Vagrantfile gets processed again


2. Authenticate on the Host
---------------------------

From your host operating system:

    ssh-add ~/.ssh/id_rsa


3. Add Your Public Key to the Authorized Keys File on the Guest
---------------------------------------------------------------

From your guest operating system:

    cat /path/to/a/copy/of/public/key/id_rsa.pub >> ~/.ssh/authorized_keys


4.  Use the '-- -A' Flag When You ssh Into Vagrant
------------------------------------------------

Instead of just running `vagrant ssh`, you need to take the key with you. 

    vagrant ssh -- -A

The double dashes is a separator that makes all arguments after it be passed directly to the system ssh call.


Troubleshooting
---------------

Vagrant is pretty loose about letting you into your VM. Because of this, it doesn't even ask for the passphrase to your key. That's why step 2 above was required. Without step 2, Vagrant will let you in, but whenever you try to user your key you will get a publickey error.
