## Installation Instructions for setting up InfiniBand on Fedora 26

**NOTE and Disclaimer** - While these instructions are specific in parts, they will not help with debugging issues you may encounter. It may seem beginner ready, but this does require a good bit of Linux knowledge or a lot of debugging skills (including the all-helpful Google open on another computer) to get things going. I do not provide much in the way of troubleshooting help, where I would expect some general Linux knowledge to have the answer.

I do not take responsibility for the issues you may encounter, or if you completely wipe your system while using ```dd```, for example.

### Step 1: Install Fedora 26

#### Download the Iso

I fetched Fedora from [http://mirror.clarkson.edu](http://mirror.clarkson.edu) and went to the Distributions page, and selected [Fedora Iso's](http://mirror.clarkson.edu/fedora/linux/releases/)

Once there, I clicked on ```26```, and then ```Everything``` and then ```x86_64``` and then ```iso```. ([Link to folder](http://mirror.clarkson.edu/fedora/linux/releases/26/Everything/x86_64/iso))

I downloaded the iso file onto my computer (which is running Linux) and then ran ```sync``` to make sure that the file was actually on my computer, and not just in RAM. (This is not strictly necessary).

#### DD the iso to a flash drive

I then copied the installation media onto my flash drive using the ```dd``` program.

This is a very specific instruction, you *cannot* just copy the iso file to a flash drive in Windows. You must use ```dd``` or a similar program to extract the iso onto the disk properly.

To do this, I ran the following commands:

```lsblk``` - This program will tell you the partition layout and block devices available to the system. Generally, look for the device you want by its size (ex, ```/dev/sdb```) and then if necessary, unplug and replug to make sure you have the correct device.

I then use ```dd``` to write the file to the drive. The command looks as below:

```bash
dd if=fedora.iso of=/dev/sdb bs=4M
```

```dd``` is the command, ```if=<path to iso>``` is the path to the iso file on your local system, and ```of=/dev/sdb``` is the path to the file. ```bs=4M``` changes the size to copy per iteration, and generally the file will copy fastest at around 4M, but sometimes lower. Depends on your hardware. Bigger (or smaller) does not necessarily regulate how fast it is.

**WARNING** - ```dd``` can directly overwrite your operating system! Be sure you are doing it to the right filesystem, or you could wipe out your running operating system, or other documents. This is completely irrecoverable, even with special software!

Once the ```dd``` program has finished, you want to run ```sync``` again. This will force all of the changes to be applied to the flash drive. After you run this, it is safe to remove the flash drive.

#### Booting the system

**NOTE** - This was hardware specific. You may have to do something different.

On the servers in the lab, you press <kbd>F2</kbd> for Setup, <kbd>F6</kbd> for Boot Menu, and <kbd>F12</kbd> for PXE network boot.

**NOTE** - End hardware specific zone

You may need to enter the BIOS to change the boot order. Once you do that, you need to boot from your bootable flash drive on the system.

#### Installer

From here on, I will call the primary disk ```/dev/sda```

Once the bootable drive has booted, you will see the installer. Just flow through the instructions. I generally start by configuring the disk partitions. I always use the advanded partitioner in basic partition mode (not LVM or BTRFS, etc), which is good for drives up to 2TB. Beyond that, you may need to use UEFI, which requires a special 100MiB partition at the beginning of the disk, which has the /boot folder in it.

Here's a general layout:

##### For MBR based installs:

```
/dev/sda
|- /dev/sda1 - ext4 primary partition with >8GB of space, mounted as /
\- /dev/sda2 - swap primary partition with ~8GB of space. (Optional)
```

##### For UEFI based installs:

```
/dev/sda
|- /dev/sda1 - FAT16/FAT32 primary partition with 100MiB of space, mounted as /boot
|- /dev/sda2 - ext4 primary partition with >8GB of space, mounted as /
\- /dev/sda3 - swap primary partition with ~8GB of space. (Optional)
```

Depending upon whether you have enough RAM for the task at hand, you may want to have swap available. Don't worry, you can also put swap on the ext4 filesystem as well (with a neglible performance hit).

After I have set up the partitions, it's a good idea to set the Mirror.

A mirror is the location that it downloads the files from. You can leave it as the default, but if you want to have the best response, you want to select the closest geographic mirror, considering bandwidth.

At Clarkson University, we have a student-operated [mirror](http://mirror.clarkson.edu) on campus, which has a very fast connection (10G backbone on campus).

The Mirror URL is as follows to use in the dialog that appears when you select the Mirror for Fedora 26:

```
http://mirror.clarkson.edu/fedora/linux/releases/26/Everything/x86_64/os/
```

Once you enter that, click Done. It will take a moment, as it gathers all of the package information.

Next, select the software configuration menu. I selected Server on the left side, and then I selected C/C++ libraries, network utilities, RPM build tools, and other useful tools of which I cannot recall. **(Note to self - update this)**

Once you do that, go back to the main menu (click done) click Begin Install, and then the magic begins.

It will then prompt you for the root password, and a username and password. You will want to make the user administrator. (This means they are added to the wheel group and can use ```sudo```, which is super important later, but so long as you rembmer the root password, you can get anywhere)

After that, you wait for about 30+ minutes as it downloads and installs the packages. The time it takes depends on your disk and network speed for the most part, but a good CPU helps a lot as well.

### First Boot configuration

First comes first, we need to configure the system.

If you haven't already gathered an IP, you will need to get one. Usually, DHCP works out of the box if your network hardware is supported.

Once you know you have a connection, continue.

Run all of the below with the root user (or use ```sudo -i``` on the user account)

```bash
# update the system (this is mandatory or you will hate yourself later)
dnf update

# say yes to all of the prompts (press y and enter)

# install Vim (a text editor), htop (a process monitor), and iperf (a network bandwidth tester)
dnf install -y vim htop iperf

# I tend to disable NetworkManager. It gets in the way too much.
systemctl disable NetworkManager
systemctl stop NetworkManager

# Set the hostname. Delete whatever is there, and then add a hostname which does not start with a number and contains only alphanumeric characters or dashes. No whitespace
vim /etc/hostname

# Reboot to allow changes to occur to the kernel and other hardware that was just updated above
reboot
```

On reboot:

```bash
# When you reboot, there will not be an IP address. Get one via DHCP
# First, we need to determine which ethernet adapter is connected to the internet.
ip a

# Look at the output of this command. lo is always connected, but generally en(p)#(s)#(f)# will be ethernet (where # is generally a number)
# Generally, the first one below lo is what you want. On my system, it's ens2f0
# It will say something like "up" or "lower up" often and will not say "no carrier"

# If it says down, you may just need to poke it with "ip l set ens2f0 up", if it's still down, that is not the correct interface, and it's not connected

# run dhclient on the connected interface to get an IP address
dhclient ens2f0

# where ens2f0 is replaced by the interface that is connected.

# check to see if we have an IP address
ip a l dev ens2f0

# Look for an inet line. The thing directly after inet will be your IP address.

# ex: inet 10.0.3.4/24 ...

# If you have an IP address, you're good to go!
```

### Installing the IB drivers

In general, our hopes are to use OEFD for drivers and such.

We still need to nab some dependencies though.

#### Dependencies

```bash
# Install dependencies for IB
dnf install -y opensm rdma opensm-libs libibumad libibverbs libibverbs-devel glib2-devel  libibumad-devel opensm-devel libibmad tk libibmad-devel tk-devel libibcommon libvma librdmacm openssl-devel ibutils infiniband-diags

# note, I am adding some of these after the fact, they may not be able to be installed yet. Install them as you can otherwise

# 7z, a file decompression utility (because I like it, tar and unzip are also good)
dnf install -y p7zip p7zip-plugins

# If you're using a GUI, GUI for 7z: (commented to prevent mistakes such as installing Xorg)
#dnf install -y p7zip-gui

```

We need to fetch the drivers from OFED's page as well.

#### Custom Drivers

Go [here](http://downloads.openfabrics.org/OFED/ofed-4.8-1/) and then get the latest version. (Note, if a newer version has come out of 4.8.1, go up a folder and then get the newest version that is not daily AND is a greater number than your kernel version, found from running ```uname -a``` and reading the third field.)

```bash
# Download the RPM sources
wget http://downloads.openfabrics.org/OFED/ofed-4.8-1/OFED-4.8-1-rc2.tgz -O ofed.tgz

# Extract the files
7z x ofed.tgz
tar -xvf ofed.tar

# Optionally, remove the archives
rm ofed.t*

# cd into the new folder.
cd ofed..somepath../SRPMS

# Create the sources for installing from.
rpm -ihv *.rpm

# move to build directory
cd ~/rpmbuild/SPECS

# Run ls here. You should find a bunch of .spec files. If not, something is wrong.
ls

# This next stuff will take much longer than everything else, since it's compiling all of the packages from source.

# Install the necessary requirements (in this order)
rpmbuild -ba ~/rpmbuild/SPECS/libfabric.spec
rpmbuild -ba ~/rpmbuild/SPECS/libibscif.spec
rpmbuild -ba ~/rpmbuild/SPECS/infiniband-diags.spec
rpmbuild -ba ~/rpmbuild/SPECS/opensm.spec
rpmbuild -ba ~/rpmbuild/SPECS/ibpd.spec
rpmbuild -ba ~/rpmbuild/SPECS/dapl.spec

# Throw a reboot in here somewhere...

```

### Connectiong IPoIB

First, make sure that your connection is established with ```ibstat```.

```bash
ibstat
```

The output will spit out a bunch of information about the link, but the two things we care about are the ```State``` and ```Physical State``` lines under the port definition.

If the link state is ```Active```, then you are ready to go onto the next step.

If the link state is ```Down``` and the physical state is ```Polling```, then you need to start ```opensm``` (the OPEN Subnet Manager) on the master server.

If you get something else and the link state is down, then you need to check the physical connections.

#### Example Output

Here's an example output of a happy InfiniBand link: (this is from the internet)

```
# ibstat
 CA 'mlx4_0'
       CA type: MT26428
       Number of ports: 1
       Firmware version: 2.9.1000
       Hardware version: b0
       Node GUID: 0x0002c903004af586
       System image GUID: 0x0002c903004af589
       Port 1:
               State: Active
               Physical state: LinkUp
               Rate: 40
               Base lid: 250
               LMC: 0
               SM lid: 250
               Capability mask: 0x0251086a
               Port GUID: 0x0002c903004af587
               Link layer: InfiniBand
```

### Establish IP link

Now that we have established the physical layer, we need to establish IP addresses. Potentially, you could have a router or DHCP server later which can do this automatically, but this assumes such an action has not been carried out.

**NOTE** - This is not presistent over reboots!

#### Find the link

From the output of the below program, determine the interface name for the Infiniband network. It should be ```ib0``` in most configurations.

```bash
ip l
```

#### Set the link to enabled

Set the interface to the ```Up``` state. This permits IP traffic to traverse the network.

```bash
ip l set dev ib0 up
```

#### Add an IP address

Set the IP address of the network. In this case, I am setting the IP to ```10.0.3.2\24```, which is an IP of ```10.0.3.2``` with a [CDIR](https://doc.m0n0.ch/quickstartpc/intro-CIDR.html) subnet mask equivelant to the netmask ```255.255.255.0```. You also specify that you are adding the IP address, and that you want it on device ```ib0```.

```bash
ip a add 10.0.3.2/24 dev ib0
```

#### Verify the IP address and routes

Now, verify that you have correctly established the IP address.

```bash
ip a
```

It should list the IPv4 address that you now have. The line you care about is the ```inet``` line. The first quad octet (seperated by dots) is the IP address, followed by the CDIR notation for the subnet (in this case, ```\24```).

Example: (this is my WiFi interface on my laptop)

```
3: wlp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether f4:96:34:eb:48:26 brd ff:ff:ff:ff:ff:ff
    inet 192.168.8.183/24 brd 192.168.8.255 scope global dynamic wlp1s0
       valid_lft 41925sec preferred_lft 41925sec
    inet6 fde5:8f5b:570a::569/128 scope global
       valid_lft forever preferred_lft forever
    inet6 fde5:8f5b:570a:0:b7b7:6882:475c:4f80/64 scope global noprefixroute
       valid_lft forever preferred_lft forever
    inet6 fe80::6a5b:a280:4863:8348/64 scope link
       valid_lft forever preferred_lft forever
```


You may also want to check the routes:

```bash
ip r
```

This will show the default route for each subnet and to which interface it should go to. Generally, when you set yourself an IP, it should be on a seperate subnet from other IP addresses that are available on other interfaces such as other subnets.

In general, for most cases, the ```10.0.0.0/8``` is a perfectly good place to fiddle with this, and if you have a ```/24``` in this space, there is plenty of octets worth of space to fiddle (in fact, over 65k unique ```/24``` subnets within a ```/8```).

Example: (this is also my laptop)

```
default via 192.168.8.1 dev wlp1s0 proto static metric 600
192.168.8.0/24 dev wlp1s0 proto kernel scope link src 192.168.8.183 metric 600
```

#### Addendum: Get a DHCP leased IP address

This is particularly useful if you want to recieve an IP address for connecting over ethernet to the normal network for SSH access to the servers. If you configure a DHCP server on a server, you could also use this command to recieve an IP address.

This command below specifies to run the dhclient program (which gets an IP over DHCP) from the ```ens2f0``` interface. Look at ```ip l```'s output to determine the interface you need. Often, they are formatted en(s#)(p#)(f#) to be as unique as possible against other interfaces on the system.

```bash
dhclient ens2f0
```

In a moment, if you did it to the correct device, you will have an IP address, which will be shown by the output of ```ip a```.

### Testing the capabiliites of IPoIB

To test the speed of the IB network from the IP layer, we will be using the program ```iperf```.

If you didn't install it already, do that now.

```bash
dnf install -y iperf
```

You will need this program on both the server and the client.

To run the server side program, enter the below:

```bash
iperf -s
```

To run the client side program (the server must be running, and only one client can connect at a time):

```bash
iperf -c 10.0.3.2
```

where ```10.0.3.2``` is replaced with the IP address where the iperf server is running.

In a moment, you will see an output, which specifies the size of data that it moved in a specific amount of time, as well as the general link speed. Check the units to see which is which.

### Random Debugging Tidbits

If you forgot to add a user to administrator, you can do it later with the following command.

```bash
gpasswd -a <username> wheel
```

Printing the users (this shows their name, uid, shell, etc)

```bash
getent passwd
```

Printing the groups (this shows the groups, gid, and their memeber users)

```bash
getent group
```
