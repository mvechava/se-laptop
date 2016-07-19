# SUSE SE Laptop Setup Guide

##Installation of SUSE Solutions on SE laptops

1. Install openSUSE Leap 42.1 and SLES 12 SP1 on ThinkPad W540

	1.1. Partitioning Scheme:

		256GB M.2 SSD

		/dev/sdb1 - FAT16 /boot/efi/EFI for openSUSE (200MB)

		/dev/sdb2 - FAT16 /boot/efi/EFI for SLES 12 SP1 (200MB)

		/dev/sdb3 - btrfs / for openSUSE (50GB)

		/dev/sdb4 - btrfs / for SLES 12 SP1 (50GB)

		/dev/sdb5 - xfs   /home for openSUSE (Rest / 2)

		/dev/sdb6 - xfs	  /home for SLES 12 SP1 (Rest / 2)

		1TB SATA III SSD

		/dev/sda1 - xfs	  /data for both OSes

2. Register SLES 12 SP1 using YaST -> Product Registration

3. Install SLES 12 SP1 Workstation Extension & Container Module using YaST -> Add-On Products

4. Install docker

	4.1. `zypper in docker docker-compose sle2docker zypper-docker sles11sp3-docker-image sles11sp4-docker-image sles12-docker-image sles12sp1-docker-image`

	4.2. `sudo systemctl enable docker.service`

	4.3. `sudo systemctl start docker.service`

	4.4. `sudo usermod -aG docker mechavarria`

	4.5. Enable IPv4 forwarding using yast2 lan

	4.6. `vi /etc/sysconfig/SuSEfirewall2` and change FW_ROUTE="yes"
	
