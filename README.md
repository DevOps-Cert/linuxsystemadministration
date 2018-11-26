
# linuxsystemadministration
This guide provides lab activities to help co-workers learn to become Linux system administrators.  To practice common Linux system administrative functions, tasks are prescribed and then the fastest way(s) to accomplish the goal is listed for our selected Linux distribution which is Ubuntu 16.04 LTS server.   This also helps me learn markdown for documentation.

Work on list
* SELinux and AppArmor
* PAM modules (figure out fastest way to search for appropriate ones to install and configure)
* Managing libvirt machines (use old competencies as a guide)
* Verify the integrity and availability of resources: fsck and memtest86+ 
# Table of Contents

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

# Setup Process
## Caveats
While this guide could make use of sudo for all super privileged commands (/sbin) to avoid the use of the root account; efficiency of time makes it optimal to use the root account for almost all functions.  In a few cases, an unprivileged account will be used such as testing file and user privileges.  It is with heavy heart that the use of root account herein is documented; but, the goal is speed and this shortcut turns into a tremendous time saver.  Do not pursue this approach with production servers with multiple administrators where you should be using sudo and visudo rather than these shortcuts.  Root can be accessed via `sudo -i` or `sudo su - root`.
## Platform generic focus
Take careful note of the version of Ubuntu as this presumes 16.04.  These instructions are platform specific to Ubuntu 16.04 to avoid extra commands to master.  When possible, generic concepts and methods are covered such as iptables instead of ufw for example.
## Setup notes
There are a multitude of possible ways to setup your learning environment.  However, given the overall benefit of standardizing your configuration to match this guide so that it works every time from scratch and is identical, vagrant is the preferred solution.  A libvirt hypervisor or manual configuration of a group of virtualbox/vm ware hosts can work but provide some complexity to keep standardized.

Port forwarding will be used on localhost so that a ssh client can be used always to the same port to access the Linux boxes.

Setup should several additional physical volume on lb40 for RAID, swap space, quota, and other file system testing.

Make vim default editor: ```update-alternatives --config editor```

To create the working folders to be consistent in /tmp; use ```mkdir /tmp/{1..9}.{1..9}```
### Layout of drives
The disk layout will be as follows for lb40 and is reflected in Vagrantfile supplied (to avoid confusion and drive re-use between exercises as well as partitioning for exercises):
1. /dev/sda VagrantRoot
2. /dev/sdb ubuntu cloud fs
3. /dev/sdc 1000MB will be for partition practicing and other non-persistent across boot stuff
4. /dev/sdd 50MB normal_swap
5. /dev/sde 50MB encrypted_swap
6. /dev/sdf 50MB encrypted_fs
7. /dev/sdg 50MB ext4 /tmp
8. /dev/sdh 50MB ext4 /quota 
9. /dev/sdi 50MB mdadm RAID
10. /dev/sdj 50MB mdadm RAID
11. /dev/sdk 50MB mdadm RAID
12. /dev/sdl 50MB mdadm RAID
13. /dev/sdm 50MB LVM
14. /dev/sdn 50MB LVM
15. /dev/sdo 50MB LVM

### Initial setup without vagrant
```
sudo apt -y update; sudo apt -y upgrade ; echo 'datasource_list: [ None ]' | sudo -s tee /etc/cloud/cloud.cfg.d/90_dpkg.cfg; sudo apt -y purge cloud-init; sudo rm -rf /etc/cloud/; sudo rm -rf /var/lib/cloud/; sudo apt -y install tmux 
```
We are going to setup a group of computers all running Ubuntu 16.04 Server within a NAT Network of 10.20.30.0/24 with primary machine being 10.20.30.40 and other clients being .50 and up not to exceed .99. Static IPs will be used to make consistent for these examples to work especially for networking so consider starting there.  Hostnames will be lbXX with XX being the last octet such as 10.20.30.40 being lb40 and the domain name search will be lfcs.local.

