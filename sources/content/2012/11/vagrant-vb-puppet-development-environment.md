Title: Vagrant, VirtualBox, Puppet, yet another boring explanation
Tags: Vagrant, VirtualBox, Puppet, Development
Date: 2012-11-10 20:00
Category: Misc

How many times in your career you heard from other programmers in your team: *"For me it works"* while developing some functionality. It happen to me really often. Why does it happen? 99% of the cases it's the difference between environments on which app was run. By myself, I install huge amount of libs on my system not relevant to current project and I forget about them really often having a mess in my personal system. I think, lots of people does the same.

Coming back to the previously stated problem, how to solve it? One of the possible approaches might be using **Continuous Integration** for that, by creating for e.g. unstable, un-reviewed branch and setupping CI to make builds based on it. Even if this will work then still it's not optimal solution, because it forces you to push unstable code to repository. Additionally you have to schedule new build and wait for the results.

Much better approach would be having during the development a way to ensure that everybody have the same environment and as close as it's possible similar to production one. Answer for all and even more might be [**Vagrant!**](http://vagrantup.com "Vagrant home page").

**Vagrant** uses **VirtualBox** to build for you a virtual machine ready for work. The main advantage of it is it's simplicity to run it. As manual says to start a VM it's enough

    :::bash
    vagrant box add lucid32 http://files.vagrantup.com/lucid32.box
    vagrant init lucid32
    vagrant up

