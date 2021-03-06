# Ultimate OpenStack IceHouse Guide

This is a Quick Guide to deploy OpenStack IceHouse on top of Ubuntu 14.04, it is IPv6-Ready!

It is compliant with OpenStack's official documentation (docs.openstack.org).

The tenant's subnets are based on Neutron, with ML2 plugin and `Single Flat Network` topology, dual-stacked.

The `Single Flat Network` is the simplest network topology supported by OpenStack (I think). So, it is easier to understand and follow.

The IPv6 support in Neutron L3 router isn't ready yet but, things are different when you don't need it. With ML2 (and probably with OVS too), you can have a dual-stacked environment with `Single Flat Network`. But, there is no IPv6 auto-configuration yet, so, you'll just need to configure the IPv6 address statically on each Instance. IPv6 Security Groups is working!

Also, basically everything else, including physical servers, basic services (RabbitMQ, MySQL, Apache, SSH), Horizon, APIs, endpoints, SPICE Consoles and etc, are all reachable through IPv6!

Apparently, only Metadata (and GRE / VXLAN subnet) still requires IPv4 (I'm not sure, maybe I am missing something).

This is a "step-by-step", a "cut-and-paste" guide.

**This Guide covers:**

* Ubuntu (hostname, hosts, network interfaces, namespaces and LVM volumes);
* Open vSwitch;
* RabbitMQ;
* MySQL;
* Keystone;
* Glance;
* Nova;
* SPICE Consoles - VDI (Virtual Desktop Infrastructure);
* Neutron with ML2 `Single Flat Network` (DHCPv4), Metadata and Security Groups;
* Cinder;
* and Dashboard.

**This Guide does not covers:**

* L3 Routers;
* NAT;
* Floating IPs;
* GRE or VXLAN tunnels;
* VLANs.

## Bitcoin Donations Accepted!

If you think that this guide is great! Please, consider a Bitcoin donation to the following address:

Bitcoin Address: `1JpNbLczAhUkbqQMv9iaQksiTd8yLaDx6K`

## TODO List

* Heat
* Ceilometer
* Trove
* Host Aggregates
* Docker.io on Heat & Nova
* Swift
* Enable Sound Device for Instances

## References

- OpenStack IceHouse Documentation for Ubuntu:

 http://docs.openstack.org/icehouse/install-guide/install/apt/content/

- OpenStack `Single Flat Network` documentation:

 http://docs.openstack.org/trunk/install-guide/install/apt/content/section_neutron-single-flat.html

- References about IPv6 for OpenStack:

 http://www.nephos6.com/pdf/OpenStack-Havana-on-IPv6.pdf

- Inspired by:

 http://openstack-folsom-install-guide.readthedocs.org/en/latest/

## Limitations

* Only 1 Ethernet per physical server (it might be a bottleneck for production environments, if you don't build BOND channels or more Ethernet segments).
* Horizon in IceHouse triggers errors (LP #1292022) when there is no L3 extensions but, there are workarounds and it is usable besides this problem.

## Features

* No NAT within the Cloud, no *Floating IPs*, no *multihost=true* (i.e. no NAT at the Compute Node itself).

## Observations

### About external router (provider / upstream):

The "border gateway / external router" (dual-stacked, default route, for both physical serves and instances) is located outside of the cloud. This means that we're *mapping our physical network into OpenStack*, using ML2 plugin with *Single Flat Network* topology.

### About IPv6:

For tenant's subnet, in the future, the "border gateway / external router" will have the IPv6 Router Advertisement daemon running on it (radvd), so, when OpenStack finishes its *IPv6 SLAAC mode from upstream* routers ( covered here: https://blueprints.launchpad.net/neutron/+spec/ipv6-provider-nets-slaac ? ), the instances will be able to use it as its own IPv6 default router automatically.

### About NAT:

My idea is to move on and forget about IPv4 and NAT tables, so, with IPv6, we don't need NAT anymore. NAT66 is a bad thing (from my point of view), be part of the real Internet with IPv6 address from the subnet 2000::/3! Again, **do not use** "ip6tables -t nat", unless you want to break your network, and the Internet itself.

One last word about NAT: it breaks the end-to-end Internet connectivity, effectively kicking you out from the real Internet, and it is just a workaround created to deal with the IPv4 exhaustion, so, there is no need for NAT(66) on an IPv6-World, for real...

### About config files:

*NOTE #1: The config file examples are supposed to be added to the respective files, it is not a entire config file replacement. Keep the rest of the original files intact when possible (i.e. when not duplicating the entries).*

*NOTE #2: You might want to replace the domain `yourdomain.com` with your own real domain.*

---

# *Outside of the Cloud Computing*

# 1. First things first, the border gateway

This lab have a Ubuntu acting as a firewall, with our WAN ISP attached to it, so, behind it, will sit the entire OpenStack infrastructure.

This Firewall Ubuntu might have the package **aiccu** installed, so, you'll have at least, one IPv6 /64 block to play with (if you don't have it native from your ISP, get one from SixxS.net and start using it **aiccu**).

Also, if you go with IPv6, you'll probably need the package **radvd** installed here, so, you'll be able to advertise your IPv6 blocks within your LAN. And, for the Ubuntu IPv6 clients (including future IPv6-Only Instances), you'll also need the package **rdnssd** to auto-configure the Instance's /etc/resolv.conf file according.

For IPv6, this Ubuntu gateway will act as the "Provider Networking - upstream SLAAC" router (I'm researching about this), so, we'll be able to start testing (probably) the blueprint called "ipv6-provider-nets-slaac".

## 1.1. Gateway (Ubuntu 14.04)

Install a Ubuntu 14.04 with at least two network cards (can be a small virtual machine).

* Network Topology:

* WAN - eth0

    * IPv6 (If you have native)

        * IP address: 2001:db8:0::2/64
        * Gateway IP: 2001:db8:0::1

    * IPv4

        * IP address: 200.10.1.2/28
        * Gateway IP: 200.10.1.1

* LAN - eth1

    * IPv6 (From SixxS.net, for example)

        * IP addresses: 2001:db8:1::1/64 (Management + Instances)

    * IPv4 (Legacy)

        * IP addresses: 10.32.14.1/24 (Management + Instances)

## 1.2. Example of its /etc/network/interfaces file:

    # The loopback network interface
    auto lo
    iface lo inet loopback
    iface lo inet6 loopback

    # ETH0 - BEGIN - WAN faced
    # The primary network interface connected to your ISP's WAN
    auto eth0

    # IPv6
    #
    # If you have native IPv6, configure it here, otherwise, aiccu will create
    # a new interface for your IPv6 WAN, called sixxs, tunneled through your
    # eth0 IPv4 address.
    iface eth0 inet6 static
        address 2001:db8:0::2
        netmask 64
        gateway 2001:db8:0::1
        # dns-* options are implemented by the resolvconf package, if installed
        dns-search yourdomain.com
        dns-domain yourdomain.com
        # Google Public DNS
        dns-nameservers 2001:4860:4860::8844 2001:4860:4860::8888
        # OpenNIC 
    #    dns-nameservers 2001:530::216:3cff:fe8d:e704 2600:3c00::f03c:91ff:fe96:a6ad 2600:3c00::f03c:91ff:fe96:a6ad
        # OpenDNS Public Name Servers:
    #    dns-nameservers 2620:0:ccc::2 2620:0:ccd::2

    # IPv4 - Legacy
    #
    iface eth0 inet static
    	address 200.10.1.2
    	netmask 28
        gateway 200.10.1.1
        # Google Public DNS
    #	dns-nameservers 8.8.4.4
        # OpenDNS
    #   dns-nameservers 208.67.222.222 208.67.220.220 208.67.222.220 208.67.220.222
        # OpenNIC
    #   dns-nameservers 66.244.95.20 74.207.247.4 216.87.84.211
    # ETH0 - END

    # ETH1 - BEGIN - LAN faced
    auto eth1

    # IPv6
    #
    # Your routed block, SixxS.net or TunnelBroker provides one for you, for free.
    # You might want to run "radvd" on eth1 to advertise it to your LAN / Cloud.
    #
    # OpenStack Management + Instance's gateway
    iface eth1 inet6 static
    	address 2001:db8:1::1
        netmask 64

    # IPv4 - Legacy
    #
    # OpenStack Management + Instance's gateway
    iface eth1 inet static
    	address 10.32.14.1
    	netmask 24
    # ETH1 - END

## 1.3. Enable IPv6 / IPv4 packet forwarding

Run the following commands

    sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
    sed -i 's/#net.ipv6.conf.all.forwarding=1/net.ipv6.conf.all.forwarding=1/' /etc/sysctl.conf
    sysctl -p

## 1.4. Upstream IPv6 Router Advertisement (SLAAC)

Note: OpenStack isn't compatible with any upstream SLAAC router right now, so, this is optional.

Example of its /etc/radvd.conf file:

    # With IPv6, the DHCP(v6) is entirely optional, now, we have the
    # Router Advertisement daemon, called radvd. So, we'll not use DHCPv6.
    #
    # Ubuntu, "Provider Networking - upstream SLAAC" router for Instances.
    #
    # eth1 - LAN faced:
    interface eth1
    {

        # Enable RA on this gateway:
        AdvSendAdvert on;
 
        # Disable clients from getting their IPs from DHCPv6 (no need for it):
        AdvManagedFlag off;

        # Disable clients from getting other configs from DHCPv6:
        AdvOtherConfigFlag off;
 
        # More options:
        AdvLinkMTU 1500;
        AdvDefaultPreference high;

        # Enable RA to the following subnet
        prefix 2001:db8:1::/64
        {
           AdvOnLink on;
 
           # Allow clients to auto-generate their IPv6 address (SLAAC),
           # DHCPv6 will not be used here.
           AdvAutonomous on;
        };

        # Ubuntu IPv6 clients requires the package rdnssd to deal with RDNSS: 
        RDNSS 2001:4860:4860::8844 2001:4860:4860::8888 { };
        DNSSL yourdomain.com { };

    };

Install the following package (RA Daemon):

    sudo apt-get install radvd

Now your entire LAN have IPv6! Have fun!

## 1.5. NAT rule (Legacy)

    iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

*NOTE #1: There is only 1 NAT rule on this environment, which resides on this gateway itself, to do the IPv4 SNAT/DNAT to/from the old Internet infrastructure. There is no IPv4 NAT within this OpenStack environment itself (no Floating IPs, "no multihost=true"). Also, there is no NAT when enjoying the New Internet Powered by IPv6!*

*NOTE #2: If your have more IPv4 public blocks available (i.e. at your gateway's eth1 interface, your Instances can also have public IPs on it!*

---

# *Inside of the Cloud Computing*

# 2. Deploying the Controller Node

The OpenStack Controller Node is powered by Ubuntu 14.04!

* Requirements:

    * 1 Virtual Machine (KVM/Xen) with 2G of RAM
    * 1 Virtual Ethernet VirtIO Card
    * 2 Virtual VirtIO HDs about 100G each (one for Ubuntu / Nova / Glance and another for Cinder)
    * Hostname: controller.yourdomain.com
    * 64 bits O.S. Recommended

* IPv6

    * IP Address: 2001:db8:1::10/64
    * Gateway IP: 2001:db8:1::1

* IPv4 - Legacy

    * IP Address: 10.32.14.10/24
    * Gateway IP: 10.32.14.1

Install Ubuntu 14.04 on the first disk, can be the  *Minimum Virtual Machine* flavor, using *Guided LVM Paritioning*, leave the second disk untouched for now (it will be used with Cinder).

## 2.1. Prepare Ubuntu O.S.

Login as root and run:

    echo controller > /etc/hostname

    apt-get update
    
    apt-get dist-upgrade -y

    apt-get install vim iptables openvswitch-switch

    vi /etc/hosts

Make sure it have the following contents:

    127.0.0.1       localhost.localdomain   localhost

    # IPv6
    2001:db8:1::10  controller.yourdomain.com   controller
    2001:db8:1::20  compute-1.yourdomain.com   compute-1
    2001:db8:1::30  compute-2.yourdomain.com   compute-2

    # IPv4 - Not needed:
    #10.32.14.10    controller.yourdomain.com   controller
    #10.32.14.20    compute-1.yourdomain.com   compute-1
    #10.32.14.30    compute-2.yourdomain.com   compute-2

## 2.2. Configure the network

Edit your Controller Node network interfaces file:

    vi /etc/network/interfaces

With:

    # The primary network interface
    # ETH0 - BEGIN
    auto eth0
    iface eth0 inet manual
    	up ip link set $IFACE up
    	up ip address add 0/0 dev $IFACE
    	down ip link set $IFACE down
    # ETH0 - END

    # BR-ETH0 - BEGIN
    auto br-eth0

    # IPv6
    #
    iface br-eth0 inet6 static
        address 2001:db8:1::10
        netmask 64
        gateway 2001:db8:1::1
        # dns-* options are implemented by the resolvconf package, if installed
    	dns-search yourdomain.com
    	dns-domain yourdomain.com
        # Google Public DNS
        dns-nameservers 2001:4860:4860::8844 2001:4860:4860::8888
        # OpenNIC
    #    dns-nameservers 2001:530::216:3cff:fe8d:e704 2600:3c00::f03c:91ff:fe96:a6ad 2600:3c00::f03c:91ff:fe96:a6ad
        # OpenDNS Public Name Servers
    #    dns-nameservers 2620:0:ccc::2 2620:0:ccd::2

    # IPv4 - Legacy
    #
    iface br-eth0 inet static
    	address 10.32.14.10
    	netmask 24
    	gateway 10.32.14.1
        # Google Public DNS
    	dns-nameservers 8.8.4.4
        # OpenDNS
    #    dns-nameservers 208.67.222.222 208.67.220.220 208.67.222.220 208.67.220.222
        # OpenNIC
    #    dns-nameservers 66.244.95.20 74.207.247.4 216.87.84.211
    # BR-ETH0 - END

Login as root and run:

    ovs-vsctl add-br br-int

    ovs-vsctl add-br br-eth0

The next OVS command will kick you out from this server (if connected to it via eth0), that's why we should reboot after running it:

    ovs-vsctl add-port br-eth0 eth0 && reboot

## 2.3 Install OpenStack "base" dependecies

    apt-get install mysql-server python-mysqldb ntp curl openssl rabbitmq-server python-keyring

Configure it:

Replace RABBIT_PASS with a suitable password.

    rabbitmqctl change_password guest guest

Reconfigure MySQL, edit my.cnf:

    vi /etc/mysql/my.cnf

With:

    [mysqld]
    #
    # * For OpenStack - Keystone, etc - utf8
    #
    default-storage-engine = innodb
    collation-server = utf8_general_ci
    init-connect='SET NAMES utf8'
    character-set-server = utf8
    innodb_file_per_table

    # Bind in a Dual-Stacked fashion
    bind-address = ::

Creating the required databases:

    service mysql restart

Make your MySQL a bit safer:

    mysql_install_db

    mysql_secure_installation

Now make the required databases:

    mysql -u root -p

Once within MySQL prompt, create the databases:

    CREATE DATABASE keystone;
    GRANT ALL ON keystone.* TO 'keystoneUser'@'%' IDENTIFIED BY 'keystonePass';
    CREATE DATABASE glance;
    GRANT ALL ON glance.* TO 'glanceUser'@'%' IDENTIFIED BY 'glancePass';
    CREATE DATABASE nova;
    GRANT ALL ON nova.* TO 'novaUser'@'%' IDENTIFIED BY 'novaPass';
    CREATE DATABASE cinder;
    GRANT ALL ON cinder.* TO 'cinderUser'@'%' IDENTIFIED BY 'cinderPass';
    CREATE DATABASE neutron;
    GRANT ALL ON neutron.* TO 'neutronUser'@'%' IDENTIFIED BY 'neutronPass';
    CREATE DATABASE heat;
    GRANT ALL ON heat.* TO 'heatUser'@'%' IDENTIFIED BY 'heatPass';
    quit;

### Document references

 * http://docs.openstack.org/icehouse/install-guide/install/apt/content/basics-database-controller.html

## 2.4. Install Keystone

    apt-get install keystone

Edit the keystone.conf and and change it for this:

    vi /etc/keystone/keystone.conf

File contents:

    [DEFAULT]
    admin_token = ADMIN

    bind_host = 2001:db8:1::10

    [database]
    connection = mysql://keystoneUser:keystonePass@controller.yourdomain.com/keystone

Then run:

    rm /var/lib/keystone/keystone.db

    su -s /bin/sh -c "keystone-manage db_sync" keystone

    service keystone restart

Create Keystone basics and endpoints:

    cd ~

    wget https://gist.githubusercontent.com/tmartinx/7002197/raw/838770e4848c78dcd896fcfb6e4627d754051a72/keystone_basic.sh

    wget https://gist.githubusercontent.com/tmartinx/7002255/raw/b13b1a09a675ff8c680545afe83b946cb03a36a5/keystone_endpoints_basic.sh

    chmod +x keystone_basic.sh

    chmod +x keystone_endpoints_basic.sh

    ./keystone_basic.sh

    ./keystone_endpoints_basic.sh

Preliminary Keystone test

    curl http://controller.yourdomain.com:35357/v2.0/endpoints -H 'x-auth-token: ADMIN' | python -m json.tool

You might want to cleanup your expired tokens, otherwise, your database will increase in size indefinitely. So, do this:

    (crontab -l 2>&1 | grep -q token_flush) || echo '@hourly /usr/bin/keystone-manage token_flush >/var/log/keystone/keystone-tokenflush.log 2>&1' >> /var/spool/cron/crontabs/root

Create your NOVA Resource Configuration file:

    vi ~/.novarc

With:

    # COMMON OPENSTACK ENVS
    export SERVICE_TOKEN=ADMIN
    export OS_USERNAME=admin
    export OS_PASSWORD=admin_pass
    export OS_TENANT_NAME=admin
    export OS_AUTH_URL="http://controller.yourdomain.com:5000/v2.0/"
    export SERVICE_ENDPOINT="http://controller.yourdomain.com:35357/v2.0/"
    export OS_AUTH_STRATEGY=keystone
    export OS_NO_CACHE=1

    # LEGACY NOVA ENVS
    export NOVA_USERNAME=${OS_USERNAME}
    export NOVA_PROJECT_ID=${OS_TENANT_NAME}
    export NOVA_PASSWORD=${OS_PASSWORD}
    export NOVA_API_KEY=${OS_PASSWORD}
    export NOVA_URL=${OS_AUTH_URL}
    export NOVA_VERSION=1.1
    export NOVA_REGION_NAME=RegionOne

    # EUCA2OOLs ENV VARIABLES
    export EC2_ACCESS_KEY=ab2f155901fb4be5bae4ddc78c924665
    export EC2_SECRET_KEY=ef89b9562e9b4653a8d68e3117f0ae32
    export EC2_URL=http://controller.yourdomain.com:8773/services/Cloud

Append to your bashrc:

    vi ~/.bashrc

With:

    if [ -f ~/.novarc ]; then
        . ~/.novarc
    fi

Then, load it:

    source ~/.bashrc

Test Keystone with basic option to see if it works:

    keystone tenant-list

### Document references

 * http://docs.openstack.org/icehouse/install-guide/install/apt/content/keystone-install.html

# 3. Install Glance

Lets install Glance

    apt-get install glance python-mysqldb

## 3.1. Configure Glance API

Edit glance-api.conf...

    vi /etc/glance/glance-api.conf

With:

    [DEFAULT]
    bind_host = 2001:db8:1::10
    
    registry_host = controller.yourdomain.com

    rabbit_host = controller.yourdomain.com

    [database]
    connection = mysql://glanceUser:glancePass@controller.yourdomain.com/glance

    [keystone_authtoken]
    auth_uri = http://controller.yourdomain.com:5000
    auth_host = controller.yourdomain.com
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = service
    admin_user = glance
    admin_password = service_pass
    
    [paste_deploy]
    flavor = keystone

## 3.2. Configure Glance Registry

Edit glance-registry.conf...

    vi /etc/glance/glance-registry.conf

With:

    [DEFAULT]
    bind_host = 2001:db8:1::10

    [database]
    connection = mysql://glanceUser:glancePass@controller.yourdomain.com/glance

    [keystone_authtoken]
    auth_uri = http://controller.yourdomain.com:5000
    auth_host = controller.yourdomain.com
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = service
    admin_user = glance
    admin_password = service_pass
    
    [paste_deploy]
    flavor = keystone

Then run:

    rm /var/lib/glance/glance.sqlite

    su -s /bin/sh -c "glance-manage db_sync" glance

    service glance-api restart; service glance-registry restart

## 3.3. Adding O.S. images into your Glance

Run the following commands to add some O.S. images into your Glance repository.

### 3.3.1. CirrOS (Optional - TestVM):

    glance image-create --location http://download.cirros-cloud.net/0.3.2/cirros-0.3.2-i386-disk.img --name "CirrOS 0.3.2 - Minimalist - 32-bit - Cloud Based Image" --is-public true --container-format bare --disk-format qcow2

    glance image-create --location http://download.cirros-cloud.net/0.3.2/cirros-0.3.2-x86_64-disk.img --name "CirrOS 0.3.2 - Minimalist - 64-bit - Cloud Based Image" --is-public true --container-format bare --disk-format qcow2

### 3.3.2. Ubuntu 13.10:

    glance image-create --location http://uec-images.ubuntu.com/releases/13.10/release/ubuntu-13.10-server-cloudimg-i386-disk1.img --is-public true --disk-format qcow2 --container-format bare --name "Ubuntu 13.10 - Saucy Salamander - 32-bit - Cloud Based Image"

    glance image-create --location http://uec-images.ubuntu.com/releases/13.10/release/ubuntu-13.10-server-cloudimg-amd64-disk1.img --is-public true --disk-format qcow2 --container-format bare --name "Ubuntu 13.10 - Saucy Salamander - 64-bit - Cloud Based Image"

### 3.3.3. Ubuntu 12.04.4 - LTS:

    glance image-create --location http://uec-images.ubuntu.com/releases/12.04/release/ubuntu-12.04-server-cloudimg-i386-disk1.img --is-public true --disk-format qcow2 --container-format bare --name "Ubuntu 12.04.4 LTS - Precise Pangolin - 32-bit - Cloud Based Image"

    glance image-create --location http://uec-images.ubuntu.com/releases/12.04/release/ubuntu-12.04-server-cloudimg-amd64-disk1.img --is-public true --disk-format qcow2 --container-format bare --name "Ubuntu 12.04.4 LTS - Precise Pangolin - 64-bit - Cloud Based Image"

### 3.3.4. Ubuntu 14.04 - LTS:

    glance image-create --location http://uec-images.ubuntu.com/releases/14.04/release/ubuntu-14.04-server-cloudimg-i386-disk1.img --is-public true --disk-format qcow2 --container-format bare --name "Ubuntu 14.04 LTS - Trusty Tahr - 32-bit - Cloud Based Image"

    glance image-create --location http://uec-images.ubuntu.com/releases/14.04/release/ubuntu-14.04-server-cloudimg-amd64-disk1.img --is-public true --disk-format qcow2 --container-format bare --name "Ubuntu 14.04 LTS - Trusty Tahr - 64-bit - Cloud Based Image"

### 3.3.5. Ubuntu 14.10:

    glance image-create --location http://uec-images.ubuntu.com/releases/14.10/beta-1/ubuntu-14.10-beta1-server-cloudimg-i386.tar.gz --is-public true --disk-format qcow2 --container-format bare --name "Ubuntu 14.10 - Utopic Unicorn - 32-bit - Cloud Based Image"

    glance image-create --location http://uec-images.ubuntu.com/releases/14.10/beta-1/ubuntu-14.10-beta1-server-cloudimg-amd64.tar.gz --is-public true --disk-format qcow2 --container-format bare --name "Ubuntu 14.10 - Utopic Unicorn - 64-bit - Cloud Based Image"

### 3.3.5. CoreOS 438.0.0 - Linux 3.16.2 - Docker 1.2.0 - etcd 0.4.6 - fleet 0.8.0

Info: https://coreos.com

    cd ~

    wget http://alpha.release.core-os.net/amd64-usr/current/coreos_production_openstack_image.img.bz2

    bunzip2 coreos_production_openstack_image.img.bz2

    glance image-create --name "CoreOS 438.0.0 - Linux 3.16.2 - Docker 1.2.0 - etcd 0.4.6 - fleet 0.8.0" --container-format ovf --disk-format qcow2 --file coreos_production_openstack_image.img --is-public True

### 3.3.6. Windows 2012r2:

If you need to run Windows 2012 in your OpenStack, visit: http://cloudbase.it/ws2012r2 to download the image "windows_server_2012_r2_standard_eval_kvm_20140607.qcow2.gz", then, run:

    gunzip /root/windows_server_2012_r2_standard_eval_kvm_20140607.qcow2.gz

    glance image-create --name "Windows Server 2012 R2 Standard Eval" --container-format bare --disk-format qcow2 --is-public true < /root/windows_server_2012_r2_standard_eval_kvm_20140607.qcow2

### 3.3.6. Listing the O.S. images:
    
    glance image-list


### Document references

 * http://docs.openstack.org/icehouse/install-guide/install/apt/content/glance-install.html

# 4. Install Nova

Run:

    apt-get install python-novaclient nova-api nova-cert nova-consoleauth nova-scheduler nova-conductor nova-spiceproxy

## 4.1 Configure Nova

Run:

    cd /etc/nova

    mv /etc/nova/nova.conf /etc/nova/nova.conf_Ubuntu

    wget https://gist.githubusercontent.com/tmartinx/10784491/raw/384356d1b072d1a65f4c9175a26e4154e4d97079/nova.conf

    chown nova: /etc/nova/nova.conf

    chmod 640 /etc/nova/nova.conf

*NOTE: Edit your nova.conf file, before running "db sync", to reflect your own FQDN (*.yourdomain.com), if desired.*

    rm /var/lib/nova/nova.sqlite

    su -s /bin/sh -c "nova-manage db sync" nova

I figured out that *Nova SPICE Proxy* doesn't listen on a Dual-Stacked setup, even after configuring it at nova.conf, something is wrong with it (BUG LP #1308418), so, just patch the spicehtml2proxy.py file to force it listen on both IPv4 and IPv6, like this:

    sed -i 's/0.0.0.0/::/' /usr/lib/python2.7/dist-packages/nova/cmd/spicehtml5proxy.py

Now, you can restart all Nova services:

    cd /etc/init/; for i in $(ls nova-* | cut -d \. -f 1 | xargs); do sudo service $i restart; done

## 4.2. Personalize your flavors (optional):

Delete the default examples:

    nova flavor-delete 1
    nova flavor-delete 2
    nova flavor-delete 3
    nova flavor-delete 4
    nova flavor-delete 5

Create new flavors:

    # Standard Flavor
    nova flavor-create --ephemeral 0 --swap 128 --rxtx-factor 1.0 --is-public yes m1.micro 1 256 5 1
    nova flavor-create --ephemeral 25 --swap 256 --rxtx-factor 1.0 --is-public yes m1.tiny 2 512 10 1
    nova flavor-create --ephemeral 50 --swap 512 --rxtx-factor 1.0 --is-public yes m1.small 3 1024 10 1
    nova flavor-create --ephemeral 100 --swap 1024 --rxtx-factor 1.0 --is-public yes m1.medium 4 2048 10 2
    nova flavor-create --ephemeral 200 --swap 2048 --rxtx-factor 1.0 --is-public yes m1.large 5 4096 10 4
    nova flavor-create --ephemeral 400 --swap 4096 --rxtx-factor 1.0 --is-public yes m1.xlarge 6 8192 10 8

    # RAM Memory optimized
    nova flavor-create --ephemeral 0 --swap 256 --rxtx-factor 1.0 --is-public yes r1.micro 7 512 5 1
    nova flavor-create --ephemeral 25 --swap 512 --rxtx-factor 1.0 --is-public yes r1.tiny 8 1024 10 1
    nova flavor-create --ephemeral 50 --swap 1024 --rxtx-factor 1.0 --is-public yes r1.small 9 2048 10 1
    nova flavor-create --ephemeral 100 --swap 2048 --rxtx-factor 1.0 --is-public yes r1.medium 10 4096 10 2
    nova flavor-create --ephemeral 200 --swap 4096 --rxtx-factor 1.0 --is-public yes r1.large 11 8192 10 4
    nova flavor-create --ephemeral 400 --swap 8192 --rxtx-factor 1.0 --is-public yes r1.xlarge 12 16384 10 8

    # Storage optimized 
    nova flavor-create --ephemeral 50 --swap 256 --rxtx-factor 1.0 --is-public yes s1.tiny 13 512 10 1
    nova flavor-create --ephemeral 100 --swap 512 --rxtx-factor 1.0 --is-public yes s1.small 14 1024 10 1
    nova flavor-create --ephemeral 200 --swap 1024 --rxtx-factor 1.0 --is-public yes s1.medium 15 2048 20 2
    nova flavor-create --ephemeral 400 --swap 2048 --rxtx-factor 1.0 --is-public yes s1.large 16 4096 40 4
    nova flavor-create --ephemeral 800 --swap 4096 --rxtx-factor 1.0 --is-public yes s1.xlarge 17 8192 80 8

    # Windows optimized
    nova flavor-create --ephemeral 100 --swap 0 --rxtx-factor 1.0 --is-public yes w1.small 18 1024 20 1
    nova flavor-create --ephemeral 200 --swap 0 --rxtx-factor 1.0 --is-public yes w1.medium 19 2048 20 2
    nova flavor-create --ephemeral 400 --swap 0 --rxtx-factor 1.0 --is-public yes w1.large 20 4096 40 4
    nova flavor-create --ephemeral 800 --swap 0 --rxtx-factor 1.0 --is-public yes w1.xlarge 21 8192 40 8

### Document references

 * http://docs.openstack.org/icehouse/install-guide/install/apt/content/nova-controller.html

# 5. Install Neutron

Run:

    apt-get install neutron-server neutron-plugin-ml2 neutron-plugin-openvswitch-agent neutron-dhcp-agent neutron-metadata-agent

## 5.1. Configure Neutron

First, take a note of the "Service Tenant ID" with:

    keystone tenant-get service

Edit neutron.conf...

    vi /etc/neutron/neutron.conf

With:

    [DEFAULT]
    bind_host = 2001:db8:1::10
    auth_strategy = keystone
    allow_overlapping_ips = True
    rabbit_host = controller.yourdomain.com
    
    notify_nova_on_port_status_changes = True
    notify_nova_on_port_data_changes = True
    nova_url = http://controller.yourdomain.com:8774/v2
    nova_region_name = RegionOne
    nova_admin_username = nova
    nova_admin_tenant_id = $SERVICE_TENANT_ID
    nova_admin_password = service_pass
    nova_admin_auth_url = http://controller.yourdomain.com:35357/v2.0

    [keystone_authtoken]
    auth_uri = http://controller.yourdomain.com:5000
    auth_host = controller.yourdomain.com
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = service
    admin_user = neutron
    admin_password = service_pass
    signing_dir = $state_path/keystone-signing
    
    [database]
    connection = mysql://neutronUser:neutronPass@controller.yourdomain.com/neutron

Edit ml2_conf.ini...

    vi /etc/neutron/plugins/ml2/ml2_conf.ini

With:

    [ml2]
    type_drivers = local,flat
    
    mechanism_drivers = openvswitch,l2population
    
    [ml2_type_flat]
    flat_networks = *
    
    [securitygroup]
    enable_security_group = True
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

    [ovs]
    enable_tunneling = False
    local_ip = 10.32.14.10
    network_vlan_ranges = physnet1
    bridge_mappings = physnet1:br-eth0

Edit metadata_agent.ini...
    
    vi /etc/neutron/metadata_agent.ini

With:

    # The Neutron user information for accessing the Neutron API.
    auth_url = http://controller.yourdomain.com:5000/v2.0
    auth_region = RegionOne
    admin_tenant_name = service
    admin_user = neutron
    admin_password = service_pass
    
    nova_metadata_ip = 10.32.14.10
    nova_metadata_port = 8775
    metadata_proxy_shared_secret = metasecret13

Edit dhcp_agent.ini...

    vi /etc/neutron/dhcp_agent.ini

With:

    [DEFAULT]
    interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver

    use_namespaces = True

    enable_isolated_metadata = True

    dhcp_domain = yourdomain.com

Run:

    cd /etc/init/; for i in $(ls -1 neutron-* | cut -d \. -f 1); do sudo service $i restart; done

## 5.2. Create the OpenStack Neutron Network

First, get the admin tenant id and note it (like var $ADMIN_TENTANT_ID).

    keystone tenant-list

### 5.2.1. Creating the Flat Neutron Network

Previous versions of this Quick Guide, had two IPv4 subnets, one for OpenStack Management and Physical Serves (10.32.14.0/24), and another for the Instances (10.33.14.0/24). But now, we have only one IPv4 subnet for everything (which is 10.32.14.0/24), for both Openstack Management and Instances, less subnets to deal with, so, the subdivision of the IP ranges comes now from the neutron "--allocation-pool" option. This way, it will be easier to introduce IPv6.

Mapping the physical network, that one from your "border gateway", into OpenStack Neutron:
 
    neutron net-create --tenant-id $ADMIN_TENTANT_ID sharednet1 --shared --provider:network_type flat --provider:physical_network physnet1

Create an IPv4 subnet on "sharednet1":

    neutron subnet-create --ip-version 4 --tenant-id $ADMIN_TENANT_ID sharednet1 10.32.14.0/24 --allocation-pool start=10.32.14.129,end=10.32.14.254 --dns_nameservers list=true 8.8.4.4 8.8.8.8

Create an IPv6 subnet on "sharednet1":

    neutron subnet-create --ip-version 6 --disable-dhcp --tenant-id $ADMIN_TENANT_ID sharednet1 2001:db8:1::/64 --allocation-pool start=2001:db8:1::8000,end=2001:db8:1:0:ffff:ffff:ffff:fffe

### Document references

 * http://docs.openstack.org/icehouse/install-guide/install/apt/content/neutron-ml2-controller-node.html

# 6. Install Cinder

## 6.1. Cinder API / endpoint access:

    apt-get install cinder-api cinder-scheduler python-mysqldb

Edit cinder.conf...

    vi /etc/cinder/cinder.conf

with:

    [DEFAULT]

    my_ip = 2001:db8:1::10
    glance_host = 2001:db8:1::10
    osapi_volume_listen = 2001:db8:1::10

    rpc_backend = cinder.openstack.common.rpc.impl_kombu
    rabbit_host = controller.yourdomain.com

    connection = mysql://cinderUser:cinderPass@controller.yourdomain.com/cinder

    [keystone_authtoken]
    auth_uri = http://controller.yourdomain.com:5000
    auth_host = controller.yourdomain.com
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = service
    admin_user = cinder
    admin_password = service_pass

Run:

    su -s /bin/sh -c "cinder-manage db sync" cinder

    cd /etc/init/; for i in $(ls cinder-* | cut -d \. -f 1 | xargs); do sudo service $i restart; done

## 6.2. Cinder iSCSI block storage service

This procedure will make use of the extra Virtual HD of your controller.yourdomain.com (about 100G).

*If don't have it, add one: halt VM -> go to "virt-manager" -> Add hardware -> VirtIO Disk / 100G / RAW.*

Then start it again and run:

    # Create a primary partition on it, type LVM (8e)
    cfdisk /dev/vdb

    # Create the LVM Physical Volume
    pvcreate /dev/vdb1

    # Create the LVM Volume Group
    vgcreate cinder-volumes /dev/vdb1

Install Cinder Volume:

    apt-get install cinder-volume

### Document references

 * http://docs.openstack.org/icehouse/install-guide/install/apt/content/cinder-controller.html

# 7. Install Horizon Dashboard

Run:

    apt-get install openstack-dashboard memcached

    apt-get purge openstack-dashboard-ubuntu-theme

Edit Dashboard config file:

    vi /etc/openstack-dashboard/local_settings.py

With:

    OPENSTACK_HOST = "controller.yourdomain.com"

Done! You can try to access the Dashboard to test admin login...

### Document references

 * http://docs.openstack.org/icehouse/install-guide/install/apt/content/install_dashboard.html

---

# 8. Deploying your first Compute Node

This OpenStack Compute Node is powered by Ubuntu 14.04!

* Requirements:1 Physical Server with Virtualization support on CPU, 1 ethernet

    * 1 Physical Server with Virtualization support on CPU
    * 1 Ethernet Card
    * 1 HardDisk about 500G
    * Hostname: compute-1.yourdomain.com
    * 64 bits O.S. highly recommended

* IPv6

    * IP address: 2001:db8:1::20/64
    * Gateway IP: 2001:db8:1::1

* IPv4 - Legacy

    * IP address: 10.32.14.20/24
    * Gateway IP: 10.32.14.1

## 8.1. Install Ubuntu 14.04

This installation can be the "Minimum Installation" flavor, using `Manual Paritioning', make the following partitions:

* /dev/sda1 on /boot (~256M - /dev/md0 if raid1[0], bootable)
* /dev/sda2 on LVM VG vg01 (~50G - /dev/md1 if raid1[0]) - lv_root (25G), lv_swap (XG) of compute-1
* /dev/sda3 on LVM VG nova-local (~450G - /dev/md2 if raid1[0]) - Instances

Login as root and run:

    echo compute-1 > /etc/hostname

    apt-get update
    
    apt-get dist-upgrade -y

    # If your kernel gets upgraded, do a reboot before running the next command:

    apt-get install linux-image-extra-`uname -r` vim iptables ubuntu-virt-server libvirt-bin pm-utils nova-compute-kvm python-guestfs neutron-plugin-openvswitch-agent openvswitch-switch -y

When prompted to create a *supermin* appliance, respond **yes**.

make the current kernel readable (BUG LP #759725):

    dpkg-statoverride --update --add root root 0644 /boot/vmlinuz-$(uname -r)

    virsh net-destroy default

    virsh net-undefine default

## 8.2. Configure your Ubuntu KVM Hypervisor

Run:

    # Add vhost_net module to be enabled during server boot:
    echo vhost_net >> /etc/modules

    # Prepare /etc/libvirt/libvirtd.conf:
    sed -i 's/^#listen_tls = 0/listen_tls = 0/' /etc/libvirt/libvirtd.conf

    sed -i 's/^#listen_tcp = 1/listen_tcp = 1/' /etc/libvirt/libvirtd.conf
    
    sed -i 's/^#auth_tcp = "sasl"/auth_tcp = "none"/' /etc/libvirt/libvirtd.conf

    # Prepare /etc/init/libvirt-bin.conf:
    sed -i 's/^env libvirtd_opts="-d"/env libvirtd_opts="-d -l"/' /etc/init/libvirt-bin.conf

    # Prepare /etc/default/libvirt-bin:
    sed -i 's/^libvirtd_opts="-d"/libvirtd_opts="-d -l"/' /etc/default/libvirt-bin

## 8.3. Configure the network

Edit:

    vi /etc/hosts

With:

    127.0.0.1       localhost.localdomain   localhost

    # IPv6
    2001:db8:1::10  controller.yourdomain.com   controller
    2001:db8:1::20  compute-1.yourdomain.com   compute-1
    2001:db8:1::30  compute-2.yourdomain.com   compute-2

    # IPv4 - Not needed:
    #10.32.14.10    controller.yourdomain.com   controller
    #10.32.14.20    compute-1.yourdomain.com   compute-1
    #10.32.14.30    compute-2.yourdomain.com   compute-2

Edit:

    vi /etc/network/interfaces

With:

    # The primary network interface
    # ETH0 - BEGIN
    auto eth0
    iface eth0 inet manual
            up ip link set $IFACE up
            up ip address add 0/0 dev $IFACE
            down ip link set $IFACE down
    # ETH0 - END

    # BR-ETH0 - BEGIN
    auto br-eth0

    # IPv6
    iface br-eth0 inet6 static
    	address 2001:db8:1::20
    	netmask 64
        gateway 2001:db8:1::1
    	# dns-* options are implemented by the resolvconf package, if installed
    	dns-domain yourdomain.com
        dns-search yourdomain.com
        # Google Public DNS
        dns-nameservers 2001:4860:4860::8844 2001:4860:4860::8888
        # OpenNIC 
    #    dns-nameservers 2001:530::216:3cff:fe8d:e704 2600:3c00::f03c:91ff:fe96:a6ad 2600:3c00::f03c:91ff:fe96:a6ad
        # OpenDNS Public Name Servers:
    #    dns-nameservers 2620:0:ccc::2 2620:0:ccd::2

    # IPv4 - Legacy
    iface br-eth0 inet static
        address 10.32.14.20
        netmask 24
        gateway 10.32.14.1
        # Google Public DNS
    	dns-nameservers 8.8.4.4
        # OpenDNS
    #    dns-nameservers 208.67.222.222 208.67.220.220 208.67.222.220 208.67.220.222
        # OpenNIC
    #    dns-nameservers 66.244.95.20 74.207.247.4 216.87.84.211
    # BR-ETH0 - END

Run:

    ovs-vsctl add-br br-int

    ovs-vsctl add-br br-eth0

The next OVS command will kick you out from this server (if connected to it via eth0), that's why we should reboot after running it:

    ovs-vsctl add-port br-eth0 eth0 && reboot

## 8.4. Configure Nova:

Run:

    mv /etc/nova/nova.conf /etc/nova/nova.conf_Ubuntu

    cd /etc/nova

    wget https://gist.githubusercontent.com/tmartinx/10784896/raw/8088aee54877caca18c3020f91b662cdea627213/nova.conf

    chown nova: /etc/nova/nova.conf

    chmod 640 /etc/nova/nova.conf

    cd /etc/init/; for i in $(ls nova-* | cut -d \. -f 1 | xargs); do sudo service $i restart; done

### Document references

 * http://docs.openstack.org/icehouse/install-guide/install/apt/content/nova-compute.html

## 8.5. Install Neutron

NOTE: Run this command on compute-1.yourdomain.com, not on controller.yourdomain.com.

Edit:

    vi /etc/neutron/neutron.conf

With:

    [DEFAULT]
    # debug = True
    # verbose = True
    allow_overlapping_ips = True
    rabbit_host = controller.yourdomain.com
    
    notify_nova_on_port_status_changes = True
    notify_nova_on_port_data_changes = True
    nova_url = http://controller.yourdomain.com:8774/v2
    nova_region_name = RegionOne
    nova_admin_username = nova
    nova_admin_tenant_id = $SERVICE_TENANT_ID
    nova_admin_password = service_pass
    nova_admin_auth_url = http://controller.yourdomain.com:35357/v2.0

    [keystone_authtoken]
    auth_uri = http://controller.yourdomain.com:5000
    auth_host = controller.yourdomain.com
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = service
    admin_user = neutron
    admin_password = service_pass
    signing_dir = /var/lib/neutron/keystone-signing

    [database]
    connection = mysql://neutronUser:neutronPass@controller.yourdomain.com/neutron

Edit:

    vi /etc/neutron/plugins/ml2/ml2_conf.ini

With:

    [ml2]
    type_drivers = local,flat
    
    mechanism_drivers = openvswitch,l2population
    
    [ml2_type_flat]
    flat_networks = *
    
    [securitygroup]
    enable_security_group = True
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

    [ovs]
    enable_tunneling = False
    local_ip = 10.32.14.20
    network_vlan_ranges = physnet1
    bridge_mappings = physnet1:br-eth0

Run:

    service neutron-plugin-openvswitch-agent restart

### Document references

 * http://docs.openstack.org/icehouse/install-guide/install/apt/content/neutron-ml2-compute-node.html

# 9. Creating your first Dual-Stacked Instance

Now, go back to the node `controller.yourdomain.com` and run the following commands as root:

Show the O.S. images to get the IDs:

    glance image-list

Boot your Ubuntu 14.04 - 32-bit fits better on m1.micro:

    nova boot --image $your_ubuntu_14_04_lts_image_id --key-name my_ssh_key --flavor 1 ubuntu-1

The above command will create your Instance but, the IPv6 address will not be configured automatically within it, so, do the following steps to enable it:

Get Instance's info:

    nova list

Something like this will appear:

    +--------------------------------------+-------------+--------+------------+-------------+----------------------------------------------+
    | ID                                   | Name        | Status | Task State | Power State | Networks                                     |
    +--------------------------------------+-------------+--------+------------+-------------+----------------------------------------------+
    | 0460d770-372a-4549-80db-ccbafddda22c | ubuntu-1    | ACTIVE | -          | Running     | sharednet1=2001:db8:1::8000, 10.32.14.130    |
    +--------------------------------------+-------------+--------+------------+-------------+----------------------------------------------+

Go there and configure the IPv6 statically:

    ssh ubuntu@10.32.14.130
    sudo ip -6 a a 2001:db8:1::8000/64 dev eth0
    sudo ip -6 r a default via 2001:db8:1::1
    
Verify IPv6 connectivity:

    ubuntu@ubuntu-1:~$ ping6 -c1 google.com
    PING google.com(2800:3f0:4004:800::1001) 56 data bytes
    64 bytes from 2800:3f0:4004:800::1001: icmp_seq=1 ttl=52 time=36.2 ms

Now, to make it persistent across Instance's reboots, do this:

Edit the interfaces configuration file:

    vi /etc/network/interfaces.d/eth0.cfg

With:

    # The primary network interface
    auto eth0
    iface eth0 inet dhcp

    iface eth0 inet6 static
            address 2001:db8:1::8000
            netmask 64
            gateway 2001:db8:1::1

*NOTE: The IPv6 subnet 2001:db8:1::/64 is ONLY used for documentation purposes, it will not be routed. So, replace it with your own block, for example, the one from SixxS.net.*

# Well Done!

Point mycloud.yourdomain.com to 2001:db8:1::10 (and/or 10.32.14.10) and open
the *Horizon Dashboard* at:

http://mycloud.yourdomain.com/horizon - user admin, pass admin_pass

**Congratulations!!**

You have your own Private Cloud Computing Environment up and running! With IPv6!!

*Enjoy it!*

By Thiago Martins <thiagocmartinsc@gmail.com>