## Caching Updates and Packages
As you will be running more than one identical Ubuntu box; it makes some sense to maintain local LAN copies of packages you are going to be using to speed up installation.  There is no one size fits all; merely different ways of approaching the problem.
Additional resources: [Cache APT packages with Squid proxy](http://www.rushiagr.com/blog/2015/06/05/cache-apt-packages-with-squid-proxy/)
### Squid
One of the things we are practicing anyways is setting up a squid caching server.  So, this is the ideal approach to make updates apply faster locally off your LAN using lb40 as the caching server and can be up and running quickly (apt-mirror requires you to download the entire mirror which can take a long time).  Before doing anything else, setup and optimize squid on lb40 as a recommendation.  Although it was tempting to force all vms to use this feature, the clients should be added manually after lb40 is working.  Set Debian package files to a longer life than normal (perhaps several months).  
#### Squid Server
```
http_port 8080
refresh_pattern -i \.(tar.gz|tar|deb|rpm|bz2|gz|xml)$ 129600 90% 129600 override-expire ignore-no-cache ignore-no-store ignore-private refresh-ims
```
#### Squid Clients
```
# cat /etc/apt/apt.conf.d/01proxy
Acquire::http::Proxy "http://10.20.30.40:8080/";
Acquire::https::Proxy "http://10.20.30.40:8080/";
```
The first download will be at WAN speed; but, subsequent downloads should be much faster. 
More tweaks can be found at https://wiki.ubuntu.com/SquidDebProxy
### apt-mirror
An alternative idea would be a local apt-mirror repository which will never slow you down in that all files will be local; but, it will take a significant amount of time and space to populate your repository.  This will take in the ballpark of 129.0 GB heavily depending on configuration (use deb-amd64 to not also download i386).
### squid-deb-proxy
This is the easiest solution, and easiest to maintain and install, it requires no configuration on the client side, and almost or no configuration on the server side.  However, you really aren't going to learn squid this way so doing this is to be discouraged.
_Install on the server_
```
sudo apt-get install squid-deb-proxy squid-deb-proxy-client;  sudo start squid-deb-proxy
```
_Install on clients_
```
sudo apt-get install squid-deb-proxy-client
```
### apt-cacher-ng
Another idea would be to use apt-cacher-ng which will be slow on the first download of a file; but, like squid will keep a copy around for all future usage.  In contrast to apt-mirror, when clients are requesting for a package, apt-cacher checks if it has it cached, if yes – the package is served, if no – apt-cacher-ng fetches it from repositories, serves it to the client, and caches it [for other clients].  The biggest concern about this approach is people report it isn't stable.  Squid seems to be a better option.

## Files to support Vagrant
Install vagrant and put the supplied Vagrantfile and standardize.txt file in the same folder.  Bring up the VMs (after they finish downloading) with vagrant up.   
As we are using vagrant, the default username and password to accomplish tasks for these activities is vagrant/vagrant.  

## Vagrantfile
```
Vagrant.configure("2") do |config|
 config.vm.provision :shell, path: "standardize.txt", run: 'always'
 config.vm.define "lb40" do |lb40|
    lb40.vm.box = "lb40"
	lb40.vm.network "private_network", ip: "10.20.30.40", virtualbox__intnet: true, :adapter => 2
    lb40.vm.network "forwarded_port", guest: 22, host: 55540
	lb40.vm.hostname = "lb40"
	lb40.vm.box = "ubuntu/xenial64"
	lb40.vm.provider :virtualbox do |lb40|
        lb40.name = "lb40"
		lb40.memory = "4000"
		# 3 drive
		unless File.exist?('./3Disk.vdi')
            lb40.customize ['createhd', '--filename', './3Disk.vdi', '--variant', 'Fixed', '--size', 1 * 1024]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 2, '--device', 0, '--type', 'hdd', '--medium', './3Disk.vdi']
        # 4 drive
		unless File.exist?('./4Disk.vdi')
             lb40.customize ['createhd', '--filename', './4Disk.vdi', '--variant', 'Fixed', '--size', 50]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 3, '--device', 0, '--type', 'hdd', '--medium', './4Disk.vdi']
		# 5 drive
		unless File.exist?('./5Disk.vdi')
             lb40.customize ['createhd', '--filename', './5Disk.vdi', '--variant', 'Fixed', '--size', 50]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 4, '--device', 0, '--type', 'hdd', '--medium', './5Disk.vdi']
		# 6 drive
		unless File.exist?('./6Disk.vdi')
             lb40.customize ['createhd', '--filename', './6Disk.vdi', '--variant', 'Fixed', '--size', 50]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 5, '--device', 0, '--type', 'hdd', '--medium', './6Disk.vdi']
		# 7 drive
		unless File.exist?('./7Disk.vdi')
             lb40.customize ['createhd', '--filename', './7Disk.vdi', '--variant', 'Fixed', '--size', 50]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 6, '--device', 0, '--type', 'hdd', '--medium', './7Disk.vdi']
		# drive 8
		unless File.exist?('./8Disk.vdi')
             lb40.customize ['createhd', '--filename', './8Disk.vdi', '--variant', 'Fixed', '--size', 50]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 7, '--device', 0, '--type', 'hdd', '--medium', './8Disk.vdi']
		# drive 9
		unless File.exist?('./9Disk.vdi')
             lb40.customize ['createhd', '--filename', './9Disk.vdi', '--variant', 'Fixed', '--size', 50]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 8, '--device', 0, '--type', 'hdd', '--medium', './9Disk.vdi']
		# drive 10
		unless File.exist?('./10Disk.vdi')
             lb40.customize ['createhd', '--filename', './10Disk.vdi', '--variant', 'Fixed', '--size', 50]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 9, '--device', 0, '--type', 'hdd', '--medium', './10Disk.vdi']
		# drive 11
		unless File.exist?('./11Disk.vdi')
             lb40.customize ['createhd', '--filename', './11Disk.vdi', '--variant', 'Fixed', '--size', 50]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 10, '--device', 0, '--type', 'hdd', '--medium', './11Disk.vdi']
		# drive 12
		unless File.exist?('./12Disk.vdi')
             lb40.customize ['createhd', '--filename', './12Disk.vdi', '--variant', 'Fixed', '--size', 50]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 11, '--device', 0, '--type', 'hdd', '--medium', './12Disk.vdi']
		# drive 13
		unless File.exist?('./13Disk.vdi')
             lb40.customize ['createhd', '--filename', './13Disk.vdi', '--variant', 'Fixed', '--size', 50]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 12, '--device', 0, '--type', 'hdd', '--medium', './13Disk.vdi']
		# drive 14
		unless File.exist?('./14Disk.vdi')
             lb40.customize ['createhd', '--filename', './14Disk.vdi', '--variant', 'Fixed', '--size', 50]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 13, '--device', 0, '--type', 'hdd', '--medium', './14Disk.vdi']
	end
 end
 config.vm.define "lb50" do |lb50|
    lb50.vm.box = "lb50"
    lb50.vm.network "forwarded_port", guest: 22, host: 55550
	lb50.vm.network "private_network", ip: "10.20.30.50", virtualbox__intnet: true, :adapter => 2
	lb50.vm.hostname = "lb50"
	lb50.vm.box = "ubuntu/xenial64"
	lb50.vm.provider :virtualbox do |lb50|
        lb50.name = "lb50"
	end
 end
  config.vm.define "lb60" do |lb60|
    lb60.vm.box = "lb60"
    lb60.vm.network "forwarded_port", guest: 22, host: 55560
	lb60.vm.network "private_network", ip: "10.20.30.60", virtualbox__intnet: true, :adapter => 2
	lb60.vm.hostname = "lb60"
	lb60.vm.box = "ubuntu/xenial64"
	lb60.vm.provider :virtualbox do |lb60|
        lb60.name = "lb60"
	end
 end
end
```
## standardize.txt file
```
# file is to be in same folder as Vagrantfile and named standardize.txt
# It will allow for getting systems into a standardize state 
# such as booting without Ubuntu's cloud-init
# enable vagrant ssh remotely with passwordsed
sed -i 's/ChallengeResponseAuthentication no/ChallengeResponseAuthentication yes/g' /etc/ssh/sshd_config  
service ssh restart
# disable cloud for a faster bootpin
sudo apt-get -y purge cloud-init
sudo mv /etc/cloud/ ~/; sudo mv /var/lib/cloud/ ~/cloud-lib
sudo apt remove -y open-iscsi
mkdir /tmp/{1..9}.{1..9}
apt-get -y install --install-recommends linux-generic-hwe-16.04
useradd -m fred
useradd -m sally
useradd -m george
sudo apt-get install --install-recommends linux-generic-hwe-16.04
```
## tmux and fish and apropos and documentation
Before doing any work within a vm at the command line, the first thing which should always be run is tmux (or screen if you are so inclined).  This will allow you to look up documentation without opening a new session or stopping your existing command.  Some vendors run tmux as the first thing they do when they log into our machines on shared sessions.  The most basic thing to do is open two terminals after tmux is loaded is to use `CTRL-B` then `" `.  To switch between them, use the `CTRL-B` then `up` or `down` arrow key.  There is a lot more a person could learn about tmux; but, this is enough to get several terminals up and running as well as navigate quickly between them.

fish can also help to avoid having to remember command syntax.

If you just can't remember how to do something, try apropos "idea" to see what the man pages say.

### What to do when network is down to box or proxy servers block you: put the Linux documentation to work for you:
Way too often the Internet isn't reachable because you have messed something up or the company proxy servers are just painful to detail with; so, how do you find information when you can't reach Internet or this guide?

Extract all the compressed gzip files in /usr/share
```
# find /usr/share -name *.gz -type f -exec ls -l {} \; 
# find /usr/share -name *.gz -type f -exec gunzip -k {} \;
# grep "route add.*" -R /usr/share/
/usr/share/man/fr/man8/route.8:.B route add \-net 192.57.66.0 netmask 255.255.255.0 gw ipx4
```
Additional information: [tmux for noobs](https://eoinoc.net/tmux-for-noobs)
## Syntax in this guide
All shell commands will be proceeded by a # indicating root account in use or $ indicating another account is in use that isn't the root account.  Sometimes the username will be displayed first as in this case the account fred:
```
fred $ ping -c 1 8.8.8.8
ping -c 1 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=118 time=84.6 ms

--- 8.8.8.8 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 84.683/84.683/84.683/0.000 ms
fred $
```
If a specific box is being used, the prompt may also have the hostname prepended:
```
lb60 # 
```
This maybe inconsistent in earlier incarnations of the guide; but, basically the example solutions should indicate enough information to understand what to do if you get stumped without being a cut and paste mechanism.
# Basic Linux Commands
https://www.cheatography.com/nhatlong0605/cheat-sheets/lfcs-module1-essentialcommand/
## Login to Both Local and Remote Text Consoles
You may not have a GUI to use when first logging into Ubuntu on a VM or physical box and may be presented with a black screen with white text.  This is known as the terminal mode or “multi-user” mode and is desirable for servers.  There are multiple consoles accessible using the key combination of ALT+F# with the number being a specific console (example: ALT-F2 switches to access console 2 called tty2).  Enter your username and password to login.  Sometime a specific tty is dedicated to logging and this tty won’t allow you to login.

Above all else, get used to using the CLI and not the Linux GUI (Xwindows).  GUI is never to be used on production servers so learn vi and not gnotepad+.

#### TASK: System is set to a graphical environment.  Set to multi-user only on Ubuntu.
```
# systemctl get-default
graphical.target
# systemctl enable multi-user.target
# systemctl start multi-user.target
# systemctl set-default multi-user.target
```
## Finding Files
To begin this exercise; make sure you are in /tmp/1.2 folder which should have been created during setup.
```
cd /tmp/1.2
```
### Find binary path
Figure out where the program 'ping' is stored:
```
# which ping
/bin/ping
```
### Find by name
To find a specific file named file in /tmp/1.2/ after creating it.
```
$ touch file
$ find /tmp/1.2 -name file -print
/tmp/1.2/file
```
### Find with permissions
Find all files in etc which are read only for all users:
```
# find /etc/ -perm 444 -exec ls -al {} \;
-r--r--r-- 1 root root 1504 Nov  9 08:30 /etc/apt/apt.conf.d/01autoremove-kernels
-r--r--r-- 1 root root 33 Nov  9 22:24 /etc/machine-id
```
### Find owner
To find a file with ownership of fred in /tmp/1.2 after creating it:
```
# touch freds
# chown fred freds
# find . -user fred -print
./freds/
```
### Find -iname
 -iname pattern
Like -name, but the match is case insensitive.  For example, the  patterns `fo*'  and  `F??'  match  the  file names `Foo', `FOO', `foo', `fOo', etc. The pattern `*foo*` will also match a file called '.foobar'.
### Find type
To find directories: `find . -type d -print`
```
-type c
File is of type c:
b      block (buffered) special
c      character (unbuffered) special
d      directory
p      named pipe (FIFO)
f      regular file
l      symbolic link; this is never true if the -L option or  the  -follow
s      socket
```
### Find -exec
In etc, find all files and then output their type to /tmp/1.2/etctypes and take a look at the top of the resulting file:
```
# find /etc -type f -exec file '{}' > /tmp/1.2/etctypes \;
# head etctypes
/etc/overlayroot.local.conf: ASCII text
/etc/rpc: ASCII text
/etc/mysql/my.cnf.fallback: BSD makefile script, ASCII text
/etc/mysql/conf.d/mysqldump.cnf: ASCII text
/etc/mysql/conf.d/mysql.cnf: ASCII text
/etc/X11/Xsession.d/60xdg-user-dirs-update: ASCII text
```
### Find -ok
Add file deleteme in /tmp/1.2.  Find all files in /tmp/1.2 and ask if it is ok to delete them one by one but answer no to each delete other than deleteme: 
```
# touch deleteme
# find /tmp/1.2 -type f -ok rm {} \;
< rm ... ./1999 > ? n
< rm ... ./2018 > ? n
< rm ... ./etctypes > ? n
< rm ... ./deleteme > ? y
```
### Find size
Make 10 files which are sized 1k through 10k with such names (1k size is named 1k).  Find all files sized less than 6k.

### Find time
First, let inside /tmp/1.2/ make dummy file named 1999 with the modify time of January 1st, 1999 and a current file with all timestamps for today named 2018; then, find only the 1999 file using find command and the 2018 file using find.
```
# touch -t 199901010101 1999 -m 1999
# touch 2018
# stat 1999 | grep Modify
Modify: 1999-01-01 01:01:00.000000000 +0000
# find . -atime +6480 -print
./1999
# find . -mtime 0
.
./2018
```
### File path
Sometimes it is helpful to find out the full path for a file:
```
# readlink -f /tmp/1.2/freds
/tmp/1.2/freds
```
Extra credit commands that can help out in a jam:
```
# locate ping
# whatis ping
```
## Evaluate and Update Text Files
### touch
It can be very useful to create a blank file.  Touch allows for this:
Create files named file1-file9:
```
# touch file{1..9}
# ls
file1  file2  file3  file4  file5  file6  file7  file8  file9
```
Make a file named 1978 which was created that year: 
```
# touch -t 197801010101 1978
# ls -al 1978
-rw-r--r-- 1 root root 0 Jan  1  1978 1978
```
### textfiles: vi(m)
There are many text editors.  For the serious administrator, traditional vi or one of the many newer versions which are vi improved (vim) is the one to at least be able to use with some proficiency as it is installed on systems when others are not.  When you are using a system without nano and you have no Internet connection, a knowledge of vi is critical. Unfortunately, vi requires more than a passing commitment to make it worth the effort.  It will pay huge dividends in the long run to learn vi/vim; but, requires some commitment to master.

Additional resources: [How to Install and Use vi/vim as a Full Text Editor](https://www.tecmint.com/vi-editor-usage/)
### textfiles: nano
For someone just learning Linux, there is no shame in using nano.  Were vi and emacs are similar in power to a full word processor, nano functions as a simple text editor and provides you on the bottom the commands need to complete a task.
### textfiles: diff
#### TASK: Compare /etc/passwd and /etc/shadow: `diff /etc/passwd /etc/shadow`

#### TASK:  Make two files and compare them in /tmp/1.4/
```
# printf "hello\nworld\n" > /tmp/1.4/hello
# printf "world\n" > /tmp/1.4/world
# diff /tmp/1.4/world /tmp/1.4/hello
0a1
> hello
```
#### There are two directories named /tmp/1.4/A_dir and /tmp/1.4/B_dir which have different files and sometimes similar files.  Produce a report to /tmp/1.4/B of files only that exist only in /tmp/1.4/B_dir and not at all in /tmp/1.4/A_dir .
```
# mkdir /tmp/1.4/{A_dir,B_dir}; touch /tmp/1.4/A_dir/{3..25}.txt ; touch /tmp/1.4/B_dir/{5..27}.txt
# cd /tmp/1.4
# diff -q A_dir/ B_dir/ 
# diff -q A_dir/ B_dir/ | grep B_ > /tmp/1.4/B
# cat /tmp/1.4/B
Only in B_dir/: 26.txt
Only in B_dir/: 27.txt
``` 
### textfiles: patch
### textfiles: cut
Let’s see how we can just display the third and forth columns for the first line in /etc/passwd:
```
# head -1 /etc/passwd
root:x:0:0:root:/root:/bin/bash
# head -1 /etc/passwd | cut -d : -f 3,4
0:0
```
### textfiles: vimdiff
Vimdiff starts Vim on two (or three or four) files.  Each file gets its own window. The differences between the files are highlighted.  This is a nice way to inspect changes and to move changes from one version to another version of the same file.
### textfiles: pr
### textfiles: cat
### textfiles: tac
### textfiles: head
### textfiles: tail
### textfiles: tr
Capitalize everything in /etc/passwd and send to /tmp/1.4/CAPITALIZED.txt
```
cat /etc/passwd | tr '[:lower:]' '[:upper:]' > /tmp/1.4/CAPITALIZED.txt
```
### textfiles: sed
To capitalize all "a": `# sed 's/a/A/g' printer2.txt > printer3.txt`
Special character escaped with a backslash in sed: `# sed -e "s/'/\"/g" printer2.txt`
Print only user lab in /etc/passwd with sed: `# sed -n '/lab/p' /etc/passwd`
Don't study harder than you need to, /usr/share/doc/sed/README contains many examples.
### Binary files: ls -l, cmp, md5sum
### Identify kind of file type
```
# file /boot/vmlinuz-4.15.0-38-generic
/boot/vmlinuz-4.15.0-38-generic: Linux kernel x86 boot executable bzImage, version 4.15.0-38-generic (buildd@lcy01-amd64-023) #41~16.04.1-Ubuntu S, RO-rootFS, swap_dev 0x7, Normal VGA
# file /etc/passwd
/etc/passwd: ASCII text
ls -ld <filename>
```
See how first entry of "ls -l" shows file type:
```
# ls -al /etc/rc2.d/S06rc.local
lrwxrwxrwx 1 root root 18 Nov  8 13:12 /etc/rc2.d/S06rc.local -> ../init.d/rc.local
# ls -ld /etc/passwd
-rw-r--r-- 1 root root 2767 Nov  8 15:15 /etc/passwd
```
##	Redirecting shell and command output 
To get help: `man sh` or `man bash` and look for redirect.
This is as good of a time as any to introduce cat.  cat - concatenate files and print on the standard output. It dumps to standard output (stdout) input files.
Also, echo is useful to take text enclosed in quotes to either send to stdout or with redirect to a file.
Write output to file `# echo "hello" > file`
Append output to file `# command >> file`
Send output from one command to another (indefinitely stackable): `# cat file | grep A`
Write stdout and errors to two seperate files: `# command > out 2>error`
Write stdout and errors to same file: `# command &> out`
To read a file back into a command: `# wc -l < syslog.pdf`
##	Regular expression syntax
grep, grep -i, grep -v, grep -E '^root' /etc/passwd

.(dot) match a single character
a|z match a or z
$ match end of line
\* match 0 or more preceding items
\^ match start of line
find out how to look these up in man or /usr/share/doc
#### Task: Given that man man 

#### Find all the lines in /usr/share/common-licenses/GPL-3 that start with the word "The" but case insenstive and put result in /tmp/7.4/the
```
grep -i "^The " /usr/share/common-licenses/GPL-3 > /tmp/7.4/the
```
#### Find all the lines that end with the word "you" case sensitive and put result in /tmp/7.4/you
```
grep " you$" /usr/share/common-licenses/GPL-3 > /tmp/7.4/you
```
#### Find all the lines in /usr/share/common-licenses/GPL-3 that contain word "the" followed by the word "you"  and put result in /tmp/7.4/theyou
```
# grep -w "the.*you" /usr/share/common-licenses/GPL-3 > /tmp/7.4/theyou
# head -5 /tmp/7.4/theyou
them if you wish), that you receive source code or can get it if you
these rights or asking you to surrender the rights.  Therefore, you have
or can get the source code.  And you must show them these terms so they
(1) assert copyright on the software, and (2) offer you this License
of having them make modifications exclusively for you, or provide you
```

##	Archive or compress files and directories
Commands: gzip, bzip2, gunzip, bunzip2, tar, xz, zip, star or tar --selinux for SELinux, rsync to another machine with ssh

#### TASK: Place a gzipped archive in /tmp/3.2/ named "cron.tar.gz" which contains /etc/cron.d/ but without absolute paths
```
# cd /etc/cron.d/
# tar cfgv /tmp/3.2/cron.tar.gz *
```
#### TASK: Place a XZ compressed tar archive in /tmp/3.2/ named "ssh_config.tar.xz" which contains all of /etc/ssh but without folders from root.
```
# cd /etc/ssh/
# tar cfJ /tmp/3.2/ssh_config.tar.xz *
```
#### TASK: Extract  /usr/share/doc/apg/php.tar.gz into a new directory named /tmp/3.2/php
```
# mkdir -p /tmp/3.2/php
# tar xfvz /usr/share/doc/apg/php.tar.gz -C /tmp/3.2/php
# ls /tmp/3.2/php
index.php  lang/  README  themes/
```
Additional resources: [tecmint - Archiving Files/Directories](https://www.tecmint.com/sed-command-to-create-edit-and-manipulate-files-in-linux/)
##	Affecting directories and files with mv, cp, and others
Create directories dir1-dir8 in /tmp/1.8
```
mkdir /tmp/1.8/dir{1..8}
# ls /tmp/1.8/
dir1  dir2  dir3  dir4  dir5  dir6  dir7  dir8
```
Create dummy files file1-99 in /tmp/3.1 and then delete file40 through file49
```
# touch /tmp/3.1/file{1..99}
# rm /tmp/3.1/file{40..49}
# ls /tmp/3.1/file{40..49}
# ls /tmp/3.1/file* # result truncated to avoid filling this space
```
Copy /etc/debian_version to /tmp/1.8/ 
```
# cp /etc/debian_version /tmp/1.8/
# ls /tmp/1.8/debi*
/tmp/1.8/debian_version
```
Make directories in /tmp/1.8/dirtest named 1,2,3,4, and 5 with one command; then delete them all with one command:
```
# mkdir -p /tmp/1.8/dirtest/{1..5}
# find /tmp/1.8/dirtest/ -type d -print
/tmp/1.8/dirtest/
/tmp/1.8/dirtest/4
/tmp/1.8/dirtest/2
/tmp/1.8/dirtest/3
/tmp/1.8/dirtest/1
/tmp/1.8/dirtest/5
# rm -rf /tmp/1.8/dirtest/
```
Ask for confirmation before deleting directory /tmp/1.8/ then cancel: `rm -Rfi /tmp/1.8/`
Make the following path: /tmp/1.8/4/5/1/3/2/1/4/1/3/2/1/4/5/6/4 
```
mkdir -p /tmp/1.8/4/5/1/3/2/1/4/1/3/2/1/4/5/6/4
```
Change your working directory (the directory you are currently in) to /tmp/1.8 and print working directory:
```
# cd /tmp/1.8
# pwd
/tmp/1.8
```
Other commands
```
echo 'text' > file
cat
touch
vi(m)
nano
mkdir
mkdir -p
rm
cp
mv
```
##	Shortcuts to files and directories with hard and soft links
#### TASK: Create a hard link from /etc/apg.conf to /tmp/2.4/apglink: 
```
ln /etc/apg.conf /tmp/2.4/apglink
```
#### TASK: Create a symbolic link between /etc/appstream.conf and /tmp/5.6/appstream.txt
```
# ln -s /etc/appstream.conf /tmp/5.6/appstream.txt
# ls -al /tmp/5.6/appstream.txt
lrwxrwxrwx 1 root root 19 Nov 10 01:47 /tmp/5.6/appstream.txt -> /etc/appstream.conf
```
#### TASK: Figure out where the symbolic link /etc/vtrgb points to and put the result in /tmp/3.6/vtrgb
```
# ls -al /etc/vtrgb > /tmp/3.6/vtrgb
# cat /tmp/3.6/vtrgb
lrwxrwxrwx 1 root root 19 Nov 10 01:47 /tmp/5.6/appstream.txt -> /etc/appstream.conf
```
#### TASK: /etc/ has a bunch of symbolic links.  Provide a report of them all in /tmp/3.6/etclinks.txt
```
# find /etc -type l  -exec ls -l {} \; > /tmp/3.6/etclinks.txt
# head /tmp/3.6/etclinks.txt
lrwxrwxrwx 1 root root 24 Nov 12 01:43 /etc/mysql/my.cnf -> /etc/alternatives/my.cnf
lrwxrwxrwx 1 root root 17 Nov 14 00:55 /etc/rc5.d/S02anacron -> ../init.d/anacron
lrwxrwxrwx 1 root root 17 Nov 15 20:42 /etc/rc5.d/S04dovecot -> ../init.d/dovecot
lrwxrwxrwx 1 root root 22 Nov 15 20:42 /etc/rc5.d/S04cpufrequtils -> ../init.d/cpufrequtils
```
##	Standard file permissions
Commands: chmod, chown, chgrp, umask
Create empty files in /tmp/8.2 with following names and permissions:
readonly r--r--r--, readwriteowner rw-------, and readwritegroup ---rw---- 
```
# cd /tmp/8.2
# touch readonlyowner readwriteowner readwritegroup
# chmod 600 readwriteowner
# chmod 400 readonlyowner
# chmod 060 readwritegroup
# ls -al
total 8
drwxr-xr-x  2 root root 4096 Nov 10 02:20 .
drwxrwxrwt 91 root root 4096 Nov 10 02:20 ..
-r--------  1 root root    0 Nov 10 02:20 readonlyowner
----rw----  1 root root    0 Nov 10 02:19 readwritegroup
-rw-------  1 root root    0 Nov 10 02:19 readwriteowner
```
Set fred's account to allow group members to read only any of his new files he creates for current session.
``` 
fred $ umask -p
0002
fred $ umask -S
u=rwx,g=rwx,o=rx
# umask 0037
# apt install -y pam_umask
```
Other stuff:
```
ls -l file.txt
chown o+w file.txt
chown 755 file; chown 644 file; chown 700 file, chown 600 file; chown -r
Create a directory named /private. Use an acl to only allow access (rwx) to tom to the private directory.
umask
```

Additional resources: [tecmint - File Permissions & Attributes](https://www.tecmint.com/manage-users-and-groups-in-linux/)
## Documentation on systems
```
# man ls #### Truncated but is simple system manual
# info mount #### the links at bottom work in info
# whatis -w mount*
mount (8)            - mount a filesystem
mount.fuse (8)       - format and options for the fuse file systems
mount.lowntfs-3g (8) - Third Generation Read/Write NTFS Driver
mount.ntfs (8)       - Third Generation Read/Write NTFS Driver
mount.ntfs-3g (8)    - Third Generation Read/Write NTFS Driver
mount_namespaces (7) - overview of Linux mount namespaces
mountpoint (1)       - see if a directory or file is a mountpoint
# whereis mount
mount: /bin/mount /sbin/mount.lowntfs-3g /sbin/mount.vmhgfs /sbin/mount.ntfs /sbin/mount.ntfs-3g /sbin/mount.fuse /usr/share/man/man8/mount.8.gz


# ls -l /usr/share/doc/gzip-1.5/
```
Many commands offer a --help or -h to get more information.
```
# tune2fs --help
tune2fs 1.42.13 (17-May-2015)
tune2fs: invalid option -- '-'
Usage: tune2fs [-c max_mounts_count] [-e errors_behavior] [-g group]
        [-i interval[d|m|w]] [-j] [-J journal_options] [-l]
        [-m reserved_blocks_percent] [-o [^]mount_options[,...]] [-p mmp_update_interval]
        [-r reserved_blocks_count] [-u user] [-C mount_count] [-L volume_label]
        [-M last_mounted_dir] [-O [^]feature[,...]]
        [-Q quota_options]
        [-E extended-option[,...]] [-T last_check_time] [-U UUID]
        [ -I new_inode_size ] device

# quotatool -h
  quotatool version 1.4.12
  Copyright (c) 1999-2012 Mike Glover / Johan Ekenberg
  Distributed under the GNU General Public License
  http://quotatool.ekenberg.se

Usage: quotatool -u uid | -g gid options [...] filesystem
       quotatool -u | -g -i | -b  -t time filesystem
Options:
  -b      : set block limits
  -i      : set inode limits
  -q n    : set soft limit to n blocks/inodes
  -l n    : set hard limit to n blocks/inodes
  -t time : set global grace period to time
  -r      : restart grace period for uid or gid
  -R      : raise-only, never lower quotas for uid/gid
  -d      : dump quota info in machine readable format (see manpage)
  -h      : show this help
  -v      : be verbose (twice or thrice for debugging)
  -V      : show version
  -n      : do nothing (useful with -v)
```

Additional resources: [tecmint]( https://www.tecmint.com/explore-linux-installed-help-documentation-and-tools/)

##	Root account and root-like access
Commands: su, sudo, 
Files: /etc/sudoers, /etc/sudoers.d/*
Documentation: /usr/share/doc/sudo/examples/sudoers
#### TASK: Change to root account
Change to root using password: `sudo -i`
#### TASK: Change root password: 
```
# passwd root
Enter new UNIX password:
Retype new UNIX password:
passwd: password updated successfully
```
#### TASK: Allow sally to run all the root commands without entering her password:
```
root # sudo su - sally
sally $ sudo fdisk -l /dev/sdj
[sudo] password for sally:
sally is not in the sudoers file.  This incident will be reported.
sally $ exit
root # less /usr/share/doc/sudo/examples/sudoers
root # echo "sally ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers.d/sally
sally $ sudo fdisk -l /dev/sdj
Disk /dev/sdj: 50 MiB, 52428800 bytes, 102400 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```
#### TASK: Allow fred to become sally during her vacation but no other user (ethical issues aside):
```
root # grep "joe" /usr/share/doc/sudo/examples/sudoers    # joe may su only to operator
joe             ALL = /bin/su operator 
root # echo "fred ALL = /bin/su sally" >> /etc/sudoers.d/fred
root # sudo su - fred
fred $ su - sally
Password:
sally $ whoami
sally
```
#### TASK: Allow fred to activate edit passwd file and change users passwords:
```
root # swapoff -a
root # free | grep -i swap
Swap:             0           0           0
root # sudo su - fred
fred $ /sbin/swapon -a
swapon: cannot open /dev/sdd: Permission denied
swapon: cannot open /dev/mapper/swap_encrypt: Permission denied
fred $ exit
root # echo "fred ALL = VIPW " >> /etc/sudoers.d/fred
root # echo "Cmnd_Alias      VIPW = /usr/sbin/vipw, /usr/bin/passwd" >> /etc/sudoers.d/fred
root # sudo su - fred
fred $ $ sudo passwd sally
Enter new UNIX password:
Retype new UNIX password:
passwd: password updated successfully
```
#### TASK: Recover root password if nobody knows the root password on a machine or a sudo account:
```
#### HOLD left shift DURING BIOS POST.
#### Select a kernel that says (recovery mode) 
### Wait until you get recovery menu, then select root
### At root prompt, remount root filesystem.
#### Check for rw
# mount | grep -w /
# mount -o remount,rw /
# passwd 
# reboot
```

#### TASK: Set walt, fred, and joe to all be part of a new group called "ops" and grant all ops users full control of system as if they were root
```
root # 
```
Additional resources: [tecmint - Enabling sudo Access on Accounts](https://www.tecmint.com/manage-users-and-groups-in-linux/)
#	Operation of Running Systems

##	Bootloader configuration
We are clearly talking about GRUB2 which is the boot loader.
https://help.ubuntu.com/community/Grub2 
/etc/grub.d/40_custom is the correct place to edit many things and sometimes /etc/default/grub.
Make any text file changes for grub settings and auto add additional found operating systems like Windows with:
```
# update-grub
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-4.15.0-29-generic
Found initrd image: /boot/initrd.img-4.15.0-29-generic
Found memtest86+ image: /memtest86+.elf
Found memtest86+ image: /memtest86+.bin
Done
$
```
Additional information: [tecmint](https://www.tecmint.com/configure-and-troubleshoot-grub-boot-loader-linux/), [Configure GRUB2 Boot Loader Settings In Ubuntu 16.04](https://www.ostechnix.com/configure-grub-2-boot-loader-settings-ubuntu-16-04/)

##	Rebooting and halting system properly
#### TASK: Just reboot immediately:
```
# reboot
```
#### TASK: Power off system (called halting it):
```
# shutdown -h now 
```
#### TASK: Shutdown in 5 minutes with explanation to users who are logged in using write all (wall):
```
# shutdown -r +5 "I need to apply new kernel, back in a few." 
```
##	Set system to boot at a different runlevel
Check current runlevel with below indicating multi-user (no Xwindows) and previous unknown: 
```
# runlevel
N 3
```
Find out names of runlevels: 
```
# info runlevel # result truncated
       Table 1. Mapping between runlevels and systemd targets
       ┌─────────┬───────────────────┐
       │Runlevel │ Target            │
       ├─────────┼───────────────────┤
       │2, 3, 4  │ multi-user.target │
       ├─────────┼───────────────────┤
       │5        │ graphical.target  │
       └─────────┴───────────────────┘
```
Enable GUI boot with one command (unlikely this is desired as focus is CLI): 
```
# systemctl {start,enable} graphical.target
```
Disable GUI boot with one command: 
```
# systemctl {start,enable} multi-user.target
```


##	System log files
The main system logs are located in /var/log/

/var/log/apache2 stores apache2 logs


#### TASK: Find all occurrences of sudo and su authentication and place a report in /tmp/4.5
```
# grep "su|sudo" /var/log/auth.log
```
/var/log/syslog is the kernel boot information

Make use of grep on these to find what you are looking for in previous content or use tail -f <logfile> to watch events as they happen.

##	Running commands at a scheduled date and time
Commands at, cron, crontab, sleep
Files: /etc/crontab; /etc/cron.*
How do you tell BASH to take a 1 minute break bfore doing something? `sleep 1m`
Use crontab tool to list and set up scheduled tasks.
Let’s take a look at current scheduled tasks:
```
crontab -l
```
Edit crontab:
```
update-alternatives --config editor # Great time to set text editor 
crontab -e
```
List a specific user’s scheduled jobs:
```
crontab -u lab -l
```
Run `df -h > /var/log/df.log` that daily at 10:15 checks disk space and writes it to /var/log/systemstatus.log
```
# apt install sysstat
# grep ENA /etc/default/sysstat
ENABLED="true"
# service sysstat restart
# crontab -l
15 22 * * * /usr/bin/sar -A > /var/log/systemstatus.log
```
Run command `df -h > /var/log/df.log` on at 21:50 every Friday.

Run command `ping -c 1 8.8.8.8 > /var/log/networkstatus.log` during work hours of 8 AM to 5 PM (8-13) at 30 minutes after the hour on weekdays only (M-F).


Additional Resources: [tecmint](https://www.tecmint.com/11-cron-scheduling-task-examples-in-linux/ )

##	Evaluating results of scheduled commands
cron will log output which can be useful to investigate how a job performed.  The logs will inform you as to the failure or success of your task.
##	Apply software patches from Linux vendor
Commands: dpkg, apt-get, apt-cache, aptitude
### Update packages from the network, a remote repository, or from the local file system
Install a specific DEB file given URL: `# apt-get -y install `
install web server: `apt install -y apache2`
dump the information for package apache to a file named /tmp/3.2/apacheinfo `apt-cache show apache2 > /tmp/3.2/apacheinfo`
Remove package named finger: `apt purge -y finger`
How many files are part of finger?
```
dpkg -L finger | wc -l
13
```
Copy all files that contain the word grub from /etc/grub.d into /tmp/4.7: 
```
cp `grep -il "grub" /etc/grub.d/*` .
for file in /etc/grub.d/*; do if grep -q "grub" "$file";then cp "$file"
```
Additional resources: [tecmint - Linux Package Management](https://www.tecmint.com/linux-package-management/), [tecmint - 25 apt-get Command Examples](https://www.tecmint.com/linux-package-management/)
##	File system fixing and memory testing
Verify the integrity of the main resources a Linux server needs using the ubiquitous fsck and memtest86+ tools.
### fsck
Commands: fsck
#### TASK: Fix your ext4 partition at /dev/sdg1 which has errors
```
fsck.ext4 -y /dev/sdg1
```

### memtest86+
Commands: memtest86+ in grub or boot CD
Simply access this using the grub menu item or boot CD.  It runs pretty much auto-magic.

### Produce and deliver reports on system use (processor, memory, disk, and network), outages, and user requests
##	Monitor Linux Processes Resource Usage
ps, top and htop
##	Set Kernel Runtime Parameters in Linux
First thing is to figure out what kernel parameters are of interest.  They are active in /proc/sys as files but you should think of this as a temperary file created at boot time.  While these files can be updated, they will not persist upon reboot. Here we will look at a few options related to MySQL optimization:
```
# sysctl -a # Too much text to include so truncated
# sysctl vm.swappiness
vm.swappiness = 1
# cat /proc/sys/vm/swappiness
1
```
Changing kernel runtime parameters temperarily until next boot such as decreasing swappiness for MySQL:
```
# echo 0 > /proc/sys/vm/swappiness
# cat /proc/sys/vm/swappiness # to verify
0
```
OR
```
# sysctrl -w vm.swappiness = 0
# sysctl vm.swappiness # to verify
0
```
Changing kernel runtime parameters which will remain in effect after reboot but you must apply it with 'sysctl -p' :
```
# vi /etc/sysctl.conf
vm.swappiness=0
sysctl -p
```
Additional resources: [tecmint](https://www.tecmint.com/change-modify-linux-kernel-runtime-parameters/)
##	Basic Shell Scripting
Commands: mktemp, touch, crontab, at, cron
Files: /etc/cron.d/, /etc/crontab
Create a very basic script to perform a series of commands in a set sequence.
Make a temporary file in /tmp/ and display name: `mktemp`
Make a temporary directory in /tmp/ and display name: `mktemp -d`

Create a bash script named “infiniteloop” in /tmp/6.7 that does the following:
1.  Continually writes the string "Hello, world!" to the folder /dev/null
2. Has no exit loop (that is, it will continue to run until told to stop    
3.  Give fred permissions to run infinite_loop
4.  Switch user to fred and have him run infiniteloop
5.  Switch back to root
6.  As root, locate the now running infiniteloop and forcefully kill it by PID
Firstly, man bash and then look for "Compound Commands" to avoid forcing yourself to memorize syntax.
Also, take a look at /usr/share/doc/util-linux/examples/getopt-parse.bash
```
# cat infiniteloop
while true; do echo "Hello, world!" > /dev/null ; done
# ps auxf | grep fred
# kill <PID>
```
Write a BASH script called 2MB.sh in the /tmp/6.3/ directory that creates 17 files of 1MB each using the fallocate command. The files should have the name of: file_N where N is a number from 1 to 17.
```
# SUM=1 ; 
> until [ $SUM = 18 ]  ;
> do
>        touch file_$SUM
>        fallocate -l 1M file_$SUM
>        ((SUM++));
> done
# ls -alh file* # to verify, output removed.
```
Simpler answer:
```
# for N in `seq 1 17`; do fallocate -l 1M file_$N; done
```

Additional resources: [tecmint - Learning Basic Shell Scripting](https://www.tecmint.com/sed-command-to-create-edit-and-manipulate-files-in-linux/)
##	Manage system services with systemd
Check specific service: 
```
$ service apache2 status
```
Check all services: 
```
# service --status-all
```
Check status on, disable, and stop ufw:
```
# systemctl status ufw
# systemctl stop ufw
# systemctl disable ufw
```
Additional resources: [tecmint - Managing System Startup Process and Services)](https://www.tecmint.com/linux-boot-process-and-manage-services/)

###	Configure a service to run every time after boot
Enable network service dns to start at boot time.
```
systemctl enable bind
```
Disable network service dns to start at boot time.
```
systemctl disable bind
```
Additional resources: [tecmint - Configuring Services Automatic Start on Boot](https://www.tecmint.com/installing-network-services-and-configuring-services-at-system-boot/)

###	Start, stop, and check the status of network services
Start network service dns for current session.
```
systemctl start bind
```
Start network service dns for current session.
```
systemctl stop bind
```
Check status of network service dns.
```
systemctl status bind
```


##	Individual processes
Commands: nice, renice, ps, kill, top, htop
Files: /etc/security/limits.conf
Getting information about running processes on a system 
```
top (1)              - display Linux processes 
htop (1)             - interactive process viewer
ps (1)               - report a snapshot of the current processes.
```
Managing running processes
```
kill (1)             - send a signal to a process
nice - run a program with modified scheduling priority 
renice (1)           - alter priority of running processes 
```
Find the NICE value of your web server on 40.  Change NICE value of web server to -15.  Confirm value has changed.
```
# ps auxf -u root | grep apache | head -1
root      1511  0.0  0.5 260352 21924 ?        Ss   Nov08   0:03 /usr/sbin/apache2 -k start
# renice -15 1511
1511 (process ID) old priority 0, new priority -15
```
Find all processes which are being run by user fred: `ps -u fred`
Display all processes: `ps -ef`
Display all processes with one line per thread: `ps -eLf`
Display all processes on a system in BSD like syntax: `ps aux`
Show the current resource limits for user: `ulimit -a`
Two commands to set a process priority: `nice` and `renice`
How do you find out which libraries a binary needs? `ldd <binary>`
Additional resources: [tecmint - Monitor Linux Processes Resource Usage}(https://www.tecmint.com/monitor-linux-processes-and-set-process-limits-per-user/)

## SELinux and AppArmor
Let's investigate the utilities view both file and process contexts for both SELinux and AppArmor.

Access control lists are enabled by setting the  ``acl`` directive in `/etc/fstab` file for desired filesystem.  But, you must remount the filesystem to activate them.

Syntax
Set:  `setfacl -m u:fred:rw /home/users/file`
Remove:  `setfacl -x u:fred /home/users/file`
Query:  `getfacl /home/users/file`
TASK 1:
- Create the directory /home/acl/fred
- Create a file in this directory called “fredsfile”
- Set an ACL such that fred will have access to read and write to the file
- Verify that fred cannot create new files in this directory
- Verify that fred can edit the file “fredsfile”
```
root # mkdir -p /home/acl/fred
root # touch /home/acl/fred/fredsfile
root # setfacl -m u:fred:rw /home/acl/fred/fredsfile
root # getfacl /home/acl/fred/fredsfile
root # sudo su - fred
fred $ cd /home/acl/
fred /home/acl/fred $ touch 1
touch: cannot touch '1': Permission denied
fred /home/acl/fred $ echo "1" > fredsfile
fred /home/acl/fred $ cat fredsfile
1
fred /home/acl/fred $ exit
```
TASK 2:
- Create the directory “/home/acl/group”
- Create the secondary group “acltest”
- Assign fred and sally to it
- Set the group ownership of “/home/acl/group” to “acltest”
- Set permissions to enable SGID and to lock-out non-group users
- Test and verify this using root account and test with all appropriate user accounts including walt
    * As fred, Create a file called “test” in the “/home/acl/group” directory
    * As fred, add the line “Fred was here.”
    * As sally, add the line “Sally was here.”
    * As walt, fail to use the file named "/home/acl/group/test".
    * As george, see both sally and fred's entries inside the file and clobber the file
```
# mkdir /home/acl/group
# addgroup acltest
Adding group `acltest' (GID 1005) ...
Done.
# grep acl /etc/group
acltest:x:1005:fred,sally
# setfacl -m g:acltest:wr /home/acl/group/
# getfacl /home/acl/group/
getfacl: Removing leading '/' from absolute path names
# file: home/acl/group/
# owner: root
# group: root
user::rwx
group::r-x
group:acltest:rw-
mask::rwx
other::r-x
# touch /home/acl/group/acltest
# setfacl -m g:acltest:rw /home/acl/group/acltest
```
To see more about a process or file with SE information:
```
* ls -Z
* ps -eZ
```
Task: Restore default SELinux file contexts for /etc/passwd
```
#
```

Additional resources: [tecmint - Implementing Mandatory Access Control](https://www.tecmint.com/mandatory-access-control-with-selinux-or-apparmor-linux/), [Ubuntu AppArmor Wiki](https://wiki.ubuntu.com/AppArmor)
##	Manage Software
On Ubuntu you can add software with `apt install -y <packagename>`
Find out which package provides ping6 and install: 
```
# apt-cache search ping6
iputils-ping - Tools to test the reachability of network hosts
oping - sends ICMP_ECHO requests to network hosts
thc-ipv6 - The Hacker Choice's IPv6 Attack Toolkit
# dpkg -S ping6
bash-completion: /usr/share/bash-completion/completions/ping6
iputils-ping: /bin/ping6
iputils-ping: /usr/share/man/man8/ping6.8.gz
# apt install -y iputils-ping
```
##	Identify the component of a Linux distribution that a file belongs to placing the package name only in the file /tmp/7.5/pingpackage
```
# dpkg -S /bin/ping | cut -d \:  -f 1 >  /tmp/7.5/pingpackage
# cat /tmp/7.5/pingpackage
iputils-ping
```
What is in a package:
```
# dpkg -L quota | head -3
/.
/sbin
/sbin/quotaon
```
Additional resources: [tecmint - 15 dpkg Command Examples](https://www.tecmint.com/dpkg-command-examples/)
#	User and Group Management
https://www.cheatography.com/nhatlong0605/cheat-sheets/lfcs-module2-userandgroupmanagement/
##	Local user accounts
We are going to want several accounts; if they haven't been created already from previous activity.  Make sure you have fred, sally, and walt.
```
# useradd -m sally
# useradd -m walt
# useradd -c "Fred Flintstone" -s /bin/bash -m fred
# passwd fred
# su fred
# sudo chage -l fred
# sudo usermod -g users fred
```
Set Fred's account to expire after 8 months, sally after one year after setting both passwords to t3sting.
```

```
Add account hal with UID of 4000, group named hal with GUID of 4000.
```

```

#### TASK: Add account nut with shell of tcsh, primary group of staff, password of abc123 but force user to rotate password upon login.
```
# useradd -g staff -s /usr/bin/tcsh nut
# passwd nut
Enter new UNIX password:
Retype new UNIX password:
passwd: password updated successfully
# passwd --expire nut 
#### Alternatively use 
# chage -E -1 nut
```
Give fred access to all of hal's home directory: 
```

```
## Local groups and group memberships
Commands: vi /etc/group, groups, groupadd
For future purposes, we will want the group named "limit":

For speed, just edit /etc/groups as the CLI tools are slow to use.
```
sudo groupadd limit
cat /etc/group | grep limit
groups command
addgroup 
groupadd
chmod
chgrp
```
Create a directory named /home/partners. Allow fred and hal to share documents in this directory using a group called partners. Both of them can read, write and remove documents from the other in this directory but any user not member of the group can’t.
```
# mkdir /home/partners
```
Set sally to use Korn Shell, fred to use CSH, walt to use tcsh, and george to use sh.

Additional resources: [tecmint - Managing Users & Groups](https://www.tecmint.com/sed-command-to-create-edit-and-manipulate-files-in-linux/)
##	System-wide profiles
##	New user templates
Adding, deleting, and modifying the files in /etc/skel for new users
##	Restricting resource limits for users and groups
/etc/security/limits.conf is the persistant file; ideally we would use /etc/security/limits.d/fred.conf but there is no documentation in there so the main file is easier to use.
We can check with ulimit -a 
#### TASK: Add a limit configuration file for fred limiting him to 5 user processes. Two separate sessions can be maintained to help; one for root and one for fred:
```
fred $ ulimit -u
max user processes (-u) 60201
root # -s
root # echo -e "fred\thard\tnproc\t5" >> /etc/security/limits.conf
fred $ grep fred /etc/security/limits.conf
fred        hard    nproc           5
```
Test this as fred after logging out and then back in again as fred:
```
fred $ exit
root # sudo su - fred
fred $ ulimit -u
5
fred $ echo “Don’t run the following as root, run as fred!\n”
fred $ :(){ :|:& };: # This will explode some machines!
[1] 18921
fred $ -su: fork: retry: Resource temporarily unavailable
-su: fork: retry: Resource temporarily unavailable
-su: fork: retry: No child processes
-su: fork: retry: No child processes
-su: fork: retry: Resource temporarily unavailable
-su: fork: retry: Resource temporarily unavailable
-su: fork: retry: No child processes
-su: fork: retry: No child processes
```
#### TASK: Make a group named "problems" in which new users "sid" and "ed" are placed and make it so all group members are restricted to 20 processes (both hard and soft limits) as they have been taxing the server recently.  However, since sid hasn't been as big a problem, allow sid use as many processes as needed:
```
# echo -e "@problems\t-\tnproc\t20" >> /etc/security/limits.conf
# echo -e "sid\t-\tnproc\tunlimited" >> /etc/security/limits.conf
# useradd sid -m ; useradd ed -m ; groupadd problems; usermod -g problems sid; usermod -g problems ed
# sudo su - sid
$ ulimit -a | grep proce
max user processes              (-u) unlimited
$ exit
# sudo su - ed
$ ulimit -a | grep proc
max user processes              (-u) 20
$ exit
``` 
Additional resources: [Forkbomb explained](https://www.cyberciti.biz/faq/understanding-bash-fork-bomb/),  [Set Process Limits on a Per-User Basis](https://www.tecmint.com/monitor-linux-processes-and-set-process-limits-per-user/)

##	Updating privileges for groups and users
Let's go ahead and add fred to the sudo user group so he can manage system:
```
# cat /etc/group | grep ^sudo
sudo:x:27:lab
# usermod -a -G sudo lab
# cat /etc/group | grep ^sudo
sudo:x:27:lab
# usermod -a -G sudo fred
# cat /etc/group | grep ^sudo
sudo:x:27:lab,fred
```
##	Configure PAM
You might be asked to use a PAM module, which you will have to determine, in order to achieve a specific goal.

https://www.tecmint.com/use-pam_tally2-to-lock-and-unlock-ssh-failed-login-attempts/ 
### Configure LDAP Server
To setup lb40 as an ldap server: ```# sudo apt -y install slapd ldap-utils```
Fill in admin password desired when prompted, in this case let's use t3sting.
Use the following to configure: ```sudo dpkg-reconfigure slapd```
Let's match up with our DNS settings:
* domain: **ubuntu.local**
* Organization name: **Ubuntu**
* Administrator password: **Same password set during installation.**
* Database backend:  **MDB**.
* Do you want the database to be removed when slapd is purged? **No.**
* Move old database? **Yes**
* Allow LDAPv2 protocol? **No**.

### Configuring the LDAP Clients

`/etc/ldap/ldap.conf`  is the configuration file for all OpenLDAP clients. Open this file.

sudo nano /etc/ldap/ldap.conf

We need to specify two parameters: the  **base DN**  and the  **URI**  of our OpenLDAP server. Copy and paste the following text at the end of the file. Replace  `your-domain`  and  `com`  as appropriate.

BASE     dc=ubuntu,dc=local
URI      ldap://10.20.30.40
### To have PAM use LDAP server:
To configure PAM within an application to use specifically LDAP:

Additional resources: [Ubuntu LDAP Server](https://www.linuxbabe.com/ubuntu/install-configure-openldap-server-ubuntu-16-04)

#	File System
https://www.cheatography.com/nhatlong0605/cheat-sheets/lfcs-module5-storagemanagement/
## Introduction to File System 
### Standard layout
Root mounted at /
Home mounted at /home
Temporary files go into /tmp

### Common Linux file systems
* Ext4 – 4th Extended FS: Very Large FS sizes, Individual file can be 16TB + Perf - Standard for many enterprises
* XFS – Developed by SGI: Individual file can be 8 EiB – Less performance - Default for RHEL7
* Ext3 – 3rd Extended FS: Large FS sizes, Individual file can be 2TB + Journaling
* btrfs – Pronounced “ButterFS”: Many ideals of ext4 & ZFS + Duplication
* Old file system support included for NTFS/FAT32/FAT16
### Investigating deployed file system
Commands: cat /etc/fstab, fdisk -l, df -Th, du, e2label /dev/sda1
```
# cat /etc/fstab
# fdisk -l /dev/sda
# df -Th
# du
# e2label /dev/sda1
```
#### Task: Make Partition 4 on /dev/sdc which starts at 400 MB and ends at 499 MB and is ext4.
parted --script /dev/sdc mktable gpt
parted -s /dev/sdc mkpart pri ext4 400 500 set 4 lvm on

### fdisk fundamentals
Blah, who wants to write this up.  Just know parted or fdisk, gdisk, or parted.

### File System Extended Attributes

Each filesystem can have various extended attributes which can be investigated via stat, chattr, lsattr

#### TASK: Make a file that is immutable and one that is mutable after creating both and then check both:
```
# touch /tmp/1.3/{immutable,mutable}
# chattr +i /tmp/1.3/immutable
# lsattr /tmp/1.3/*mutable
----i---------e--- /tmp/1.3/immutable
--------------e--- /tmp/1.3/mutable
# rm /tmp/1.3/immutable
rm: cannot remove '/tmp/1.3/immutable': Operation not permitted
```
#### TASK: Delete both immutable mutable created earlier
```
# chattr -i /tmp/1.3/immutable
# lsattr /tmp/1.3/*mutable
----i---------e--- /tmp/1.3/immutable
--------------e--- /tmp/1.3/mutable
# rm /tmp/1.3/{immutable,mutable}
# lsattr /tmp/1.3/*mutable
lsattr: No such file or directory while trying to stat /tmp/1.3/*mutable
```
Disable access time for a file such as would be desirable on an SSD drive and test:
```
# touch /tmp/1.3/{timenoatime,timeatime}
# chattr +A /tmp/1.3/timenoatime
# echo 1 > /tmp/1.3/timeatime
# echo 1 > /tmp/1.3/timenoatime
# stat /tmp/1.3/time* | egrep "time|Access: 20"
  File: '/tmp/1.3/timeatime'
Access: 2018-11-08 15:22:07.248033652 -0600
  File: '/tmp/1.3/timenoatime'
Access: 2018-11-08 15:18:12.279613170 -0600
```
##	List, create, delete, and modify physical storage partitions
```
fdisk
parted # parted can also be used if you are more proficient with it
```
#### TASK: Create two partitions sized 50MiB on /dev/sdc with partition 1 labeled one and 2 labeled two
```
# gdisk
n 
1
+50MiB
 
n 
2
+50MiB
c
1
one
c
2
two
w
```
#### TASK: Format 1 as ext2 and 2 as ext4 and mount 1 as /ext2 and 2 as /ext4
```
# mkfs.ext2 /dev/sdc1
# mkfs.ext4 -q /dev/sdc2
# mkdir /ext4; mkdir /ext2
# mount /dev/sdc1 /ext2
# mount /dev/sdc2 /ext4
```

Additional resources: [tecmint - Formatting Filesystems](https://www.tecmint.com/create-partitions-and-filesystems-in-linux/)
##	Manage and configure LVM storage
Commands: lvm, mkfs.*, mount/umount
Files: /etc/fstab
### Practice Environment
My preferred way of handling this is to use a virtual machine inside virtualbox and then add several 50 MB hard drives.  I am assuming that configuration for the following commands with vagrant file and that sdc, sdm, sdn, and sdo are available.	This avoids having to undo LVM to do RAID and also allows for doing swapspace on the LVM drives.
### lvm shell
Just get into lvm shell to make command lookup faster:
```
# lvm
lvm>
```

#### TASK: Create logical volume named "projects" of 20 MB ext4 and set it be persistent after reboot at /projects:
```
lvm> lvcreate -n projects -L 20M VG
  Rounding up size to full physical extent 20.00 MiB
  Logical volume "projects" created.
lvm> lvdisplay /dev/VG/projects
  --- Logical volume ---
  LV Path                /dev/VG/projects
  LV Name                projects
  VG Name                VG
  LV UUID                psfwcN-LF1y-qX3D-YPqI-6PuR-Tuqi-OjdQDI
  LV Write Access        read/write
  LV Creation host, time lb40, 2018-11-16 19:39:30 +0000
  LV Status              available
  # open                 0
  LV Size                20.00 MiB
  Current LE             2
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:2
# mkfs.ext4 -q /dev/mapper/VG-projects
# mkdir /projects
# # blkid | grep /dev/mapper/VG-projects -w
/dev/mapper/VG-projects: UUID="51ec3153-272f-45c1-843c-e6d9099e3c69" TYPE="ext4"
#### Add to fstab and test with mount using directory which will test persistancy 
```
#### TASK: Create logical volume named "backups" of 5 MB mirrored ext2 across two drives and set it be persistent after reboot at /backups:
```
lvm> lvcreate --type raid1 -n backups -L 5M VG
  Rounding up size to full physical extent 8.00 MiB
  Logical volume "backups" created.
lvm> lvdisplay /dev/VG/backups
  --- Logical volume ---
  LV Path                /dev/VG/backups
  LV Name                backups
  VG Name                VG
  LV UUID                pHRwNf-6s30-FQIv-gT95-yf7l-BovG-Zw8mNi
  LV Write Access        read/write
  LV Creation host, time lb40, 2018-11-16 19:41:25 +0000
  LV Status              available
  # open                 0
  LV Size                8.00 MiB
  Current LE             2
  Mirrored volumes       2
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:7
# mkfs.ext2 -qq /dev/mapper/VG-backups
# mkdir /backups
# blkid | grep /dev/mapper/VG-backups -w
/dev/mapper/VG-backups: UUID="93918f45-77d7-4fc0-bf26-9a8256d9daa7" TYPE="ext2"
#### Add to fstab and test with mount
```
#### TASK: Make backups larger by 5 MB 
```
lvm> lvextend -L +5M /dev/mapper/VG-backups
  Rounding size to boundary between physical extents: 8.00 MiB
  Extending 2 mirror images.
  Size of logical volume VG/backups changed from 8.00 MiB (2 extents) to 16.00 MiB (4 extents).
  Logical volume backups successfully resized.
```

#### TASK: Add /dev/sdo drive into VG, then mirror to three drives instead of two
```
lvm> lvconvert -m2 VG/backups
lvm> lvdisplay VG/backups
  --- Logical volume ---
  LV Path                /dev/VG/backups
  LV Name                backups
  VG Name                VG
  LV UUID                pHRwNf-6s30-FQIv-gT95-yf7l-BovG-Zw8mNi
  LV Write Access        read/write
  LV Creation host, time lb40, 2018-11-16 19:41:25 +0000
  LV Status              available
  # open                 0
  LV Size                4.00 MiB
  Current LE             1
  Mirrored volumes       3
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:7
```
#### Task: Create 20 MB swap stripped across two drives withing lvm but don't active it yet:
```
lvm> lvcreate --stripes 2 -L 20M -n swap_lvm VG
  Using default stripesize 64.00 KiB.
  Logical volume "swap_lvm" created.

lvm> lvdisplay VG/swap_lvm -m
    --- Segments ---
  --- Segments ---
  Logical extents 0 to 79:
    Type                striped
    Stripes             2
    Stripe size         64.00 KiB
    Stripe 0:
      Physical volume   /dev/sdn
      Physical extents  121 to 160
    Stripe 1:
      Physical volume   /dev/sdo
      Physical extents  41 to 80
```
#### TASK: Shrink swap to 10 MB, format it, then set it to mount upon boot
```
lvm> lvreduce -L 10M VG/swap_lvm
  WARNING: Reducing active logical volume to 10.00 MiB
  THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce swap_lvm? [y/n]: y
  Size of logical volume VG/swap_lvm changed from 20.00 MiB (80 extents) to 10.00 MiB (40 extents).
  Logical volume swap_lvm successfully resized.
# mkswap /dev/mapper/VG-swap_lvm
Setting up swapspace version 1, size = 10 MiB (10481664 bytes)
no label, UUID=7e2f501c-37a1-4358-af23-3d5082bf9e47

```

Additional resources: [tecmint](https://www.tecmint.com/manage-and-create-lvm-parition-using-vgcreate-lvcreate-and-lvextend/)
##	Create and configure encrypted storage
Commands: cryptsetup, mount, umount
Files: **/usr/share/doc/cryptsetup/{README.Debian/FAQ}**
#### Task: Setup /dev/sde as an encrypted 50 MB swap partition active after every boot
Make a swap partition that fills the /dev/sde which is 50MB with label of swap_encrypted and passphrase of "swap" after destroying all partitions on the drive.  Make sure it comes up after boot.
Make sure you open the document file and use it as your reference both in practicing as well as production.  Look for word urandom
```
# swapoff -a
# free | grep -i swap
Swap:             0           0           0
# wipefs -a /dev/sde
# cryptsetup luksFormat /dev/sde1
# cryptsetup luksOpen /dev/sde1 swap_encrypt
# mkswap /dev/mapper/swap_encrypt
Setting up swapspace version 1, size = 48 MiB (50290688 bytes)
no label, UUID=528e3f7e-8174-4adb-98a7-2276efde32b9
# swapon /dev/mapper/swap_encrypt
# free | grep -i swap
Swap:         49112           0       49112
# grep swap /etc/fstab 
/dev/mapper/swap_encrypted       none            swap    sw                              0 0
# grep swap /etc/crypttab
swap_encrypted               /dev/sde1         /dev/urandom swap
```
Now remove all the work you just did by destroying partition table and removing the mount on reboot:
```
# swapoff -a
# free | grep -i swap
Swap:             0           0           0
# dd if=/dev/zero of=/dev/sde bs=1024 count=2 
# grep -H swap_encrypt /etc/{fstab,crypttab}
#
```
#### Task: Encrypt a ext4 file system on /dev/sdf 
For all of /dev/sdf make file system that is ext4. Make it active on boot under mount point of /crypt.  No password should be required during boot.  The passphrase of "passphrase" shall be used and a decryption key of /root/luks.key such that on every bootup the partition gets mounted to /crypt on lb40 with no prompt for passphrase.
```
# umount /crypt
# wipefs -a /dev/sdf
# parted /dev/sdf --script -- mklabel gpt
# parted /dev/sdf --script mkpart primary ext4 0% 100%
# parted /dev/sdf --script name 1 crypt
# dd if=/dev/random of=/root/luks.key bs=1024 count=2
# apt install cryptsetup
# lsmod dm_crypt
# modprobe dm_crypt
# lsmod | grep dm_crypt
dm_crypt               40960  0
# cryptsetup -y luksFormat -d /root/luks.key /dev/sdf1
WARNING!
========
This will overwrite data on /dev/sdf1 irrevocably.
Are you sure? (Type uppercase yes): YES
# cryptsetup --verbose luksOpen -d /root/luks.key /dev/sdf1  crypt
Enter passphrase for /dev/mapper/VG-crypt:
Key slot 0 unlocked.
Command successful.
# mkfs.ext4 /dev/mapper/crypt
# mkdir /crypt
# mount -v /dev/mapper/crypt /crypt
# cat /etc/fstab
/dev/mapper/crypt       /crypt  ext4    defaults        0       1
# cat /etc/crypttab
crypt   /dev/sdf1       /root/luks.key  none
```
Additional Resources: [tecmint - Disk Encryption](https://www.tecmint.com/disk-encryption-in-linux/), [chousensha - encryption](http://chousensha.github.io/blog/2018/02/12/lfcs-prep-managing-encrypted-partitions/)
##	Configure systems to mount file systems at or during boot
Look at system as it currently stands
```
cat /etc/fstab
fdisk -l /dev/sda
mount -t type device dir
umount dir
mkfs.____ or mkfs -t ____	
```
Create a new ext4 partition 10 MB in size on lb40 using one of the spare drives and mount it every boot using UUID to /mnt/10MBext4.
```
# fdisk /dev/sdc
Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-2097151, default 2048):
Last sector, +sectors or +size{K,M,G,T,P} (2048-2097151, default 2097151): +10M

Created a new partition 1 of type 'Linux' and of size 10 MiB.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
# mkfs.ext4 /dev/sdc1
# blkid | grep sdc1
/dev/sdc1: UUID="f0f91b53-f542-4ab6-92e8-064b63211b8e" TYPE="ext4" PARTUUID="2fa99c97-01"
# grep UUID /etc/fstab
UUID=f0f91b53-f542-4ab6-92e8-064b63211b8e       /mnt/10MBext4   ext4    defaults        0 2
# mkdir /mnt/10MBext4
# mount /mnt/10MBext4
# df -h /mnt/10MBext4
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdc1       8.7M   92K  7.9M   2% /mnt/10MBext4
```
Create a 1 MB ext4 partition that doesn't mount at bootup, copy /etc/group to it,  then resize it to 2 MB mounting it temporarily at /mnt/2MB. (In this case, we are using partition /dev/sdc1 but any valid location will do.  It is just easier to illustrate with an empty drive).
```
# fdisk /dev/sdc # details removed herein
# partprobe
# mkfs.ext4 /dev/sdc1
# mount /dev/sdc1 /mnt/2MB/
# cp /etc/group /mnt/2MB
# umount /mnt/2MB
# parted
(parted) resizepart 1 3MiB # syntax will vary heavily depending on start point
# ls -al /mnt/2MB/group # this verifies we didn't lose data
-rw-r--r-- 1 root root 1048 Nov 13 20:58 /mnt/2MB/group
```
Create a new ext3 partition 20 MB in size on lb40 using one of the spare drives using lvm and mount it every boot using /dev/mapper syntax to /mnt/20MBext3.

Do a filesystem check on any unmounted partition you have created:  `fsck /dev/sdc1`
Force filesystem check next mount for root: `touch /forcefsck` which is discussed and documented in /etc/init.d/checkfs.sh
Disable filesystem check for a partition: `touch /mnt/1MB/fastboot` documented in /etc/init.d/checkfs.sh

##	Configure and manage swap space
Commands: mkswap, swapon, swapoff
Files: /etc/fstab
#### Practice: Setting up a swap partition for activation after boot
Make a swap partition that fills the /dev/sdd which is 50MB with label of normal_swap after destroying all partitions on the drive.
```
# free | grep -i swap
Swap:             0           0           0
# parted /dev/sdd --script -- mklabel gpt
# parted /dev/sdd --script mkpart primary Linux-Swap 0% 100%
# parted /dev/sdd --script name 1 normal_swap
# mkswap /dev/sdd 
# swapon /dev/sdd
# free | grep -i swap
Swap:         51196           0       51196
```
Now set it up to work after boot:
```
# grep swap /etc/fstab 
/dev/sdd       none            swap    sw                              0 0
```
Now remove all the work you just did by destroying partition table and removing the mount on reboot:
```
# swapoff /dev/sdd
# free | grep -i swap
Swap:             0           0           0
# dd if=/dev/zero of=/dev/sdd bs=1024 count=2 
# grep -w /dev/sdd /etc/fstab
#
```
#### Practice: Make an LVM swap partition but don't set it up to work after boot
Create a new swap partition 10 MB in size stripping on lb40 using several LVM partitions and make active for current session but do not configure for after reboot to be active.  
```
lvcreate --size 1G --name lv_swap vg
mkswap /dev/vg/lv_swap
swapon /dev/vg/lv_swap
swapon -s
# grep lv_swap fstab
/dev/mapper/vg-lv_swap swap swap defaults 0 0
```
Make a simple 40MB swap partition somewhere and temporarily activate it:
```
# 
``` 
Make a 3 MB swap file at /tmp/3MBswap and temporarily use it.
```
#### pick between dd and fallocate
# fallocate  -l 3M /swap
# dd if=/dev/zero of=/swap bs=1024 count=3 
# mkswap /swap
Setting up swapspace version 1, size = 3 MiB (3141632 bytes)
no label, UUID=f0d9560a-47b0-4781-bb7d-4b1998be62ed
# free
Swap:             0           0           0
# swapon /swap
swapon: /swap: insecure permissions 0644, 0600 suggested.
# chmod 0600 /swap
# swapon /swap
swapon: /swap: swapon failed: Device or resource busy
# free | grep swap
Swap:          3068           0        3068
```
Additional information: [tecmint - Configuring Swap Partition](https://www.tecmint.com/create-partitions-and-filesystems-in-linux/)
##	Create and manage RAID devices
Previous competency LFCS 2.16: Assemble partitions as RAID devices
#### Reminder of layout of available drives
9. /dev/sdi 50MB mdadm RAID
10. /dev/sdj 50MB mdadm RAID
11. /dev/sdk 50MB mdadm RAID
12. /dev/sdl 50MB mdadm RAID

For starters, don't try to memorize all the options.  Start with the information provided by: 
```
/usr/share/doc/mdadm/README.recipes
/usr/share/doc/mdadm/examples/mdadm.conf-example 
```
 

#### TASK: Make a striped RAID array on two hard drives, save configuration.
```
# mdadm --create /dev/md0 -n 2 -l raid1 /dev/sdi /dev/sdj
# mdadm --detail /dev/md0
# mkfs.ext4 /dev/md0 
# mdadm --detail /dev/md0
# mdadm --stop /dev/md0
# mdadm --zero-superblock /dev/sdi /dev/sdj /dev/sdk /dev/sdl
```
#### TASK: Make RAID5 on three drives (/dev/sdi /dev/sdj /dev/sdk) as /dev/md0 and format as ext4
```
# mdadm --create /dev/md0 -n 3 -l raid5 /dev/sdi /dev/sdj /dev/sdk
```
#### TASK: Add fourth drive as spare /dev/sdl
```
# mdadm --add /dev/md0 /dev/sdl
```

#### TASK: Fail drive /dev/sdi then remove
```
# mdadm --fail /dev/md0 /dev/sdi
mdadm: set /dev/sdi faulty in /dev/md0
# mdadm --remove /dev/md0 /dev/sdi
```
#### TASK: Bring /dev/sdi back into array 
```
# mdadm --add/dev/md0 /dev/sdi
```
#### TASK: Verify that there are three drives as RAID 5 and a spare
```
# mdadm --detail /dev/md0
/dev/md0:
        Version : 1.2
  Creation Time : Sun Nov 18 19:24:46 2018
     Raid Level : raid5
     Array Size : 100352 (98.02 MiB 102.76 MB)
  Used Dev Size : 50176 (49.01 MiB 51.38 MB)
   Raid Devices : 3
  Total Devices : 4
    Persistence : Superblock is persistent

    Update Time : Sun Nov 18 19:35:03 2018
          State : clean
 Active Devices : 3
Working Devices : 4
 Failed Devices : 0
  Spare Devices : 1

         Layout : left-symmetric
     Chunk Size : 512K

           Name : lb40:0  (local to host lb40)
           UUID : 11285a2f:1c02a2db:834572f6:17cec6c0
         Events : 82

    Number   Major   Minor   RaidDevice State
       4       8      176        0      active sync   /dev/sdl
       6       8      144        1      active sync   /dev/sdj
       3       8      160        2      active sync   /dev/sdk

       5       8      128        -      spare   /dev/sdi

``` 
#### TASK: Make this current status persistent after boot
```
# mdadm --detail --scan >> /etc/mdadm/mdadm.conf
# echo "/dev/md0        /raid   ext4    defaults        0       2" >> /etc/fstab
```
Additional resources: [tecmint - Assembling Partitions as RAID Devices](https://www.tecmint.com/creating-and-managing-raid-backups-in-linux/), [mdadm cheat sheet](http://www.ducea.com/2009/03/08/mdadm-cheat-sheet/), [MSDN blog](https://blogs.msdn.microsoft.com/maheshk/2018/06/11/lfcs-managing-software-raid/)

##	Configure systems to mount file systems on demand
Commands: mkdir, mount, umount 
Files: /etc/fstab

##	Create, manage and diagnose advanced file system permissions
Commands: chmod, find . -perms, 
```
sticky bit, setgid, and setuid
chmod 
find files with those permissions set on them
```
#### TASK: enable suid and user's executable bit for new file called /tmp/7.4/suid
```
# touch /tmp/7.4/suid
# chmod u+sx  /tmp/7.4/suid 
# ls -lt  /tmp/7.4/suid 
-rwsr--r-- 1 root root 0 Nov 18 16:36 /tmp/7.4/suid
```
#### TASK: enable suid and user's executable bit for new file called /tmp/7.4/guid
```
# touch /tmp/7.4/guid
# chmod g+sx  /tmp/7.4/guid
# ls -lt  /tmp/7.4/guid
-rwsr--r-- 1 root root 0 Nov 18 16:36 /tmp/7.4/guid
```
#### TASK: Using octal, set /tmp/7.4/octal to GUID and read only for everyone.
First, lookup octal values with `man 2 stat`
```
# touch  /tmp/7.4/octal
# chmod 02444   /tmp/7.4/octal
# ls -al /tmp/7.4/octal
---x--s--x 1 root root 0 Nov 18 20:01 /tmp/7.4/octal
```
#### TASK: Using octal, set /tmp/7.4/permissive to SUID and readwrite for everyone.
First, lookup octal values with `man 2 stat`
```
# touch  /tmp/7.4/permissive
# chmod 04777   /tmp/7.4/permissive
# ls -al /tmp/7.4/permissive
-rwsrwxrwx 1 root root 0 Nov 18 20:04 /tmp/7.4/permissive
```

### Find all SUID files in /tmp and make a record in /tmp/5.2/suid
```
# grep -i "suid" -R /usr/share/ | grep find
/usr/share/man/man1/find.1:.B find / \e( \-perm \-4000 \-fprintf /root/suid.txt \(aq%#m %u %p\en\(aq \e) , \e
# man find 
# find /tmp -perm 4000 -print > /tmp/5.2/suid
```
##	Setup user and group disk quotas for filesystems
Commands: quotacheck, quotaon, quotaoff, edquota, quotatool, quota 
Assumption is that /quota is a separate partition on /dev/sdh1 taking up the entire 50 MB drive. If not, make one for home somewhere and somehow.
```
# apt install quota quotatool
# mkdir /quota
/etc/fstab needs to have ,usrquota,grpquota added as options
man mount # helps to point out the syntax for ,usrquota,grpquota
mount -o remount /quota
# /etc/crontab needs to have a quotacheck added to it
# quotacheck -acug
# quotaon -a
```
Limiting particular users or groups block storage using "quota"  

Hard and soft limits and set grace periods

Set quotas with a soft limit of 3 files and a hard limit of 7 for user fred  using 1 day grace limit and verify both as root and as fred:
```
# mkdir /quota/fred; chown fred /quota/fred
# quotatool -i -q 3 -u fred /quota
# quotatool -i -l 7 -u fred /quota
# quotatool -i -u fred -t 2 /quota
# sudo su – fred
$ cd /quota/fred/
touch {1..100} # this should fail after 7 files
```
We can test groups now with a  user sally and fred who both belongs to a group named limit which for the group can only store 1MB hard limit:
```
# useradd -m sally -N
# groupadd limit
# usermod sally -a -G limit
# usermod fred -a -G limit
# quotatool -b -l 1025 -g limit /quota
# sudo su – sally
# mkdir /quota/limit
# chgrp limit /quota/limit
# chmod 770 /quota/limit
# sudo su - sally
$ cd /quota/limit
$ dd if=/dev/zero of=big bs=1024 count=60000
dd: error writing 'big': Disk quota exceeded
1025+0 records in
1024+0 records out
1048576 bytes (1.0 MB, 1.0 MiB) copied, 0.00722797 s, 145 MB/s
$ ls -alh big
-rw-r--r-- 1 sally limit 1.0M Nov 15 22:31 big
```
While we are at it; let's make it so a new user named "walt" is restricted to a permanent limit of 4 MB and a temporary allowance to have up to 7 MB and make the length of time allowed to exceed temporary limit 1 day. 
```
root # useradd walt -m
root # quotatool -u walt -b -q 4M /quota
root # quotatool -u walt -b -l 7M /quota
root # quotatool -u walt -b -t "1 day"  /quota
# repquota -a
*** Report for user quotas on device /dev/sdh1
Block grace time: 24:00; Inode grace time: 7days
                        Block limits                File limits
User            used    soft    hard  grace    used  soft  hard  grace
----------------------------------------------------------------------

walt      --      28    4096    7168              5     0     0
root # sudo su – walt
walt $ dd if=/dev/urandom of=~/bigfile bs=1M count=1000
dd: error writing '/quota/walt/bigfile': Disk quota exceeded
7+0 records in
6+0 records out
7311360 bytes (7.3 MB, 7.0 MiB) copied, 0.196509 s, 37.2 MB/s
rm ~/bigfile
```
Additional resources: [tecmint - Disk Quotas for Users and Groups](https://www.tecmint.com/set-access-control-lists-acls-and-disk-quotas-for-users-groups/)

## Create and configure file systems

#### Task: Create physical volumes on top of /dev/sdn and /dev/sdo and display all volumes:
```
lvm> pvcreate /dev/sdn /dev/sdo
  Physical volume "/dev/sdn" successfully created
  Physical volume "/dev/sdo" successfully created
lvm> pvs
  PV         VG   Fmt  Attr PSize  PFree
  /dev/sdn   VG   lvm2 a--  48.00m 48.00m
  /dev/sdo   VG   lvm2 a--  48.00m 48.00m
```
#### Task: Create the volume group named VG with extends sized 256k:
```
lvm> vgcreate --physicalextentsize 256k VG /dev/sdn /dev/sdo
Volume group "vg" successfully created
lvm> vgs
  VG   #PV #LV #SN Attr   VSize  VFree
  VG     2   0   0 wz--n- 96.00m 96.00m
```

You can lookup UUID with “blkid” anytime for all drives to configure a lvm into fstab

Additional resources: [tecmint - Filesystem Troubleshooting](https://www.tecmint.com/linux-basic-shell-scripting-and-linux-filesystem-troubleshooting/)
#	Network Configuration
https://www.cheatography.com/nhatlong0605/cheat-sheets/lfcs-module3-networking/
##	Hostname resolution
```
ip addr sh
nmtui
```
### TASK: Add nameserver 4.2.2.2 as a backup to existing network configuration
```
# echo "nameserver 4.2.2.2" >> /etc/resolv.conf
```
### TASK: Add a hostname entry in lb40 for static mapping of  lb50 to 10.20.30.50
```
# echo -e "10.20.30.50\tlb50" >> /etc/hosts
```
##	Host-based firewalling with iptables
Although Ubuntu often uses ufw as an iptables front end, regular iptables is better to learn as it translates across all Linux distributions as well as it is more powerful.  

All filtering tasks to be performed on lb90 only to avoid creating difficulty of troubleshooting a broken service on a machine also running another service such as bind or squid.
#### Tasks: Disable ufw if configured on lb90,  and setup iptables to accept ssh as first rule.
```
iptables -A INPUT -p tcp --dport ssh -j ACCEPT
```
#### TASK: Rule 2 shall block 80 from 10.20.30.60 but then rule 3 allow the rest of network 10.20.30.0/24 to connect to 80.
```
iptables -A INPUT -p tcp --source 10.20.30.60 --dport 80 -j DROP
iptables -A INPUT -p tcp --source 10.20.30.0/24 --dport 80 -j ACCEPT
```
#### TASK: Save current iptable rules to /etc/iptables
```
iptables-save > /etc/iptables
```
Additional resources: [tecmint - iptables](https://www.tecmint.com/basic-guide-on-iptables-linux-firewall-tips-commands/)

##	Route traffic statically
Commands: route, netstat -nr
Documentation : `/usr/share/doc/ifupdown/examples/network-interfaces` which is at bottom of the `man interfaces` page.
#### TASK: Place a copy of the  current routing table using numeric values only in /tmp/3.2/routes  
```
# route -nr > /tmp/3.2/routes  
```
#### TASK: Statically route traffic dRestined for 1.2.3.0/24 through IP 10.20.40.20 persistent across reboot
```
# grep route /etc/network/interfaces
      up route add -net 1.2.3.0 netmask 255.255.255.0 gw 10.20.40.20
# /etc/init.d/networking restart
```
#### TASK: Temporarily for this session route traffic destined for host 2.3.4.5 through IP 10.20.40.20
```
# route add 2.3.4.5 gw 10.20.40.20
```
##	Establishing and maintaining correct time 
Traditional NTP server is best; when installing ntp package the new Ubuntu mechanism gets disabled which is what you want at this time as the new Ubuntu method drifts slightly and is just not as good.
```
# apt install ntp -y -qqq
```
Setup ntp client to using several pool servers on lb40 and host service to other lb50-lb60 devices:
```
# cat /etc/ntp.conf
pool 0.ubuntu.pool.ntp.org iburst
pool 1.ubuntu.pool.ntp.org iburst
pool 2.ubuntu.pool.ntp.org iburst
pool 3.ubuntu.pool.ntp.org iburst
restrict 10.20.30.0 netmask 255.255.255.0 nomodify notrap
```
Reset services after changing settings:```systemctl restart ntp```
Check NTP status of peers with only numeric IP entries to speed up checking:
```
# ntpq -p -n 
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
+69.197.188.178  204.9.54.119     2 u   31   64    1  126.150  -53.825  16.771
+193.225.126.76  121.131.112.137  2 u   32   64    1  188.002  -56.683  16.314
``` 
Results above are truncated.  Look for entries that aren't 16 under st column
To setup lb40 as server for lb50:
```
# cat /etc/ntp.conf
server 10.20.30.40
# systemctl restart ntp
# ntpq -p -n
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 10.20.30.40     184.105.182.16   3 u   10   64    1    0.361   93.312   0.000
```

To change timezone to CST:
```
# timedatectl set-timezone America/Chicago
# timedatectl status
      Local time: Mon 2018-11-12 11:18:47 CST
  Universal time: Mon 2018-11-12 17:18:47 UTC
        RTC time: Mon 2018-11-12 17:18:48
       Time zone: America/Chicago (CST, -0600)
 Network time on: yes
NTP synchronized: yes
 RTC in local TZ: no
```
#	Configurating Services
https://www.cheatography.com/nhatlong0605/cheat-sheets/lfcs-module6-serviceconfiguration/
## Virtual Machines
### Configure a hypervisor to host virtual guests
**Ok, so, honestly this is one limitation of the VirtualBox approach with Vagrant.  You are either going to need to setup another machine really running Linux or something like that.**
### Install Linux systems as virtual guests
### Start, stop, modify the status of virtual machines
### Access a VM console
### Migrate a VM between two hosts
### Configure systems to launch virtual machines at boot
### Evaluate memory usage of virtual machines
### Create light-weight virtualized guests via namespaces
### Resize RAM or storage of VMs
### Cloning and replicating VMs using images or snapshots
Ubuntu can provide hypervisor services to run VMs within domains.

To enable this, you will need appropriate packages:
```
apt install -y qemu-kvm qemu-img libvirt
```
Ensure that libvirtd service is running:
```
systemctl {enable,start} libvirtd
```
#### TASK: Get a default vm

#### TASK: Change processor count on an existing libvirt machine from 1 to 2
```# virsh edit vm```
#### TASK: Set a libvirt machine to boot every time machine is rebooted
```virsh autostart```
#### TASK: Disable startup of a VM on reboot
```virsh autostart --disable vm```
#### TASK: Kill a libvirt machine named vm
```virsh destroy vm```

##	DNS server
###	Simple local DNS server for lookup and caching speed
A local DNS server can speedup lookups for network.  This is often called a caching server. 
Before we can do anything, we need our IP address and hostname
```
# hostnamectl
# ifconfig | grep inet
```
Install DNS stuff and switch to configuration folder:
```
# apt install bind9 dnsutils -y
cd /etc/bind/
```
Setup DNS caching server to use 10.0.0.1:
```
grep "forwarders {" -A 2 /etc/bind/named.conf.options
        forwarders {
                10.0.0.1
        };
```
Start service, enable for reboot, and test:
```
systemctl status bind9
systemctl enable bind9
dig www.google.com @localhost
```
Additional resources: [tecmint - Caching DNS Server](https://www.tecmint.com/setup-recursive-caching-dns-server-and-configure-dns-zones/)
###	DNS zone management
Using domainname of ubuntu.local; create master DNS server
```
# cat /etc/bind/named.conf.local
zone "ubuntu.local" {
        type master;
        file "/etc/bind/db.ubuntu.local";
        };
cp /etc/bind/db.local /etc/bind/db.ubuntu.local
```
Edit values for zone to add a test.ubuntu.local at 10.20.30.40 and test:
```
tail -1  /etc/bind/db.ubuntu.local
test    IN      A       1.2.3.4
systemctl restart bind9
dig test.ubuntu.local @localhost | grep 10.20.30.40
test.ubuntu.local.      604800  IN      A       1.2.3.4
```
Set machine to use local DNS server:
```
grep "dns-" /etc/network/interfaces
dns-nameserver 192.168.1.3
```
test to make sure server is using 127.0.0.1 as name server
```
dig google.com | grep ANSWER -A 5
;; ANSWER SECTION:
google.com.             58      IN      A       172.217.14.238

;; AUTHORITY SECTION:
.                       195835  IN      NS      m.root-servers.net.
.                       195835  IN      NS      f.root-servers.net.
```
Additional resources: [tecmint - Configure Zones for Domain](https://www.tecmint.com/setup-recursive-caching-dns-server-and-configure-dns-zones/)
##	Email
###	Configure an IMAP and IMAPS service
```
apt -y install postfix dovecot
vi /etc/postfix/main.cf
	myorigin = /etc/mailname (hostname)
	file /etc/postfix/transport and command postmap /etc/postfix/transport
```	
Additional resources: [Installing Postfix and Dovecot](https://www.tecmint.com/installing-network-services-and-configuring-services-at-system-boot/)
### Configure email aliases
```
# grep /etc/postfix/aliases: 
sysadmin: fred, lab
# postalias /etc/postfix/aliases
```
##	Secure Shell (SSH) 
```
systemctl status sshd
ssh -l username ipaddress
ssh ciphers
```
#### TASK: Set a password for the account larry on hosts 40 & 50
#### TASK: Generate an ssh-keypair for the larry account on hosts 40 & 50
#### TASK: Copy the ssh-id from 40 to 50 to enable passwordless logins to 50
#### TASK: Copy the ssh-id from 50 to 40 to enable passwordless logins to 40
#### TASK: SSH as the user larry from 40 to 50 to test passwordless authentication
#### TASK: SSH as the user larry from 50 to 40 to test passwordless authentication

#### TASK: For SSH server on lb90, disable  TCP keepalives, enable password authentication, and disable remote root login 
```
# cat /etc/ssh/sshd_config
TCPKeepAlive no
PermitRootLogin no
PasswordAuthentication yes
```
#### TASK: Allow users luke and vagrant to ssh in
```
# grep Users /etc/ssh/sshd_config
AllowUsers vagrant luke
```

##	Proxy server
Install and enable http proxy server:
```
apt -y install squid
systemctl restart squid.service
systemctl enable squid.service
```
Allow subnet 10.20.30.0/24 but block ip address 10.20.30.50: 
```
acl localnet src 10.20.30.0/24 
http_access allow localnet 
acl badguy src 10.20.30.50
http_access allow localhost !badguy
```
Testing from client devices:
```
apt -y install wget
export http_proxy='http://10.20.30.40:8080/'
URL='www.google.com'
wget -q --proxy-user=test --proxy-password=test --spider $URL
if [ $? = 1 ]
then
    STATUS= echo "Proxy isn't working"
else
  STATUS="Proxy is working."
fi
echo $STATUS
```
Set lb50 to use lb40 as proxy server for updates:
```
# echo "Acquire::http::Proxy \"http://10.20.30.40:8080\"; " >  /etc/apt/apt.conf
```
## Web server
### Configure an HTTP server
###  Configure HTTP server log files
### Configure SSL with HTTP server
### Set up name-based virtual web hosts
### Deploy a basic web application
### Restrict access to a web page
#### Set up a default configuration HTTP server with SELinux in Enforcing mode and active firewalld configuration.
`
apt install -y apache2
systemctl enable apache2
systemctl start apache2
`
#### Task: Configure web server to use as document root /var/www/html
```
# grep -i DOC  /etc/apache2/sites-enabled/000-default.conf
        DocumentRoot /var/www/html
```
###	Web server log files
Customize HTTP server log files.

###	Restrict access to a web page
Restrict access to a web page in Apache2
##	Database server
MariaDB is the most common database server and is what we use.
#### TASK: Setup MariaDB 

##	Containers
#### TASK: Setup Linux container ubuntu and start it
```
# apt install lxc
# systemctl status lxc.service
# systemctl enable lxc
# lxc-ls
# lxc-start -n mylxcontainer
# lxc-info -n mylxcontainer { should see the state as RUNNING }  
# lxc-console -n mylxcontainer { console login to the container, username -ubuntu, pass-ubuntu }  
# ubuntu@mylxcontainer:~$ { upon login your console prompt changes takes you to ubuntu }  
# uname -a or hostname { to confirm you are within the container }  
# Type <cntrl+a q> to exit the console  
# lxc-stop -n mylxcontainer  
# lxc-destroy -n mylxcontainer
```
#### TASK: Use Docker to setup a web server before destroying it.
```
$ apt update  
$ free -m  
$ apt install docker.io  
$ systemctl enable docker  
$ systemctl start docker  
$ systemctl status docker  
$ docker info  
$ docker version  
$ docker run hello-world { to pull the hello-world for testing }  
$ docker ps  
$ docker ps -la or $ docker ps -a { list all the containers }  
$ docker search apache or microsoft { to search container by name }  
$ docker images { to list all the images in localhost }  
$ docker pull ubuntu  
$ docker run -it --rm -p 8080:80 nginx { for nginx, -it for interative }  
$ docker ps -la { list all the containers, look for container_id, type first 3 letters which is enough }  
$ docker start container_id or ubuntu { say efe }  
$ docker stop efe  
$ docker run -it ubuntu bash  
$ root@efe34sdsdsds:/# { takes to container bash }  
_<type cntrl p + cntrl q> to switch back to terminal_  
$ docker save debian -o mydebian.tar  
$ docker load -i mydebian.tar  
$ docker export web-container -o xyz.tar  
$ docker import xyz.tar  
$ docker logs containername or id  
$ docker logs -f containername or id { live logs or streaming logs }  
$ docker stats  
$ docker top container_id  
$ docker build -t my-image dockerfiles/ or $ docker build -t aspnet5 . { there is a dot at the end to pick the local yaml file for the build }
```
Additional information: https://blogs.msdn.microsoft.com/maheshk/2018/05/27/lfcs-commands-to-manage-and-configure-containers-in-linux/
## File Sharing
#### TASK: Setup NFS  on lb60 to share /nfs with lb40 machines on network as /nfs
```
lb60 # apt-get install nfswatch nfs-common nfs-kernel-server
lb60 # # cat /etc/hosts
127.0.1.1       lb60    lb60
10.20.30.40     lb40    lb40
10.20.30.50     lb50    lb50
lb60 # cat /etc/exports
/nfs       hostname1(rw,sync,no_subtree_check)
```
#### TASK: Setup SAMBA  on lb60 to share /smb with lb40 machines on network as /smb



# Other resources
* [tecmint series](https://www.tecmint.com/sed-command-to-create-edit-and-manipulate-files-in-linux/)
* [older guide that inspired me](http://blog.leonelatencio.com/wp-content/uploads/2014/11/Linux-Foundation-Certified-System-Administrator-LFCS-v1.3.pdf)
* [chousensha notes](http://chousensha.github.io/blog/categories/sysadmin/)
* [Awesome start on basic CLI](https://github.com/learnbyexample/Linux_command_line/blob/master/README.md)
* [Presentations](https://github.com/guruskill/LinuxAzure/tree/master/Presentations)
* [Random notes](https://github.com/ravexina/linux-notes)
* [Cert Exam Prep: Linux Foundation Certified System Administrator (LFCS) by  Mark Grimes](https://channel9.msdn.com/events/Ignite/2016/BRK3268?term=Mark%20Grimes&lang-en=true)
* [TOC Generator](https://ecotrust-canada.github.io/markdown-toc/)
* [nhatlong0605](https://www.cheatography.com/nhatlong0605/cheat-sheets/)
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE0ODgyMDE2NDcsLTE2NDQzMTI3MjYsLT
EwMDQ2OTI5NzYsMTk1NDA3NzA2MywtNzczMjQ5MDIyLC0xNDAw
MTYxNjM3LC0xMjcxMDI4NTA4XX0=
-->