This will download you a box (it's a base image of a system) and add it to system globally, initialize it (it create for you file named **Vagrantfile**) in current directory and run VirtualBox. This VM already sees the folder in which you ran these commands, so all the files in your project are already accessible. What does it give you? If you ran a server inside a VM it will be able to read all the necessary files out of the box! 
But this was just easiest example, this server does not do too much other that just running. Imagine now that we need running server with PHP support, how to get it?

Again one of the approaches would be connecting to this VM and installing all necessary libs (using my beloved Debian's apt-get). It's not the best solution for many reasons:

 * People might install different libraries and forget what did they install.
 * To have the same VM you would have to add whole image to your version control system. These images are big, imagine that with each lib install you have to push it. This is Madness!
 * How to manage conflicts when two developers install some other libraries at the same time and there's a conflict on this file?

When you use **Vagrant** you do not install this way any necessary libraries. Vagrant uses provisioning system for that. You can choose between [**Chef**](http://www.opscode.com/chef/) and [**Puppet**](http://puppetlabs.com). You define in special file what kind of libraries and actions have to be made on a server to setup it according to your needs. I think it'll be easier while trying to explain it using an example.

##A long Puppet example

Let's assume that we have web project in that's written PHP. For that we will need a web server and a PHP installation. For that let's choose *Nginx* as a server and *php-fpm*. All puppet files will be in project root (**not** document root) in *private/puppet* folder. All files for Puppet have to be put in correct place in directory structure which should look like that:
    
    manifests
        #here goes all top directory manifests for e.g. production setup file (file should end with .pp extension here)
    modules
        #here go all modules which can represent for e.g. the way how apache should be initialized
        module
            manifests
                init.pp #additional module manifets, it's important that main module file is named init.pp
            files
                #any files used by module
            templates
                #any templates used by module, I will use a template for virtual host definition

Ok, so let's prepare firstly production environment. For that I will create such a directory structure:
    
    manifests
        production.pp
        development.pp
    modules
        php-fpm
        php-devel
        nginx
        users

And an initialization manifest

    :::puppet
    # private/puppet/manifests/production.pp
    class production-init ($page) {
        exec { 'apt-get update': 
            command => '/usr/bin/apt-get update',
        }
        #create a webadmin user
        users::add_user {'webadmin':
            uid => 500
        }
        include php-fpm
        class {'nginx':
            page => $page,
        }    
    }
Let's explain what's happening here.

    class production-init ($page)

This is Puppet class definition. I've called production-init, so as the name said it's responsible for whole production initialization. This class get's one $page parameter which is the page domain name. This is not normally necessary, but I wanted to be able to easily create other pages without making copy&paste of Puppet manifests. Let's go further:

    exec { 'apt-get update': 
        command => '/usr/bin/apt-get update',
    }

Exec task just executes command on server (our VM). This will just update repositories before starting any installations.  Important thing in exec is that commands have to have **full path to executable**!

In most of the cases while running webserver on production environment you run it in different user (webadmin), so also let's create one not to have later some permission problems on a different machine, when on ours it'll be working ok (for e.g. with cache folder).

    #create a webadmin user
    users::add_user {'webadmin':
        uid => 500
    }

Here we called **defined resource type**. If you have good eyesight you saw that previously class was used (production-init). Defined types and classes are similar, but they have one difference. Classes are singletons and any call will Puppet to raise an error.

'users::add_user' means: call add_user from users module. This implies that there is an add_user.pp file in a path:

    modules/users/manifests/

Last parts are calls to *php-fpm* and *nginx* classes. You can see the difference how they're called. Why like that? It's because *php-fpm* is a class without any parameters, so it's allowed to use shorter **include** version.

    include php-fpm # === class {'php-fpm': }

You have to remember that there is **no possibility** to call a class with a param(s) with include!

##Resource add_user

Definition of *add_user* looks as follows:

    :::puppet
    #private/puppet/modules/users/manifests/user_add.pp
    define users::add_user ( $username = $title, $uid ) {

        user { $username:
                comment => "Automatically generated by puppet",
                shell   => "/bin/bash",
                uid     => $uid,
                managehome => true
        }

        group { $username:
                gid     => $uid,
                require => User[$username]
        }
        
        file { "/home/${username}/":
            ensure  => directory,
            owner   => $username,
            group   => $username,
            mode    => 750,
            require => [ User[$username], Group[$username] ]
        }

        file { "/home/${username}/.ssh":
                ensure  => directory,
                owner   => $username,
                group   => $username,
                mode    => 700,
                require => File["/home/${username}/"]
        }

        # now make sure that the ssh key authorized files is around
        file { "/home/${username}/.ssh/authorized_keys":
                ensure  => present,
                owner   => $username,
                group   => $username,
                mode    => 600,
                require => File["/home/${username}/"]
        }
    }

This code creates an user and a group with the same name. Additionally checks if home this user has home folder, *.ssh* fodler, and *authorized_keys* file.

There are still few things that have to be explained.

    define users::add_user ( $username = $title, $uid ) {
This one defines add_user 'routine', which can be invoked many times (for e.g. you don't want to create two instances of *apache* so it will be a *class* not a resource type). This resource has two params

 * $username that defaults to $title ($title is a special variable in *Puppet*, it's always present and it's a first 'param', for e.g. *user { 'webadmin': }*, here *'webadmin'* is a title of resource user).
 * $uid which will be created user uid.

In *Puppet* there is a possibility to define requirements for each resources. The simplest example will be group creation

    :::puppet
    group { $username:
        gid     => $uid,
        require => User[$username]
    }

Here group with $username name is created. But to create this grup it's required that previously user with the same name was also created. Thanks to it, if for some reason user was not created, then the creation of group will not be made at all, because of not satisfied dependencies.

Other important part is that Puppet is able to **evaluate variables in a string**, this is done by putting variable in brackets with dollar sign in front (to be honest it is enough to put $var, but according to Puppet standards brackets should be also there):

    require => File["/home/${username}/"]


I will skip rest of the code in add_user, it should be self explanatory.

##php-fpm installation

Code for *php-fpm* looks as follows:

    :::puppet
    class php-fpm {
        package { 'php5-fpm':
            ensure => present,
            require => Exec['apt-get update'],
        }

        package { "php5-mysql":
            ensure => present,
            require => Package['php5-fpm'],
            notify => Service['php5-fpm'],
        }

        package { "php5-curl":
            ensure => present,
            require => Package['php5-fpm'],
            notify => Service['php5-fpm'],
        }

        package { "php5-xcache":
            ensure => present,
            require => Package['php5-fpm'],
            notify => Service['php5-fpm'],
        }

        service { 'php5-fpm':
            ensure => running,
            require => Package['php5-fpm'],
            notify => Service['php5-fpm'],
        }
    }
Let's say what's happening here.

    class php-fpm {
This is a *php-fpm* class definition without any params. This class just install some of the PHP5 packages with **PHP5 FPM** itself.

    package { 'php5-fpm':
        ensure => present,
        require => Exec['apt-get update'],
    }
This installs *php5-fpm*, only interesting part here it's that we want to ensure that we're installing the newest possible package in repositories. For that it's required that command **apt-get update** finished successfully. 

    :::puppet
    package { "php5-xcache":
        ensure => present,
        require => Package['php5-fpm'],
        notify => Service['php5-fpm'],
    }

I will describe one more package as an example. The code above install **php5 xcache* extension. What's interesting here it's the notify part. It tells Puppet that, he has to notify *php5 fpm* service that new package was installed and it has to load it (do not mistake it with creating necessary ini files - it doesn't do that, it only informs that it should make reload/restart). Puppet knows how to notify most common services out of the box (for e.g. if it needs to be restarted or reload will be enough).

##The last step - Nginx
Nginx installation will be defined in three separate files. Let's start with first one - virtual host definition

    :::puppet
    #private/puppet/modules/nginx/manifests/virtual-host.pp
    class nginx::virtual-host ($page, $template = 'default') {
        file { "/home/virtual/${page}":
            ensure => "directory",
            owner  => "webadmin",
            group  => "webadmin",
            mode   => 755,
            require => User['webadmin']
        }

        file { "/home/virtual/${page}/public":
            ensure  => directory,
            mode    => '0755',
            owner   => 'webadmin',
            group   => 'webadmin',
            require => File["/home/virtual/${page}"]
        }

        file { 'nginx-site-available':
            path => "/etc/nginx/sites-available/${page}",
            ensure => file,
            require => Package['nginx'],
            group => root,
            owner => root,
            content => template("nginx/${template}"),
        }

        file { '/etc/nginx/sites-enabled/${page}':
            target => "/etc/nginx/sites-available/${page}",
            ensure => link,
            notify => Service['nginx'],
            require => [
                File['nginx-site-available'],
                File['default-nginx-disable'],
                Package['nginx'],
            ],
        }
    }

As always let's start from the beginning to say a few words about this code.

    class nginx::virtual-host ($page, $template = 'default') {...}

This one defines class virtual host. This class has two params where on has a default value. In our case *$page* param is a domain for which virtual host is created. The *$template* variable defines which template to use for virtual host definition. How to call this class? It's done by such a call:
    
    :::puppet
    class {'nginx::virtual-host':
        page => 'www.example.com'
    }

Here I've shown how you can call this class with a param with default value, because I omitted the $template param.

I didn't say earlier

You can ask, why this way, why nginx::virtual-host is a class not a defined resource type.
This question is good. For you define might be much better. I've used here class because during development of a project I use only one VM per domain. If you plan having multiple virtual hosts on one VM, then make it define!

##Templates

The only other interesting part in previous definition is part with virtual host file.

    :::puppet
    file { 'nginx-site-available':
        path => "/etc/nginx/sites-available/${page}",
        ensure => file,
        require => Package['nginx'],
        group => root,
        owner => root,
        content => template("nginx/${template}"), # <- load a template file in a path: nginx/templates/${template}
    }

This is the first time I've used here a template. Here I've put a virtual host definition in such a file and it looks as follows

    server {
        listen 80;
        server_name <%= @page %>; # <- put here value of $page variable
        root /home/virtual/<%= @page %>/public;
        index index.php;

        location / {
            try_files $uri $uri/ /index.php?$args;
        }

        location ~ \.php$ {
            fastcgi_pass 127.0.0.1:9000;
            fastcgi_index index.php;
            include fastcgi_params;
        }
    }
As you can see it's really simple definition, where **Nginx** will route all the addresses into index.php file. The important part here is that, we declare variables in templates slightly different. You put them between:

    <%= @variable %>
The question that you can ask right now, is how we pass a value to a template? It's simple. **All the variables in parent scope are available in a template!**. What does it mean in the end? If we have a variable named $variable in a class/define, then this variable has to have exactly the same name. To access this variable you would put in template <%= @variable %>

##Nginx itself (phew, it was long..)

Let's go to Nginx installation itself. I've done it by this Puppet manifest:

    :::puppet
    #private/puppet/modules/nginx/manifests/init.pp
    class nginx ($page, $template = 'default') {

        package { 'nginx': 
            ensure => present,
            require => Exec['apt-get update'],
        }

        service { 'nginx':
            ensure => running,
            require => Package['nginx'],
        }

        class { 'nginx::virtual-host':
            page => $page,
            template => $template,
            require => Package['nginx']
        }

        file { 'default-nginx-disable':
            path => "/etc/nginx/sites-enabled/default",
            ensure => absent,
            require => Package['nginx'],
            notify => Service['nginx']
        }
    }
This code:

 * installs **nginx** package
 * ensures that nginx is up and running
 * adds one virtual host
 * at the end it ensures also that there is no default virtual host present

##What's next?
Ok, so we defined all necessary manifests and templates, but we need to still define an entry point. At the beginning I've defined a **production-init** class. To be honest all the contents of this class can be removed from it and just called one by one. I did it this way, because I have one more file **development.pp** which calls this class. It's done this way because it allows me to use this manifest on production, but also I can make small alterations to environment for a dev (for example for a dev you might install additionally xdebug, change error_reporting to E_STRICT & E_ALL etc.)

So *development.pp* file might be looking like:

    :::puppet
    #private/puppet/manifests/development.pp
    import "production-init"
    class {'production-init': page => 'example.com'}

    #install development packages, set additional development settings
    ...

###Setup Vagrantfile
Now really last thing - we have to show Vagrant where our manifest file is and which one to use. When you'll open this *Vagrantfile* in root of your project you should add:
    
    :::vagrant
    config.vm.provision :puppet, :module_path => "private/puppet/modules", :options => "--verbose --debug" do |puppet|
        puppet.manifests_path = "private/puppet/manifests"
        puppet.manifest_file  = "development.pp"
    end
    config.vm.share_folder "app-root", "/home/virtual/example.com", "."

Now you should be able to run
    
    vagrant reload

This should restart VM and run Puppet on it.

##Closing comments
This example shows how to run a **Nginx** server with PHP using minimal configuration. Probably it might be not enough for you. You might need for example a database on VM (which I don't prefer to have there, I like more to have a real DB server accessible by everybody).

After all work is done and development environment prepared, it can be easily shared between all devs using for example Git and adding all files.

There's one important thing before you'll start developing on this VM. I **strongly suggest enabling NFS!** Shared folder build in VM is really slow. When I said really slow I meant **it's hellish slow**.

To enable NFS firstly install it on host machine by doing
    
    vagrant ssh

    sudo apt-get install nfs-common

On local machine it might be also required to install **nfs-server** package.
At the end just modify in Vagrantfile definition of shared folder by adding :nfs => true at the end, so it'll be looking like

    config.vm.share_folder "app-root", "/home/virtual/example.com", ".", :nfs => true

We did a lot of work, but was it worth? I will say yes. Here are some **pros** using Vagrant:

 * Unified development environment which is identical or really close to production environment,
 * No more problems with with "For me it works",
 * A documented configuration of environment thanks to Puppet manifests, which allows all to see which libs are necessary to be present on production env,
 * No more hacks to have few versions of libs/interpreters etc. (look at **Ruby** programs - every of them require different version of *Ruby*, some gems and other, so you need to have rbenv installed to manage it).
 
 Still there is no ideal tool and **Vagrant** is not ideal either. As a **cons** you can count:

 * It likes to hang, without any reason. The good part is that it happens only when booting up the VM. Sometimes sadly it requires then to do cleaning up by *vagrant destroy* and then *vagrant up*. This thing leads to loss of a lot of time.
 * It takes a lot of time to setup environment. 

What's easier?

    :::puppet
    sudo apt-get install nginx
or
    
    :::puppet
    class nginx {ensure => present, require => Exec['apt-get update']}
    service { 'nginx':
        ensure => running,
        require => Package['nginx'],
    }
For me the first option.

 * Without a NFS the VM is really slow, so don't try to use it for development.

Still I think that the Vagrant will stay in my computer as a really useful tool for long time.

Thanks!
