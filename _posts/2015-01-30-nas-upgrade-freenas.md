---
layout: post
category: Hard
title: NAS Upgrade - Synology or Freenas?
---

In order to replace my old [Netgear Readynas DUO v1](http://support.netgear.com/product/RND2000v1+$28ReadyNAS+Duo+v1$29), I spent a lot of time to choose the right NAS for home purpose. Synology NAS are great products but expensives if you need more than two disks. Few month ago I made a great deal buying a [HP n54l gen7](http://www8.hp.com/au/en/products/proliant-servers/product-detail.html?oid=6280789) on amazon for less than ~$200. This is where the story began.

<!--more-->

## The beast

This little homeserver have 4 disk trails (said as non hotplug but if you flash it with an unlocked feature bios you can make them hotplug - aka esata) and two pci-express expansion slots.

<p align="center">
  <img src="/data/HP_Proliant_N54L_Inside.jpg" title="HP n54l" />
</p>

I added to this little boy 16GB of ECC RAM and 4x1TB drives and it is good to go !

The interresting point is that there is a usb port on the mother board to if you need to boot from USB, this will be very usefull for what we want to achieve later.

## Give me a NAS OS

### Synology port / hack / whatever

In the first stages, I tried to use the Synology OS (DSM) on it, I finaly manage to fake some controls calls in kernel and create my own DSM 4.3 booting system. This was fun and I've one of the first guy able to by pass their new protections. It was working quite good but updating is a pain as it could break my customs hooks. DSM is sounds pretty secure for an embeded OS as it checks if the hardware it's running on match the hardware it's suppose to run onto. If it fails, you can't use it as it will degrade your volume (basicaly delete /dev/sdx devices entries). The check is done by several parts in the system, maily on cgi scripts. Hacking in the cgi scripts is a pain as everything is offuscated and protected against disassembly/tracing by detecting debuggers:

Try to attach a strace on a process give you this:

```
20954 19:03:31 [f6bf4c08] ptrace(PTRACE_TRACEME, 0, 0, 0) = -1 EPERM (Operation not permitted)
20989 19:03:38 [f6c06c08] ptrace(PTRACE_TRACEME, 0, 0, 0) = -1 EPERM (Operation not permitted)
```

And nothing else interresting.

This is well known trick to defeat debugging/tracing. A reduction could be:

```c
#include <stdio.h>
#include <sys/ptrace.h>

int main()
{
    if (ptrace(PTRACE_TRACEME, 0, 0, 0) == -1)
    {
        printf("don't trace me !!\n");
        return 1;
    }
    // normal execution
    return 0;
}
```

Compile this using `gcc -o ptrace ptrace.c` then try to strace it:

```
strace -e ptrace ./ptrace

ptrace(PTRACE_TRACEME, 0, 0, 0)       = -1 EPERM (Operation not permitted)
don't trace me !!
+++ exited with 1 +++
```

The trick here is to override using `LD_PRELOAD` the ptrace function:

```c
#include <stdio.h>

long ptrace(int x, int y, int z)
{
  printf("--this is the ptrace clone block--\n"); return 0;
}
```

Compile it as a library `gcc -shared -o faketrace.so -fPIC faketrace.c`, then

```
strace -e ptrace  -E LD_PRELOAD=/root/faketrace2.so ./ptrace

--this is the ptrace clone block--
+++ exited with 0 +++
```

Using this method I found that these files are signed using Synology certificats and each file (process) is checking others, but the most interresting part is that it's also checking `/proc/bus/pci/devices` and compare the output to a list defined in the binary itself.

The trick was to customise the kernel to respond fake read entries when a particular process tried to read certains files related to devices installed. Basically it involves kernel hooking and seq files (kernel [proc.c](https://github.com/torvalds/linux/blob/master/drivers/pci/proc.c#L421) to dispatch certains call from processes to a hand crafted [seq_file](https://www.kernel.org/doc/Documentation/filesystems/seq_file.txt)).

To find process accessing this `/proc/devices` I had to create a kooking module printing every process name trying to access this file.

I also had to disassemble and edit a proprietary module and change some others things in the kernel tree. I will not provide the all the details and the methodoloy here but it worked. Synology provide the kernel source code (with lot of ifdef/ndef stripped away to male it almost impossible to compile).

DSM is quite a great system, the UI is really fancy and they are plenty of features and addons you can install. But I was not really confortable with running a hacked system in home production and beside that, running other applications directly next to the NAS services may not be a good idea and hardware support for a non genuine sylogoy is a pain (examplem I can't use usb port as it leads to kernel panic).

And at this time, I had to pack everything, because we were moving to California... so after almost one year, it's time to respin everything!

### Freenas

Based on FreeBSD, [Freenas](http://www.freenas.org) looks quite good for a DIY NAS. The UI is quite simple but ugly when you have tasted Synology DSM, but it's a NAS and you don't spend your life in the UI. It's pretty well documented and quite open to customisations.

#### ZFS

FreeNAS like freebsd uses [ZFS](https://www.freebsd.org/doc/handbook/zfs.html) as file system and seams quite robust and provides a lot of features like:

* Self healing architecture
* Snapshots / Clones
* Compression
* Deduplication (but really RAM-hungry)
* Encryption
* Thin provisioning
* Copy on write

It also uses a smart caching system (using RAM and even SSD):

<p align="center">
  <img src="http://solori.files.wordpress.com/2009/08/l2arc-in-zfs.png" title="l2arc" />
</p>

As I had a previous good experience with FreeBSD system using [pfSense](http://www.pfsense.org), this let me think that this solution will be more robust than using a hacked synology firmware...

#### Jails

The most interresting thing about FreeBSD is the concept of [Jails](https://www.freebsd.org/doc/handbook/jails.html#jails-synopsis), this is like chroot on steroids (like LXC for linux) and combined with ZFS, it sounds like docker. Its main goals are:

* **Virtualization**: Each jail is a virtual environment running on the host machine with its own files, processes, user and superuser accounts. From within a jailed process, the environment is (almost) indistinguishable from a real system.
* **Security**: Each jail is sealed from the others, thus providing an additional level of security.
* **Ease of delegation**: The limited scope of a jail allows system administrators to delegate several tasks which require superuser access without handing out complete control over the system.

With latest FreeBSD release, a network virtualization layer as been added [vImage](https://wiki.freebsd.org/VIMAGE), basically it let you have in your jails a **full instance** of the host’s networking stack, including loopback interface, routing tables...

In order give network to your jail, you will use [epair](https://www.freebsd.org/cgi/man.cgi?query=epair&sektion=4&manpath=FreeBSD+8.0-RELEASE) as a L2 back to back connected ethernet interfaces. One interface will be in your jail, the other one on your host (idealy member of a bridge if you plan to set several jails).


### Jails and plugins in Freenas

#### Plugins

In Freenas you have the choice between plugins and jails. Plugins is the easy way using [PBI](http://wiki.pcbsd.org/index.php/AppCafe®/9.2) - Push Button Installer in order to simplify the installation of plugins. Basically it will create a jails for you and install the prepackaged software in it. You can find actual supported plugins [here](http://doc.freenas.org/9.3/freenas_plugins.html#available-plugins).

Plugins are great if you don't want to deal with your jails by hand but the caveat is that you will have one jail per plugin and you can't really hack them as you want. And it you will miss all the fun :)

#### Jails

This is where the fun is :)

Jails are using templates, actually there is only two templates (before linux was a template but the support as been deprecated due to bugs and 32 bits limitation):

* Freebsd Jail - Basic FreeBSD jail
* VirtualBox Jail - VirtualBox with phpVirtualBox frontend to virtualize linux/windows! Cool :)

The [jails documentation](http://doc.freenas.org/9.3/freenas_jails.html#jails) is a good starting point.

Another cool thing is a tool called [Warden](http://wiki.pcbsd.org/index.php/Warden®/10.0) to manage and deal with your jails. It makes setting up jails easier !

Cli usage:

    Warden version 1.3


    ---------------------------------
     Available commands
     Type in help <command> for information and usage about that command
             help -  This help file
              gui -  Launch the GUI menu
             auto -  Toggles the autostart flag for a jail
          bspkgng -  BootStrap pkgng and setup TrueOS repo
          checkup -  Check for updates to a jail
           chroot -  Launches chroot into a jail
           create -  Creates a new jail
          details -  Display usage details about a jail
           delete -  Deletes a jail
           export -  Exports a jail to a .wdn file
            fstab -  Start users $EDITOR on jails custom fstab
              get -  Gets options list for a jail
           import -  Imports a jail from a .wdn file
             list -  Lists the installed jails
             pkgs -  Lists the installed packages in a jail
             pbis -  Lists the installed pbi's in a jail
              set -  Sets options for a jail
            start -  Start a jail
             stop -  Stops a jail
             type -  Set the jail type (pbibox|pluginjail|portjail|standard)
         template -  Manage jail templates
        zfsmksnap -  Create a ZFS snapshot of a jail
     zfslistclone -  List clones of jail snapshots
      zfslistsnap -  List snapshots of a jail
     zfsclonesnap -  Clone a jail snapshot
      zfscronsnap -  Schedule snapshot creation via cron
    zfsrevertsnap -  Revert jail to a snapshot
       zfsrmclone -  Remove a clone directory

The interresting feature is import/export that let you create your jail on another system and let you deploy it to your freenas when ready.
Unfortunately on Freenas 9.3 this feature is broken and should not be used anyway (explaination below).

## Freenas Installation

Quite easy even for a headless system, you just have to grab the iso installation, boot it for example on your desktop using virtualbox with a usb key attached, perform the installation and... plug the key to your target system and let it boot ! By default it will grab an IP through DHCP and let you configure everything.

### Creating your zpool and datasets

I let you read the [documentation](http://doc.freenas.org/9.3/freenas_storage.html), you can next tweak your system for powersaving, email alerts, services...

### Creating our First Jail

The idea behing that is to host some services (SABnzbd,CouchPotatoe,Sonarr) inside the same jail and mount the proper dataset to the jail using [nullfs](https://www.freebsd.org/cgi/man.cgi?query=nullfs) this allow you to mount one part of the file system in a different location (like bind option for mount under linux systems).

You **MUST** use the Freenas UI do create your own jail! Under the hood it will use a slightly customised version of [Warden](http://wiki.pcbsd.org/index.php/Warden®/10.0). I will detail in the next chapters what's going on when you create a jail.

As everything is managed through the UI dealing with warden directly could be a bad idea and lead to weird behavior or worse.

By default creating a new jail will create a new dataset in the jail root directory.

You can change this path in the UI it will update the path  in `/etc/local/warden.conf`.

All the jails creation parameters are documentent [on the freenas doc](http://doc.freenas.org/9.3/freenas_jails.html#adding-jails).

#### Under the hood,

This is how it works when you push the **create** button in the UI (more or less).

The first time warden will download a template for the jail we want to create, this is done once. Afteward, and this is were it's clever, when you create a new jail., Warden does a snapshot of the plugin template dataset and then creates an individual plugin jails as ZFS clones of the template snapshot. When creating a new jail the UI will use warden with all paramters you set.

For entertainement purpose I split the creatins step below:

##### Creating a new jail from template:

```tcsh
[root@freenas] ~# warden create myjail --vanilla --startauto

Building new Jail... Please wait...
zfs clone test/jails/.warden-template-9.1-RELEASE-amd64@clean test/jails/myjail
Added new repo: "Official PC-BSD Repository" to the database.
Mounting user-supplied file-systems
jail -c path=/mnt/test/jails/myjail  host.hostname=myjail allow.raw_sockets=true persist
Starting jail with: /etc/rc
Success!
Jail created at /mnt/test/jails/myjail
```

##### Bring up the network

Activate ipv4 dhcp on the jail and vimage

```
warden set ipv4 myjail DHCP
warden set vnet-enable myjail
```

Start your jail again

```tcsh
[root@freenas] ~# warden start myjail

Mounting user-supplied file-systems
jail -c path=/mnt/test/jails/myjail name=myjail host.hostname=myjail allow.raw_sockets=true persist vnet=new
Getting IPv4 address from DHCP
DHCPDISCOVER on epair0b to 255.255.255.255 port 67 interval 7
DHCPOFFER from 10.0.0.1
DHCPREQUEST on epair0b to 255.255.255.255 port 67
DHCPACK from 10.0.0.1
bound to 10.0.0.5 -- renewal in 1800 seconds.
inet6_enable:  -> YES
ip6addrctl_enable: YES -> YES
route: writing to routing socket: File exists
add net default: gateway 10.0.0.1: route already in table
Starting jail with: /etc/rc

```

##### Adding some storage

It will edit your jail fstab, you can use the `warden fstab myjail`, to see for example

```
/mnt/test/mydataset /mnt/test/jails/myjail/data nullfs rw 0 0
```

Note: the edited fstab is not the fstab file located in the jail, but is located under medata directory of you jail.

**This part if done by hand using warden is overwritten by the UI when you will start the jail as the storage options are not in the UI configuraiton database**

This is why you need to use the UI to add storage to your jails...


### zfs PoV for jails

As I explained before new jails are based on clone of snaptshot of the template jail:

```tcsh
[root@freenas] ~# zfs list -o origin,name,used,avail,refer,mountpoint

ORIGIN                                               NAME                                           USED  AVAIL  REFER
-                                                    test/jails/.warden-template-9.1-RELEASE-amd64  205M  5.89G   205M
-                                                    test/jails/.warden-template-pluginjail         719M  5.89G   719M
test/jails/.warden-template-9.1-RELEASE-amd64@clean  test/jails/myjail                             2.55M  5.89G   207M
test/jails/.warden-template-pluginjail@clean         test/jails/sabnzbd_1                           122M  5.89G   840M
test/jails/.warden-template-pluginjail@clean         test/jails/sabnzbd_2                            96K  5.89G   719M
```

You can easily see the origin for each jails and the most import part, the USED size. That a clever use of zfs with jails (somehow docker is doing the same thing using AUFS)

### Hacking on your fresh jail

#### SSH to your jail

Something usefull is to enable ssh on your jail by editing the `/etc/rc.conf` and editing the ssh line to set to YES and change the hostname:

```
sshd_enable="YES"
hostname="media_grabbers"
```

Edit the `/etc/ssh/sshd_config` and add the line

```
PermitRootLogin Yes
```

Create a root password using `passwd` and then `service sshd start`, this will create the keys.

#### Install your first packages

Packagement management could done through [pkg or port](https://www.freebsd.org/doc/handbook/ports.html)

* pkg is for binary grab and install (deb/rpm like)
* port tree if for building the package from source (gentoo/archlinux like)


First update you jail using `pkg upgrade` and update the port tree `portsnap fetch extract update` then you can install some packages:

```
pkg install zsh tmux py27-virtualenv py27-virtualenvwrapper py27-pip
curl -L http://install.ohmyz.sh | sh
chsh -s /usr/local/bin/zsh
```

Now I fell like home :)


### How to export/import my jails between systems

That's a good point... but we can't use warden to do that because it's broken and doint things on the back of the UI is not a good idea... but we can find two other ways:

* Export the jail dataset using `zfs send / reveive` if you need to transfert existings jails. (warning if you transfert jails from clones you will have to transfert the origin snapshot as well)

* Use the builtin template system (this is smarter especially if you need to deploy several jails afterward)

#### Make your jail a template

My up to date and custom jail is now ready, it could be great to use it as a base for other jails comming and use the clone of snaptshot zfs clever trick for this.

* Create a snaptshot of your current jail:

    `warden zfsmksnap myjail` or `zfs snapshot test/jails/myjail@my_snap`
    This will allow you to keep a versioning from your base jail or templating from a running jail.
    Snapshots are accessibles from an hidden directory [.zfs/snapshot](http://docs.oracle.com/cd/E19253-01/819-5461/6n7ht6r4j/index.html).

* Package you jail for transport

    ```
    cd /mnt/test/jails/myjail/.zfs/snapshot/my_snap/
    tar czf /mnt/test/mydataset/base.tgz .
    ```

    It's important to create the tgz **outside** of the jail (here I create the tgz from freenas), you can use the `--exclude` tar parameter if you want to do this from the jail.
    `zfs send` is here not usable because this command is meant to send a full dataset (not only the content) targeted to a zpool.

* Deploy

For this you will have two choices to import your template:

1. Deploy the tgz to a ftp or http server and add a new template ([using the UI](http://doc.freenas.org/9.3/freenas_jails.html#managing-jail-templates)) (this is the supported way)
    Note: For an unknow reason this doesn't work using `python -m SimpleHTTPServer`, see [Bug #7811](https://bugs.freenas.org/issues/7811)

2. Create a new template from your tar with `warden template create -tar base.tgz -nick custom_template`
    * The trick to make it appears in the UI is to create a fake template entry using the **same name** as you set as nick (url could be wrong but not empty)

Your done! Create a new jail from your custom template (should be fast) and you can chack that your new jail **is** using the template.

```tcsh
[root@freenas] /mnt/test/jails# zfs list -o origin,name,used,avail,refer,mountpoint
ORIGIN                                            NAME                                         USED  AVAIL  REFER
-                                                 test/jails                                  2.34G  5.22G   788M
-                                                 test/jails/.warden-template-custom_template  207M  5.22G   207M
test/jails/.warden-template-custom_template@clean test/jails/new_jail_from_custom_template    2.44M  5.22G   207M
```
(I stripped the other entries for better readability)

## Media grabber installation

In the previous chapter we save how to deploy a custom jail and how to mount (bind) a dataset to it. Now it's time for us to install every pieces of software needed in order to grab our favorite media.

### Users and permissions

One fondamental thing is to be coherent between the permissions on your dataset and the the permissions your process will run as in your jail.

* Freenas side:

Create a new user on freenas (using UI or cmd line as you like), let's say something like this user:

```
media:*:1001:1001:Media user:/nonexistent:/sbin/nologin
```

and for group

```
media:*:1001:
```

No shell, no home and login is better for security.

On your dataset be sure that this user a the permissions deal with your files. For this you can use the UI or a `chmod -R media.media /mnt/test/mydataset`

**Note:** as every media files belong to media, I also make the AFP/NFS guest account media in order to access to the share from my mac as a guest with full privileges. (I trust my own network).

* Jail side:

Same thing as before:

```tcsh
➜  ~  adduser
Username: media
Full name: Media
Uid (Leave empty for default): 1001
Login group [media]:
Login group is media. Invite media into other groups? []:
Login class [default]:
Shell (sh csh tcsh zsh rzsh git-shell nologin) [sh]: nologin
Home directory [/home/media]: /nonexistent
Home directory permissions (Leave empty for default):
Use password-based authentication? [yes]: no
Lock out the account after creation? [no]: no
```

From there you can mount your dataset using the UI to the folder of your choice and check if everything is fine using

```tcsh
➜  ~  sudo -u media ls -al /media
total 419
drwxrwxr-x  12 media  media     13 Feb  1 22:26 .
drwxr-xr-x  17 root   wheel     21 Feb  1 13:19 ..
drwxrwxr-x   3 media  media      6 Feb 17  2014 Test
```

### Software installation

For git clone sotware we will install them in

```tcsh
mkdir /grabbers
chown media:media /grabbers
```

**[Sabnzbd](http://sabnzbd.org)**

```tcsh
cd /usr/ports/news/sabnzbdplus && make config-recursive && make install clean
chown media:media /usr/local/sabnzbd
# make listen to 0.0.0.0 (needed for first configuration)
sed -i.ori 's/host = localhost/host = 0.0.0.0/g' /usr/local/sabnzbd/sabnzbd.ini
sysrc sabnzbd_enable="YES"
sysrc sabnzbd_user="media"
sysrc sabnzbd_group="media"
```

**[Sonarr](http://sonarr.tv)**

```tcsh
pkg install mono mediainfo sqlite wget
cd /grabbers
fetch http://download.sonarr.tv/v2/master/mono/NzbDrone.master.tar.gz
tar -xzvf NzbDrone.master.tar.gz
chown -R media:media NzbDrone
```

Then you need to create a `/usr/local/etc/rc.d/sonarr` [service script](https://www.freebsd.org/doc/en_US.ISO8859-1/articles/rc-scripting/rcng-hookup.html) in order to make it run.

```sh
#!/bin/sh

# $FreeBSD$
#
# PROVIDE: sonarr
# REQUIRE: LOGIN
# KEYWORD: shutdown
#
# Add the following lines to /etc/rc.conf.local or /etc/rc.conf
# to enable this service:
#
# sonarr_enable:  Set to YES to enable sonarr
#           Default: NO
# sonarr_user:    The user account used to run the sonarr daemon.
#           This is optional, however do not specifically set this to an
#           empty string as this will cause the daemon to run as root.
#           Default: sonarr
# sonarr_group:   The group account used to run the sonarr daemon.
#           This is optional, however do not specifically set this to an
#           empty string as this will cause the daemon to run with group wheel.
#           Default: sonarr
# sonarr_data_dir:    Directory where sonarr configuration
#           data is stored.
#           Default: /var/db/sonarr

. /etc/rc.subr
name=sonarr
rcvar=${name}_enable
load_rc_config $name

: ${sonarr_enable:="NO"}
: ${sonarr_user:="sonarr"}
: ${sonarr_group:="sonarr"}
: ${sonarr_data_dir:="/var/db/sonarr"}

pidfile="${sonarr_data_dir}/nzbdrone.pid"
command="/usr/sbin/daemon"
procname="/usr/local/bin/mono"
command_args="-f ${procname} /grabbers/NzbDrone/NzbDrone.exe --data=${sonarr_data_dir} --nobrowser"

start_precmd=sonarr_precmd
sonarr_precmd() {
    if [ ! -d ${sonarr_data_dir} ]; then
        install -d -o ${sonarr_user} -g ${sonarr_group} ${sonarr_data_dir}
    fi
}

run_rc_command "$1"
```

**Note**: Don't forget to set the execution attributes (555).

And finally append the followin to your `/etc/rc.conf`

```tcsh
sysrc sonarr_enable="YES"
sysrc sonarr_user="media"
sysrc sonarr_group="media"
```

**[CouchPotato](http://couchpota.to)**

```tcsh
cd /grabbers && git clone git://github.com/RuudBurger/CouchPotatoServer.git
chown -R media:media CouchPotatoServer
cp /grabbers/CouchPotatoServer/init/freebsd /usr/local/etc/rc.d/couchpotato
# fix the path
sed -i.ori 's/\/usr\/local\/CouchPotatoServer/\/grabbers\/CouchPotatoServer/g' /usr/local/etc/rc.d/couchpotato
chmod +x /usr/local/etc/rc.d/couchpotato
# fix python path
ln -sf /usr/local/bin/python2.7 /usr/bin/python
sysrc couchpotato_enable="YES"
sysrc couchpotato_user="media"
```

### Check if everything is running fine
To see if everything is running fine you can use the `sockstat` command

```tcsh
sockstat -4
USER     COMMAND    PID   FD PROTO  LOCAL ADDRESS         FOREIGN ADDRESS
media    python2.7  9221  7  tcp4   *:8080                *:*
media    python2.7  6044  50 tcp4   *:5050                *:*
media    mono-sgen  3921  12 tcp4   *:8989                *:*
```

## Access to our grabbers

Accessing each grabber using ip:port is a pain, we will create a small reverse proxy on top of then, this way you can access them using url/service scheme. For this will use nginx.

### Install nginx

```tcsh
pkg install nginx
sysrc nginx_enable=YES
sysrc nginx_profiles="grabber"
sysrc nginx_grabber_configfile="/usr/local/etc/nginx/grabber.conf"
```

* Create your certificates for ssl

For testing purpose we can deal with autosigned certs

```tcsh
mkdir -p /etc/ssl/{private,certs}
openssl genrsa -des3 -passout pass:x -out server.pass.key 2048
openssl rsa -passin pass:x -in server.pass.key -out /etc/ssl/private/server-key.pem
rm server.pass.key
openssl req -new -key /etc/ssl/private/server-key.pem -out server.csr -subj "/C=US/ST=Somewhere/L=Somewhere/O=MyHome/OU=Home/CN=grabber.home.local"
openssl x509 -req -days 365 -in server.csr -signkey /etc/ssl/private/server-key.pem -out /etc/ssl/certs/server-cert.pem
#secure our private key
chown www /etc/ssl/private/server-key.pem
chmod 600 /etc/ssl/private/server-key.pem
```


In `/usr/local/etc/nginx/`, create:

* grabber.conf

```
user www;
worker_processes  1;

error_log  /var/log/nginx-grabber-error.log;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx-grabber-access.log  main;
    sendfile        on;
    keepalive_timeout  65;
    gzip  on;
    gzip_http_version 1.0;
    gzip_min_length 1024;
    gzip_proxied any;
    gzip_buffers 16 8k;
    gzip_types text/plain text/css application/x-javascript text/xml
               application/xml application/xml+rss text/javascript;
    gzip_vary on;

    client_max_body_size 4G;

    server_tokens     off;

    ssl_ciphers ECDHE-RSA-AES128-SHA256:AES128-GCM-SHA256:RC4:HIGH:!aNULL:!MD5:!EDH;
    ssl_prefer_server_ciphers on;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_session_cache shared:SSL:10m;

    proxy_connect_timeout 90;
    proxy_send_timeout 90;
    proxy_read_timeout 90;
    proxy_buffer_size 4k;
    proxy_buffers 4 32k;
    proxy_busy_buffers_size 64k;
    proxy_temp_file_write_size 64k;

    # Allow only local ip to 80
    server {
        listen       80;
        # allow localnetwork and localhost
        allow 192.168.0.0/24;
        allow 127.0.0.1/32;
        deny all;
        include proxy.conf;
    }

    server {
        listen       443 ssl;
        satisfy any;
        ssl_certificate     /etc/ssl/certs/server-cert.pem;
        ssl_certificate_key /etc/ssl/private/server-key.pem;
        ssl_client_certificate /etc/ssl/certs/server-cert.pem;
        ssl_verify_client on;
        include proxy.conf;
    }
}
```

* proxy.conf

```
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Host $host;
proxy_set_header X-Forwarded-Server $host;

# sabnzbd
location /sab {
    proxy_pass http://127.0.0.1:8080/sabnzbd;
    proxy_redirect default;
}
# sonarr
location /tv {
    proxy_pass http://127.0.0.1:8989;
    proxy_redirect default;
}
#couchpotato
location /movies {
    proxy_pass http://127.0.0.1:5050;
    proxy_redirect default;
}
```

**Note 1:** If you are using a certificate chain be carefull to append the server cert first in the concatenated certs (cert, intermediate ca, ca). Don't forget to set `ssl_verify_depth x;` with x according to your certification chain depth.

**Note 2:** SSL CA are not provided by default by freebsd they can be install with the package `ca_root_nss`, don't forget to symlink it `ln -sf /usr/local/etc/ssl/cert.pem /etc/ssl/cert.pem` (could cause issue with python apps)

So basically the http traffic is only allowed from local network, ssl with client cert for 443 (used to publish over internet).

We can't use both `ssl_verify_client` and `allow x.x.x.x/x` with `satisfy any` directive (nginx limitation). So we need to split them.

For some services (like sonarr,couchpotatoe) you will have to set URL Base (in settings) in order to make it works with the reverse proxy.

Once everything respond using the reverse proxy, you can change every service to listen only on localhost (using UI of services or manualy).

*   sabnzbd: Using UI in wizard
*   CouchPotatoe: Adding `host = 127.0.0.1` in `/grabbers/CouchPotatoServer/data/settings.conf`
*   Sonarr: Using UI to change both base url / listening IP



TODO:
dns update
custom alerts

