flashcache-debian
=================

flashcache debian init script

flashcache installation on debian squeeze

Install prerequirements
	apt-get install git-core dkms build-essential linux-headers-`uname -r` -y
	
Clone latest flashcache
	git clone https://github.com/facebook/flashcache.git
	cd flashcache

Compile and install module
	make -f Makefile.dkms
	make install
	
Init module
	modprobe flashcache
	
Check
	dmesg | tail
	.............................
	[ 5806.891504] flashcache: flashcache-1.0 initialized

Autoload module on boot time
	echo flashcache >> /etc/modules
	
Create flashcache writeback device (I use lvm)
	flashcache_create -p back cachedev_name /dev/mapper/ssd_partition /dev/mapper/cached_partition
	# Example for cache /var and /vz
	flashcache_create -p back var_cached /dev/mapper/s0cache-var /dev/mapper/s0-var
	flashcache_create -p back vz_cached /dev/mapper/s0cache-vz /dev/mapper/s0-vz
	
Install init script
	git clone https://github.com/feldsam/flashcache-debian.git
	cp flashcache-debian/flashcache /etc/init.d/
	chmod +x /etc/init.d/flashcache
	update-rc.d flashcache defaults
	# if you have /usr on separate partition you have to copy awk bin to root partition
	cp /usr/bin/awk /bin/
	
Reboot
	init 6
	
After reboot you should have flashcache devices initialized
	ls -l /dev/mapper
	..........................
	lrwxrwxrwx 1 root root      7 Sep 19 23:49 s0cache-var -> ../dm-4
	lrwxrwxrwx 1 root root      7 Sep 19 23:49 s0cache-vz -> ../dm-5
	lrwxrwxrwx 1 root root      7 Sep 19 23:49 s0-root -> ../dm-0
	lrwxrwxrwx 1 root root      7 Sep 19 23:49 s0-swap_1 -> ../dm-1
	lrwxrwxrwx 1 root root      7 Sep 19 23:49 s0-var -> ../dm-2
	lrwxrwxrwx 1 root root      7 Sep 19 23:49 s0-vz -> ../dm-3
	lrwxrwxrwx 1 root root      7 Sep 19 23:49 var_cached -> ../dm-6
	lrwxrwxrwx 1 root root      7 Sep 19 23:49 vz_cached -> ../dm-7	
	
	# /dev/mapper/* are only symlinks to /dev/dm-* so if you want configure some vars in sysctl use "dm", example:
	cat /etc/sysctl.conf
	# change the reclaim policy from FIFO to LRU"
	dev.flashcache.dm-5+s0-vz.reclaim_policy = 1
	dev.flashcache.dm-4+s0-var.reclaim_policy = 1
	
Finaly replace noncached to cached in /etc/fstab
	/dev/mapper/vz_cached 	/vz		ext3	defaults															0	2
	/dev/mapper/var_cached 	/var	ext3	defaults,usrjquota=aquota.user,grpjquota=aquota.group,jqfmt=vfsv0	0	2
	
Reboot
	init 6
	
Now you should have mounted cachedevs... check stats
	flashcache/utils/flashstat
	
Important!! if you working with remote server then connect KVM first, because if something goes wrong, system not boot without interaction (not found devices in fstab and pressing ctrl+d :)