5. Install SMT on laptop

	5.1. `zypper in -t pattern smt`

	5.2. YaST -> SMT Configuration Wizard

	5.3. `smt-repos -e <SLES12-SP1-Pool, SLES12-SP1-Updates, ... etc.>`

	5.4. `smt-mirror` (several times as the first one doesn't always get everything properly)

	5.5. `systemctl start smt`

	5.6. `systemctl enable smt`

6. Create VMware networks

	6.1. Create vmnet0: PXE network (bridge)

	6.2. Create vmnet1: 172.16.31.0/24 - Data network (NAT)

	6.3. Create vmnet2: 192.168.124.0/24 - Admin network (NAT)

	6.4. Create vmnet2.200: 192.168.125.0/24 (VLAN 200) - Cloud storage (external Ceph cluster, user only)

		6.4.1. vi /etc/sysconfig/network/ifcfg-vmnet2.200
 
			# VLAN 200 Interface for the vmnet2 network
			# Storage LAN for Cloud
			MANAGED='false'
			USERCONTROL='no'
			STARTMODE='manual'
			BOOTPROTO='static'
			ETHERDEVICE='vmnet2'
			IPADDR='192.168.125.1/24'
			VLAN_ID='200'
			PROMISC='yes'
			BROADCAST=''
			ETHTOOL_OPTIONS=''
			MTU=''
			NAME=''
			NETWORK=''
			REMOTE_IPADDR=''

	6.5. Create vmnet2.300: 192.168.126.0/24 (VLAN 300) - Cloud public network (NAT)

		6.5.1. vi /etc/sysconfig/network/ifcfg-vmnet2.300

			# VLAN 300 Interface for the vmnet2 network
			# Public LAN for Cloud
			MANAGED='false'
			USERCONTROL='no'
			STARTMODE='manual'
			BOOTPROTO='static'
			ETHERDEVICE='vmnet2'
			IPADDR='192.168.126.1/24'
			VLAN_ID='300'
			PROMISC='yes'
			BROADCAST=''
			ETHTOOL_OPTIONS=''
			MTU=''
			NAME=''
			NETWORK=''
			REMOTE_IPADDR=''

	6.6. Create vmnet3: 192.168.127.0/24 - iSCSI network 1 (user only)

	6.7. Create vmnet4: 192.168.128.0/24 - iSCSI network 2 (user only)

	6.8. Create vmnet5: 192.168.129.0/24 - HA network (user only)

	6.9. Create fixvmnet script

		6.9.1. sudo vi /usr/local/bin/fixvmnet

			#!/bin/bash
			chmod 777 /dev/vmnet*
			chgrp users /dev/vmnet*
			ifup vmnet2.200
			ifup vmnet2.300

		6.9.2. sudo chmod +x /usr/local/bin/fixvmnet

	6.10. `sudo fixvmnet`

7. Configure named on laptop

	7.1. `sudo zypper in bind`

	7.2. `sudo vi /etc/named.conf`

		7.2.1. Lines to add/modify:

			forwarders { 8.8.8.8; 8.8.4.4; };

			zone "demo.com" in {
				type master;
				file "demo.zone";
			};

			zone "124.168.192.in-addr.arpa" in {
				type master;
				file "192.168.124.zone";
			};

	7.3. `sudo vi /var/lib/named/192.168.124.zone`

		7.3.1.
			;
			; Reverse zone file for 192.168.124.0/24 subnet
			;
			; The full zone file
			;
			$TTL 3D
			@       IN      SOA     thinkpad-w540.demo.com. mechavarria.suse.com. (
							1		; serial
							2D		; refresh
							4H		; retry
							6W		; expiry
							1W )		; minimum
					NS		thinkpad-w540.demo.com.
 
			1		PTR	thinkpad-w540.demo.com.
			2		PTR	gateway.demo.com.
			3		PTR	manager.demo.com.
			10		PTR	admin.cloud.demo.com.

	7.4. `sudo vi /var/lib/named/demo.zone`

		7.4.1.
			;
			; Zone file for demo.com
			;
			; The full zone file
			;
			$TTL 3D
			@       IN      SOA     thinkpad-w540.demo.com. mechavarria.suse.com. (
							1		; serial
							2D		; refresh
							4H		; retry
							6W		; expiry
							1W )		; minimum
					NS	thinkpad-w540.demo.com.
  
			localhost       A       127.0.0.1
			thinkpad-w540   A       192.168.124.1
			gateway		A	192.168.124.2
			manager         A       192.168.124.3
			admin.cloud     A       192.168.124.10

	7.5. `sudo systemctl start named`

	7.6. `sudo systemctl enable named`

8. Create SUSE Manager 3 VM

	8.1. Create VM with folllowing settings:

		-2GB RAM
		-1x dual core processor
		-200GB VHD
		-eth0 - vmnet1
		-eth1 - vmnet2
		-CD/DVD - SLE-12-SP1-Server-DVD-x86_64-GM-DVD1.iso

	8.2. Boot/Install SLES 12 SP1

	8.3. Configure network:
	
		-eth0 - DHCP 172.16.31.0/24
		-eth1 - 192.168.124.3
		-Hostname - manager.demo.com
		-Gateway - 172.16.31.2
		-Nameserver - 192.168.124.1

	8.4. Configure NTP for us.pool.ntp.org to start at boot (hardware clock not running UTC)

	8.5. `/usr/lib/suseRegister/bin/clientSetup4SMT.sh https://thinkpad-w540.demo.com/center/regsvc`

	8.6. `zypper up`

	8.7. `yast2 susemanager_setup`

	8.8. Mirror channels (TIME CONSUMING)

	8.9. Create activation key according to installation guide

	8.10. Create bootstrap script according to installation guide

	8.11. Edit bootstrap script according to installation guide

9. Create SUSE OpenStack Cloud 6 Admin VM

	9.1. Create VM with following settings:

		-2GB RAM
		-1x dual core processor
		-100GB VHD (OS)
		-2GB VHD (PostgreSQL)
		-200MB VHD (RabbitMQ)
		-8MB VHD (SBD)
		-eth0 - vmnet2
		-eth1 - vmnet2
		-eth2 - vmnet1
		-CD/DVD - SLE-12-SP1-Server-DVD-x86_64-GM-DVD1.iso

	9.2. Boot/Install SLES 12 SP1 w/ SUSE Cloud 6 add-on

	9.3. Configure network:

		-eth0 - bond slave
		-eth1 - bond slave
		-eth2 - DHCP 172.16.31.0/24
		-bond0 - 192.168.124.10/24
		-Hostname - admin.cloud.demo.com
		-Gateway - 172.16.31.2
		-Nameserver - 192.168.124.1

	9.4. Configure NTP for us.pool.ntp.org to start at boot (hardware clock not running UTC)

	9.5. In YaST -> Product Registration, register system against https://thinkpad-w540.demo.com/center/regsvc`

	9.6. `zypper in -t pattern cloud_admin && zypper up`

	9.7. `yast2 crowbar`

		9.7.1. Change admin network router to 192.168.124.2

		9.7.2. Change network mode to teaming

		9.7.3. Add bastion network with following settings

			-IP Address - 172.16.31.10
			-Router - 172.16.31.2
			-Subnet - 172.16.31.0
			-Netmask - 255.255.255.0
			-Broadcast - 172.16.31.255
			-Physical interface - ?1g3 (eth2)

		9.7.4. Change repositories to Remote SMT Server (http://thinkpad-w540.demo.com)

	9.8. Copy ISO contents

		9.8.1. Select SLE-12-SP1-Server-DVD-x86_64-GM-DVD1.iso as CD/DVD device in VMware

		9.8.2. mount /dev/sr0 /mnt

		9.8.3. rsync -avP /mnt/ /srv/tftpboot/suse-12.1/x86_64/install/
		
		9.8.4. umount /mnt
		
		9.8.5. eject /dev/sr0

		9.8.6. Select SUSE-OPENSTACK-CLOUD-6-x86_64-GM-DVD1.iso as CD/DVD device in VMware

		9.8.7. mount /dev/sr0 /mnt

		9.8.8. mkdir /srv/tftpboot/suse-12.1/x86_64/repos/Cloud

		9.8.9. rsync -avP /mnt/ /srv/tftpboot/suse-12.1/x86_64/repos/Cloud

	9.9. `screen install-suse-cloud`

	9.10. Navigate to http://admin.cloud.demo.com

	9.11. Add new "Administration" group, drag admin into "Administration" group

	9.12. Edit admin node:

		Alias - Administration
		Public Name - admin.cloud.demo.com

	9.13. Edit DNS barclamp and add 192.168.124.1 to "Forwarders" and apply

	9.14. Create iSCSI targets

		9.14.1. Create xfs partition that takes up whole disk on /dev/disk/by-path/pci-0000:00:10.0-scsi-0:0:1:0 and /dev/disk/by-path/pci-0000:00:10.0-scsi-0:0:2:0
		9.14.2.	zypper in yast2-iscsi-lio-server
		9.14.3. yast2 iscsi-lio-server
			-Service Start - When Booting
			-No Authentication
			-IP Address - 192.168.124.10
			-LUN 0 - pgsql, /dev/disk/by-path/pci-0000:00:10.0-scsi-0:0:1:0
			-LUN 1 - rmq, /dev/disk/by-path/pci-0000:00:10.0-scsi-0:0:2:0
			-LUN 2 - sbd, /dev/disk/by-path/pci-0000:00:10.0-scsi-0:0:3:0

10. Create SUSE OpenStack Cloud 6 Controller VMs

	10.1. Create VMs

		-4GB RAM
		-1x dual core processor
		-20GB VHD
		-eth0 - vmnet2
		-eth1 - vmnet2

	10.2. PXE boot controller nodes

	10.3. Edit controller nodes:

		Alias - Controller-1, Controller-2
		Public Name - controller-1.cloud.demo.com, controller-2.cloud.demo.com
		Group - Controller
		Intended Role - Controller

	10.4. Allocate controller nodes

	10.5. Attach iSCSI LUNs

		10.5.1. Add iSCSI initiator names to Admin server iSCSI client configuration (cat 
		/etc/iscsi/initiatorname.iscsi)
		
		10.5.2. yast2 iscsi-client (on both controllers)
		
		10.5.3. Connect to iSCSI LUNs at 192.168.124.10 (on both controllers)

	10.6. Enable SBD

		10.6.1. zypper in sbd (on all controllers)

		10.6.2. sbd -d /dev/disk/by-path/ip-192.168.124.10\:3260-iscsi-iqn.2016-06.demo.cloud\:ec6910f2-e32d-4ebc-9e91-5f975e5ef140-lun-2 create (on all controllers)

11. Create SUSE OpenStack Cloud 6 Storage VMs

	11.1. Create Admin VM:

		-1.5GB RAM
		-1x single core processor
		-20GB VHD
		-eth0 - vmnet2
		-eth1 - vmnet2

	11.2. Create OSD VMs:

		-2GB RAM
		-1x dual core processor
		-20GB VHD
		-150GB VHD
		-eth0 - vmnet2
		-eth1 - vmnet2

	11.3. PXE boot storage nodes

	11.4. Edit storage nodes:

		Alias - Storage-1, Storage-2, Storage-3, Storage-4
		Public Name - storage-1.cloud.demo.com, storage-2.cloud.demo.com, storage-3.cloud.demo.com, storage-4.cloud.demo.com
		Group - Storage
		Intended Role - Storage

	11.5. Allocate storage nodes

12. Create SUSE OpenStack Cloud 6 Compute VMs

	12.1. Create Compute VMs:

		-4GB RAM
		-2x dual core processors
		-20GB VHD
		-eth0 - vmnet2
		-eth1 - vmnet2

	12.2. PXE boot compute VMs

	12.3. Edit compute nodes:

		Alias - Compute-1, Compute-2
		Public Name - compute-1.cloud.demo.com, compute-2.cloud.demo.com
		Group - Compute
		Intended Role - Compute

	12.4. Allocate compute nodes

	12.5. Attach iSCSI LUNs
	
		12.5.1. Add iSCSI initiator names to Admin server iSCSI client configuration (cat /etc/iscsi/initiatorname.iscsi)

		12.5.2. yast2 iscsi-client (on both compute nodes)

		12.5.3. Connect to iSCSI LUNs at 192.168.124.10 (on both compute nodes)

	12.6. Enable SBD

		12.6.1. zypper in sbd (on all compute nodes)

		12.6.2. sbd -d /dev/disk/by-path/ip-192.168.124.10\:3260-iscsi-iqn.2016-06.demo.cloud\:ec6910f2-e32d-4ebc-9e91-5f975e5ef140-lun-2 create (on all compute nodes)

13. Deploy barclamps

	13.1. Deploy Pacemaker Barclamp:

		13.1.1. Create control proposal:
			-Proposal Name - pacemaker
			-Transport for communication - Unicast UDPU
			-Policy when cluster does not have quorum - stop
			-Configuration mode for STONITH - Configured with STONITH Block Devices (SBD)
			-Kernel module for watchdog - softdog
			-Block devices for node - /dev/disk/by-path/ip-192.168.124.10:3260-iscsi-iqn.2016-06.demo.cloud:ec6910f2-e32d-4ebc-9e91-5f975e5ef140-lun-2
			-Do not start corosync on boot after fencing - false
			-Mail notifications - false
			-Prepare cluster for DRBD - false
			-Public name for public virtual IP - LEAVE BLANK
			-pacemaker-cluster-member - Controller-1, Controller-2
			-hawk-server - Controller-1, Controller-2
			-pacemaker-remote - Compute-1, Compute-2

	13.2. Deploy SUSE Manager Client Barclamp:

		13.2.1. mkdir -p /opt/dell/chef/cookbooks/suse-manager-client/files/default

		13.2.2. cd /opt/dell/chef/cookbooks/suse-manager-client/files/default

		13.2.3. wget http://manager.demo.com/pub/rhn-org-trusted-ssl-cert-1.0-1.noarch.rpm

		13.2.4. mv rhn-org-trusted-ssl-cert-1.0-1.noarch.rpm ssl-cert.rpm

		13.2.5. /opt/dell/bin/barclamp_install.rb --rpm core

		13.2.6. Create SUSE Manager proposal:
			-Activation Key - <key_name>
			-SUSE Manager server hostname - manager.demo.com
			-suse-manager-client - all nodes

	13.3. Deploy Database Barclamp:

		13.3.1. Create Database proposal:
			-Global Connection Limit - 1000
			-database-server - pacemaker
			-Storage Mode - Shared Storage
			-Name of Block Device - /dev/disk/by-path/ip-192.168.124.10:3260-iscsi-iqn.2016-06.demo.cloud:ec6910f2-e32d-4ebc-9e91-5f975e5ef140-lun-0-part1
			-Filesystem Type - xfs

	13.4. Deploy RabbitMQ Barclamp:

		13.4.1. Create RabbitMQ proposal:
			-Virtual host - /nova
			-Port - 5672
			-User - nova
			-rabbitmq-server - pacemaker
			-Storage Mode - Shared Storage
			-Name of Block Device - /dev/disk/by-path/ip-192.168.124.10:3260-iscsi-iqn.2016-06.demo.cloud:ec6910f2-e32d-4ebc-9e91-5f975e5ef140-lun-1-part1
			-Filesystem Type - xfs
			
	13.5. Deploy Keystone Barclamp:

		13.5.1. Create Keystone proposal:
			-Algorithm for Token Generation - UUID
			-Region name - Miami
			-Default Tenant - openstack
			-Administrator Username - admin	
			-Regular User Username - crowbar
			-Protocol - HTTP
			-keystone-server pacemaker

	13.6. Deploy Ceph Barclamp:

		13.6.1. Create Ceph proposal:
			-Disk selection method - All Available
			-Number of replicas of an object - 0
			-Protocol - HTTP
			-Administrator Username - admin
			-Administrator Email Address - admin@example.com
			-Protocol - HTTP
			-ceph-calamari - Storage-1
			-ceph-mon - Storage-2, Storage-3, Storage-4
			-ceph-osd - Storage-2, Storage-3, Storage-4
			-ceph-radosgw - Storage-2

		13.6.2. Grant Calamari server admin access
			-scp storage-2:/etc/ceph/ceph.client.admin.keyring .
			-scp ./ceph.client.admin.keyring storage-1:/etc/ceph/.

		13.6.3. Detect storage nodes in Calamari
			-Navigate to 192.168.124.84
			-Initialize cluster

	13.7. Deploy Glance Barclamp:

		13.7.1. Create Glance proposal:
			-Default Storage Store - Rados
			-Expose Backend Store Location - false
			-RADOS user for CephX authentication - glance
			-RADOS pool for Glance images - images
			-Protocol - HTTP
			-Enable Caching - false
			-Turn On Cache Management - false
			-Verbose logging - true
			-glance-server - pacemaker

	13.8. Deploy Cinder Barclamp:

		13.8.1. Create Cinder proposal:

			13.8.1.1. Delete default backend

			13.8.1.2. Add RADOS backend
				-Type of Volume - RADOS
				-Name for Backend - ceph
			-Use Ceph deployed by Crowbar - true
			-RADOS pool for Cinder volumes - volumes
			-RADOS user (Set only if using CephX authentication) - cinder
			-Protocol - HTTP
			-cinder-controller - pacemaker
			-cinder-volume - Storage-2, Storage-3, Storage-4

	13.9. Deploy Neutron Barclamp:

		13.9.1. Create Neutron proposal:
			-DHCP Domain - demo.local
			-Plugin - ml2
			-Modular Layer 2 mechanism drivers - openvswitch
			-Modular Layer 2 type drivers - gre
			-Use Distributed Virtual Router setup  -false
			-Start of tunnel ID range - 1
			-End of tunnel ID range - 1000
			-Protocol - HTTP
			-neutron-server - pacemaker
			-neutron-network - pacemaker

	13.10. Deploy Nova Barclamp:

		13.10.1. Create Nova proposal:
			-Virtual RAM to Physical RAM allocation ratio - 1
			-Virtual CPU to Physical CPU allocation ratio - 16
			-Virtual Disk to Physical Disk allocation ratio - 1
			-Enable Libvirt Migration - true
			-Set up Shared Storage on nova-controller for Nova instances - false
			-Shared Storage for Nova instances has been manually configured - false
			-Protocol - HTTP
			-Keymap - en-us
			-NoVNC Protocol - HTTP
			-Verbose Logging - true
			-nova-controller - pacemaker
			-nova-compute-qemu - pacemaker (2 remote nodes)

	13.11. Deploy Horizon Barclamp:

		13.11.1. Create Horizon proposal:
			-horizon-server - pacemaker

	13.12. Deploy Heat Barclamp:

		13.12.1. Create Heat proposal:
			-heat-server - pacemaker
