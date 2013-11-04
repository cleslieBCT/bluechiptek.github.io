---
layout: post
title: "Installing OpenStack Cinder in 10 Minutes"
date: 2013-11-04 11:38
comments: true
categories: openstack
---
The [Cinder project](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=1&ved=0CC4QFjAA&url=http%3A%2F%2Fwiki.openstack.org%2FCinder&ei=4IonUufGCIPuiQKS2oDICA&usg=AFQjCNEMptnoNy9ZGAR6Qbuy9KhaMfvZzQ&sig2=6LIC8BxMqWAbsycyyUNL3A&bvm=bv.51773540,d.cGE) provides volume storage management for OpenStack deployments.  Cinder volumes appear as block storage devices to instances started in an OpenStack environment.  Cinder is also used by OpenStack for storage of instance snapshots and can be used to boot cloned images of existing running instances.

You can familiarize yourself with the Cinder install process by watching the video below.

[![ScreenShot](https://raw.github.com/rackerlabs/vagrantstack/master/openstack_movie.png)](http://vimeo.com/73001135)

Before we drop into the guide, I'd like to thank [Blue Chip Tek](http://bluechiptek.com) for providing hardware, advice and setup assistance, [Dell Computers](http://dell.com/) for donating the test hardware, and the awesome folks at [Rackspace](http://rackspace.com/) for writing and supporting the [Chef scripts](https://github.com/rcbops/chef-cookbooks) which are used for the bulk of the setup process.

### Prerequisites for Install
Before installing Cinder, you should ensure you've installed your OpenStack cluster using the [Installing OpenStack in 10 Minutes](https://github.com/bluechiptek/bluechipstack/blob/master/README.md) guide.  The primary requirement is you have a local instance of the Vagrant Chef server running which will be used in this guide to deploy Cinder to your nodes.

Cinder does storage management, so you'll also need a storage device configured to store data.  Typically this is done with hard drives or SSDs.  This guide currently uses the method of creating a 1TB loopback device attached to a file stored in the root partition on each node in your cluster.  If your root partition doesn't have 1TB of storage available on it, you can either shrink the number to accomidate or install and provision a physical volume on another storage device.

*Note: This guide will be updated to include instructions for using a physical volume as the storage device in a later revision.*

### Build the Loopback File and Persist It
Start by checking on the amount of storage you have available on your node.  This guide assume the loopback file will be created in the root partition.  Become root and then execute the following command:

	df -ah /
	
Take a gander at the result:

	root@flop:/home/kord# df -ah /
	Filesystem      Size  Used Avail Use% Mounted on
	/dev/sda2       2.8T  2.5G  2.6T   1% /

We're good to create a 1TB file on that partition.  Now execute the command to create the file we'll loop:

	dd if=/dev/zero of=cinder-volumes bs=1 count=0 seek=1024G

Results come back instantly:

	root@flop:/home/kord# dd if=/dev/zero of=/cinder-volumes bs=1 count=0 seek=1024G
	0+0 records in
	0+0 records out
	0 bytes (0 B) copied, 1.035e-05 s, 0.0 kB/s

Now loop up the file to the loopback device (we'll use loop2 here):

	losetup /dev/loop2 /cinder-volumes
    
Finally, create a rc file to mount the loopback file after a reboot:

	echo "losetup /dev/loop2 /cinder-volumes; exit 0;" > /etc/init.d/cinder-setup-backing-file
	chmod 755 /etc/init.d/cinder-setup-backing-file
	ln -s /etc/init.d/cinder-setup-backing-file /etc/rc2.d/S10cinder-setup-backing-file
    
### Physical & Volume Group Creation
The next step is to initialize a LVM volume named *cinder-volumes*.  That name is used by default for all Cinder installs.  It can be changed in the configuration files in **/etc/cinder/** after you install it, but we'll assume here we're leaving it as the default.   Run the commands in one cut and paste frenzy:

	sudo pvcreate /dev/loop2
	sudo vgcreate cinder-volumes /dev/loop2

Revel in the vast awesomeness of your successful paste job:

	root@flop:/home/kord# sudo pvcreate /dev/loop2
	Physical volume "/dev/loop2" successfully created
	root@flop:/home/kord# sudo vgcreate cinder-volumes /dev/loop2
	Volume group "cinder-volumes" successfully created
    
Double check your work with **pvdisplay** and **vgdisplay**:
      
	root@flop:/home/kord# pvdisplay; vgdisplay
	  --- Physical volume ---
	  PV Name               /dev/loop2
	  VG Name               cinder-volumes
	  PV Size               1.00 TiB / not usable 4.00 MiB
	  Allocatable           yes
	  PE Size               4.00 MiB
	  Total PE              262143
	  Free PE               262143
	  Allocated PE          0
	  PV UUID               XnvA5r-wBw0-WIhd-9FHd-n4lT-71em-uYCLJt
	
	        --- Volume group ---
	  VG Name               cinder-volumes
	  System ID
	  Format                lvm2
	  Metadata Areas        1
	  Metadata Sequence No  1
	  VG Access             read/write
	  VG Status             resizable
	  MAX LV                0
	  Cur LV                0
	  Open LV               0
	  Max PV                0
	  Cur PV                1
	  Act PV                1
	  VG Size               1024.00 GiB
	  PE Size               4.00 MiB
	  Total PE              262143
	  Alloc PE / Size       0 / 0
	  Free  PE / Size       262143 / 1024.00 GiB
	  VG UUID               vabQ1q-mxZ2-ofrn-6640-rriR-jln1-WHm4jm
 
### Install Cinder via Vagrant Chef
Move on over to the computer that was used to provision the nodes in your install.  If you've halted the Vagrant server, restart it first and then ssh into it and become root:

    Beast:texasholdem kord$ cd bluechipstack/
	Beast:bluechipstack kord$ vagrant up
	Bringing machine 'default' up with 'virtualbox' provider...
    ...
	[default] -- /vagrant
	Beast:bluechipstack kord$ vagrant ssh
	Welcome to Ubuntu 12.04 LTS (GNU/Linux 3.2.0-23-generic x86_64)
    ...
	Welcome to your Vagrant-built virtual machine.
	Last login: Wed Sep  4 18:54:12 2013 from 10.0.2.2
	vagrant@chef-server:~$ sudo su
	root@chef-server:/home/vagrant#

Now test Chef is still aware of the servers:

    knife node list

Time to install Cinder on the nodes.  Type the following to add the Cinder storage role to the nodes (making sure to change the node name(s) in the process):

    knife node run_list add flop 'role[cinder-volume]'
    knife node run_list add turn 'role[cinder-volume]'
    
### Volume Clear
    

    

    

