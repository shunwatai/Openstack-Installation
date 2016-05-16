# Environment setup
## install useless stuffs
	# yum install -y tmux vim tcpdump net-utils nmap telnet wget
## set static IP, DNS blablabla
	# vim /etc/sysconfig/network-scripts/ifcfg-INTERFACE_NAME
## set hostname
	# vim /etc/hosts
		# controller
		10.0.0.11       controller
		# compute1
		10.0.0.31       compute1
		# block1
		10.0.0.41       block1
		# object1
		10.0.0.51       object1
		# object2
		10.0.0.52       object2
## setup NTP server
	# yum -y install chrony
	# vim /etc/chrony.conf
		#server 0.centos.pool.ntp.org iburst
		#server 1.centos.pool.ntp.org iburst
		#server 2.centos.pool.ntp.org iburst
		#server 3.centos.pool.ntp.org iburst
		server stdtime.gov.hk iburst
		allow 192.168.238.0/24
	# systemctl restart chronyd
	# systemctl enable chronyd
## Enable the OpenStack repository
	# yum install -y centos-release-openstack-mitaka
	# yum install -y https://rdoproject.org/repos/rdo-release.rpm
	# yum upgrade
	# yum install -y python-openstackclient
	# yum install -y openstack-selinux ## install it if SELinux is enabled
## SQL Database & NoSQL database
	# yum install -y mariadb mariadb-server python2-PyMySQL
	# vim /etc/my.cnf.d/openstack.cnf
		[mysqld]
		bind-address = 192.168.238.157
		default-storage-engine = innodb
		innodb_file_per_table
		collation-server = utf8_general_ci
		character-set-server = utf8
	# systemctl enable mariadb.service
	# systemctl restart mariadb.service
	# mysql_secure_installation

	# yum install -y mongodb-server mongodb
	# vim /etc/mongod.conf
		bind_ip = 192.168.238.157
		smallfiles = true ## optional, to reduce the storage space from 1GB to 128 MB
	# systemctl enable mongod.service
	# systemctl restart mongod.service
## Message queue
	# yum install -y rabbitmq-server
	# systemctl enable rabbitmq-server.service
	# systemctl restart rabbitmq-server.service
### add openstack user
	# rabbitmqctl add_user openstack <RABBIT_PASS>
	# rabbitmqctl set_permissions openstack ".*" ".*" ".*" ## grant permissions
## Memcached
	# yum install -y memcached python-memcached
	# systemctl enable memcached.service
	# systemctl restart memcached.service

---

# Identity service - keystone
## DB setup
	# mysql -u root -p
	CREATE DATABASE keystone;
	GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY '<KEYSTONE_DBPASS>';
	GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY '<KEYSTONE_DBPASS>';
	exit
## Install and configure
you can generate admin token by ```openssl rand -hex 10``` or just type it by yourself

	# yum install -y openstack-keystone httpd mod_wsgi
	# vim /etc/keystone/keystone.conf
		[DEFAULT]
		...
		admin_token = <ADMIN_TOKEN>

		[database]
		...
		connection = mysql+pymysql://keystone:<KEYSTONE_DBPASS>@controller/keystone

		[token]
		...
		provider = fernet

	# su -s /bin/sh -c "keystone-manage db_sync" keystone
	# keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
## apache config
	# vim /etc/httpd/conf/httpd.conf
		ServerName controller
	# vim /etc/httpd/conf.d/wsgi-keystone.conf
		Listen 5000
		Listen 35357

		<VirtualHost *:5000>
			WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
			WSGIProcessGroup keystone-public
			WSGIScriptAlias / /usr/bin/keystone-wsgi-public
			WSGIApplicationGroup %{GLOBAL}
			WSGIPassAuthorization On
			ErrorLogFormat "%{cu}t %M"
			ErrorLog /var/log/httpd/keystone-error.log
			CustomLog /var/log/httpd/keystone-access.log combined

			<Directory /usr/bin>
				Require all granted
			</Directory>
		</VirtualHost>

		<VirtualHost *:35357>
			WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
			WSGIProcessGroup keystone-admin
			WSGIScriptAlias / /usr/bin/keystone-wsgi-admin
			WSGIApplicationGroup %{GLOBAL}
			WSGIPassAuthorization On
			ErrorLogFormat "%{cu}t %M"
			ErrorLog /var/log/httpd/keystone-error.log
			CustomLog /var/log/httpd/keystone-access.log combined

			<Directory /usr/bin>
				Require all granted
			</Directory>
		</VirtualHost>
	# systemctl enable httpd.service
	# systemctl restart httpd.service
## Create the service entity and API endpoints
	# export OS_TOKEN=<ADMIN_TOKEN>
	# export OS_URL=http://controller:35357/v3
	# export OS_IDENTITY_API_VERSION=3

Create the service entity for the Identity service:

	# openstack service create --name keystone --description "OpenStack Identity" identity

Create the Identity service API endpoints:

	# openstack endpoint create --region RegionOne identity public http://controller:5000/v3
	# openstack endpoint create --region RegionOne identity internal http://controller:5000/v3
	# openstack endpoint create --region RegionOne identity admin http://controller:35357/v3

## Create a domain, projects, users, and roles
1. Create the default domain:

			# openstack domain create --description "Default Domain" default

2. Create an administrative project, user, and role for administrative operations in your environment:

	- Create the admin project:

			# openstack project create --domain default --description "Admin Project" admin

	- Create the admin user:

			# openstack user create --domain default --password-prompt admin

	- Create the admin role:

			# openstack role create admin

	- Add the admin role to the admin project and user:

			# openstack role add --project admin --user admin admin

3.  This guide uses a service project that contains a unique user for each service that you add to your environment. Create the service project:

	# openstack project create --domain default --description "Service Project" service

4. Regular (non-admin) tasks should use an unprivileged project and user. As an example, this guide creates the demo project and user.

	- Create the demo project:

			# openstack project create --domain default --description "Demo Project" demo

	- Create the demo user:

			# openstack user create --domain default --password-prompt demo

	- Create the user role:

			# openstack role create user

	- Add the user role to the demo project and user:

			# openstack role add --project demo --user demo user

## Verify operation
1. For security reasons, disable the temporary authentication token mechanism:

	make a backup first:

			# cp /etc/keystone/keystone-paste.ini /etc/keystone/keystone-paste.ini.bak

   Edit the ```/etc/keystone/keystone-paste.ini``` file and remove ```admin_token_auth``` from the ```[pipeline:public_api]```, ```[pipeline:admin_api]```, and ```[pipeline:api_v3]``` sections.

2. Unset the temporary OS_TOKEN and OS_URL environment variables:

		# unset OS_TOKEN OS_URL

3. As the admin user, request an authentication token:

		# openstack --os-auth-url http://controller:35357/v3 \
		--os-project-domain-name default --os-user-domain-name default \
		--os-project-name admin --os-username admin token issue

4. As the demo user, request an authentication token:

		# openstack --os-auth-url http://controller:5000/v3 \
		--os-project-domain-name default --os-user-domain-name default \
		--os-project-name demo --os-username demo token issue

5. restore ```/etc/keystone/keystone-paste.ini``` after verify

		# cp /etc/keystone/keystone-paste.ini.bak /etc/keystone/keystone-paste.ini

## Create OpenStack client environment scripts
1. Create client environment scripts for the admin and demo projects and users. Future portions of this guide reference these scripts to load appropriate credentials for client operations.

		# cd ~
		# vim admin-openrc
			export OS_PROJECT_DOMAIN_NAME=default
			export OS_USER_DOMAIN_NAME=default
			export OS_PROJECT_NAME=admin
			export OS_USERNAME=admin
			export OS_PASSWORD=<ADMIN_PASS>
			export OS_AUTH_URL=http://controller:35357/v3
			export OS_IDENTITY_API_VERSION=3
			export OS_IMAGE_API_VERSION=2

		# vim demo-openrc
			export OS_PROJECT_DOMAIN_NAME=default
			export OS_USER_DOMAIN_NAME=default
			export OS_PROJECT_NAME=demo
			export OS_USERNAME=demo
			export OS_PASSWORD=<DEMO_PASS>
			export OS_AUTH_URL=http://controller:5000/v3
			export OS_IDENTITY_API_VERSION=3
			export OS_IMAGE_API_VERSION=2

2. Test the script:

		# source admin-openrc
		# openstack token issue

---

# Image service - glance
## Prerequisites
1. To create the database, complete these steps:

		# mysql -u root -p
		CREATE DATABASE glance;
		GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY '<GLANCE_DBPASS>';
		GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY '<GLANCE_DBPASS>';
		exit

2. Source the admin credentials to gain access to admin-only CLI commands:
		# source admin-openrc

3. To create the service credentials, complete these steps:
	- Create the glance user:

			# openstack user create --domain default --password-prompt glance

	- Add the admin role to the glance user and service project:

			# openstack role add --project service --user glance admin

	- Create the glance service entity:

			# openstack service create --name glance --description "OpenStack Image" image

4. Create the Image service API endpoints:

		# openstack endpoint create --region RegionOne image public http://controller:9292
		# openstack endpoint create --region RegionOne image internal http://controller:9292
		# openstack endpoint create --region RegionOne image admin http://controller:9292

## Install and configure components
	# yum install -y openstack-glance

edit ```/etc/glance/glance-api.conf```

	# vim /etc/glance/glance-api.conf
	[database]
	...
	connection = mysql+pymysql://glance:<GLANCE_DBPASS>@controller/glance

	[keystone_authtoken]
	...
	auth_uri = http://controller:5000
	auth_url = http://controller:35357
	memcached_servers = controller:11211
	auth_type = password
	project_domain_name = default
	user_domain_name = default
	project_name = service
	username = glance
	password = <GLANCE_PASS>

	[paste_deploy]
	...
	flavor = keystone

	[glance_store]
	...
	stores = file,http
	default_store = file
	filesystem_store_datadir = /var/lib/glance/images/

edit ```/etc/glance/glance-registry.conf```

	# vim /etc/glance/glance-registry.conf
	[database]
	...
	connection = mysql+pymysql://glance:<GLANCE_DBPASS>@controller/glance

	[keystone_authtoken]
	...
	auth_uri = http://controller:5000
	auth_url = http://controller:35357
	memcached_servers = controller:11211
	auth_type = password
	project_domain_name = default
	user_domain_name = default
	project_name = service
	username = glance
	password = <GLANCE_PASS>

	[paste_deploy]
	...
	flavor = keystone

Populate the Image service database:

	# su -s /bin/sh -c "glance-manage db_sync" glance

Start the Image services and configure them to start when the system boots

	# systemctl enable openstack-glance-api.service openstack-glance-registry.service
	# systemctl restart openstack-glance-api.service openstack-glance-registry.service

## Verify operation

1. download a src image:

		# wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img

2. Upload the image to the Image service using the QCOW2 disk format, bare container format, and public visibility so all projects can access it:

		# openstack image create "cirros" \
		  --file cirros-0.3.4-x86_64-disk.img \
		  --disk-format qcow2 --container-format bare \
		  --public

3. Confirm upload of the image and validate attributes:

		# openstack image list

---

# Compute service - nova
## Install and configure controller node
### Prerequisites
1. DB setup

		# mysql -u root -p
		CREATE DATABASE nova_api;
		CREATE DATABASE nova;
		GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY '<NOVA_DBPASS>';
		GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY '<NOVA_DBPASS>';
		GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY '<NOVA_DBPASS>';
		GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY '<NOVA_DBPASS>';
		exit

2. Source the admin credentials to gain access to admin-only CLI commands:

		# source admin-openrc

3. To create the service credentials, complete these steps:

	- Create the nova user:

			# openstack user create --domain default --password-prompt nova

	- Add the admin role to the nova user:

			# openstack role add --project service --user nova admin

	- Create the nova service entity:

			# openstack service create --name nova --description "OpenStack Compute" compute

4. Create the Compute service API endpoints:

		# openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1/%\(tenant_id\)s
		# openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1/%\(tenant_id\)s
		# openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1/%\(tenant_id\)s

### Install and configure components
	# yum install -y openstack-nova-api openstack-nova-conductor openstack-nova-console openstack-nova-novncproxy openstack-nova-scheduler

edit ```/etc/nova/nova.conf```:

	# cp /etc/nova/nova.conf /etc/nova/nova.conf.bak
	# vim /etc/nova/nova.conf
	[DEFAULT]
	...
	enabled_apis = osapi_compute,metadata
	rpc_backend = rabbit
	auth_strategy = keystone
	my_ip = 192.168.238.157
	use_neutron = True
	firewall_driver = nova.virt.firewall.NoopFirewallDriver

	[api_database]
	...
	connection = mysql+pymysql://nova:<NOVA_DBPASS>@controller/nova_api

	[database]
	...
	connection = mysql+pymysql://nova:<NOVA_DBPASS>@controller/nova

	[oslo_messaging_rabbit]
	...
	rabbit_host = controller
	rabbit_userid = openstack
	rabbit_password = <RABBIT_PASS>

	[keystone_authtoken]
	...
	auth_uri = http://controller:5000
	auth_url = http://controller:35357
	memcached_servers = controller:11211
	auth_type = password
	project_domain_name = default
	user_domain_name = default
	project_name = service
	username = nova
	password = <NOVA_PASS>

	[vnc]
	...
	vncserver_listen = $my_ip
	vncserver_proxyclient_address = $my_ip

	[glance]
	...
	api_servers = http://controller:9292

	[oslo_concurrency]
	...
	lock_path = /var/lib/nova/tmp

Populate the Compute databases:

	# su -s /bin/sh -c "nova-manage api_db sync" nova
	# su -s /bin/sh -c "nova-manage db sync" nova

Start the Compute services and configure them to start when the system boots:

	# systemctl enable openstack-nova-api.service \
	  openstack-nova-consoleauth.service openstack-nova-scheduler.service \
	  openstack-nova-conductor.service openstack-nova-novncproxy.service
	# systemctl restart openstack-nova-api.service \
	  openstack-nova-consoleauth.service openstack-nova-scheduler.service \
	  openstack-nova-conductor.service openstack-nova-novncproxy.service

## Install and configure a compute node

1. Install the packages:

		# yum install -y openstack-nova-compute

2. edit ```/etc/nova/nova.conf```

		# vim /etc/nova/nova.conf
		[DEFAULT]
		...
		rpc_backend = rabbit
		...
		auth_strategy = keystone
		...
		my_ip = <MANAGEMENT_INTERFACE_IP_ADDRESS>
		...
		use_neutron = True
		firewall_driver = nova.virt.firewall.NoopFirewallDriver

		[oslo_messaging_rabbit]
		...
		rabbit_host = controller
		rabbit_userid = openstack
		rabbit_password = <RABBIT_PASS>

		[keystone_authtoken]
		...
		auth_uri = http://controller:5000
		auth_url = http://controller:35357
		memcached_servers = controller:11211
		auth_type = password
		project_domain_name = default
		user_domain_name = default
		project_name = service
		username = nova
		password = <NOVA_PASS>

		[vnc]
		...
		enabled = True
		vncserver_listen = 0.0.0.0
		vncserver_proxyclient_address = $my_ip
		novncproxy_base_url = http://controller:6080/vnc_auto.html

		[glance]
		...
		api_servers = http://controller:9292

		[oslo_concurrency]
		...
		lock_path = /var/lib/nova/tmp

3. Determine whether your compute node supports hardware acceleration for virtual machines:

		# egrep -c '(vmx|svm)' /proc/cpuinfo

If this command returns a value of one or greater, your compute node supports hardware acceleration which typically requires no additional configuration.

If this command returns a value of zero, your compute node does not support hardware acceleration and you must configure libvirt to use QEMU instead of KVM.

- Edit the ```[libvirt]``` section in the ```/etc/nova/nova.conf``` file as follows:

			[libvirt]
			...
			virt_type = qemu

4. Start the Compute service including its dependencies and configure them to start automatically when the system boots:

		# systemctl enable libvirtd.service openstack-nova-compute.service
		# systemctl start libvirtd.service openstack-nova-compute.service

## Verify operation

1. Source the admin credentials to gain access to admin-only CLI commands:

	# source admin-openrc
	
2. List service components to verify successful launch and registration of each process:

	# openstack compute service list

---

# Networking service - neutron

## Prerequisites
1. DB setup

		# mysql -u root -p
		CREATE DATABASE neutron;
		GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY '<NEUTRON_DBPASS>';
		GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY '<NEUTRON_DBPASS>';
		exit
		
2. Source the admin credentials to gain access to admin-only CLI commands:

		# source admin-openrc
	
3. To create the service credentials, complete these steps:
- Create the neutron user:

		# openstack user create --domain default --password-prompt neutron
	
- Add the admin role to the neutron user:

		# openstack role add --project service --user neutron admin
	
- Create the neutron service entity:

		# openstack service create --name neutron --description "OpenStack Networking" network
	
4. Create the Networking service API endpoints:

		# openstack endpoint create --region RegionOne network public http://controller:9696
		# openstack endpoint create --region RegionOne network internal http://controller:9696
		# openstack endpoint create --region RegionOne network admin http://controller:9696
		
## Networking Option 1: Provider networks

### Install and Configure the components

	# yum install -y openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables
	
edit ```/etc/neutron/neutron.conf```

	# cp /etc/neutron/neutron.conf /etc/neutron/neutron.conf.bak
	# vim /etc/neutron/neutron.conf
	[DEFAULT]
	...
	core_plugin = ml2
	service_plugins =
	rpc_backend = rabbit
	auth_strategy = keystone
	notify_nova_on_port_status_changes = True
	notify_nova_on_port_data_changes = True
	
	[database]
	...
	connection = mysql+pymysql://neutron:<NEUTRON_DBPASS>@controller/neutron	

	[oslo_messaging_rabbit]
	...
	rabbit_host = controller
	rabbit_userid = openstack
	rabbit_password = <RABBIT_PASS>
		
	[keystone_authtoken]
	...
	auth_uri = http://controller:5000
	auth_url = http://controller:35357
	memcached_servers = controller:11211
	auth_type = password
	project_domain_name = default
	user_domain_name = default
	project_name = service
	username = neutron
	password = <NEUTRON_PASS>
	
	[nova]
	...
	auth_url = http://controller:35357
	auth_type = password
	project_domain_name = default
	user_domain_name = default
	region_name = RegionOne
	project_name = service
	username = nova
	password = <NOVA_PASS>
	
	[oslo_concurrency]
	...
	lock_path = /var/lib/neutron/tmp
	
### Configure the Modular Layer 2 (ML2) plug-in
edit ```/etc/neutron/plugins/ml2/ml2_conf.ini```

	# cp /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/ml2_conf.ini.bak
	# vim /etc/neutron/plugins/ml2/ml2_conf.ini
	[ml2]
	...
	type_drivers = flat,vlan
	tenant_network_types =
	mechanism_drivers = linuxbridge
	extension_drivers = port_security
	
	[ml2_type_flat]
	...
	flat_networks = provider
	
	[securitygroup]
	...
	enable_ipset = True
	
### Configure the Linux bridge agent
edit ```/etc/neutron/plugins/ml2/linuxbridge_agent.ini```

	# cp /etc/neutron/plugins/ml2/linuxbridge_agent.ini /etc/neutron/plugins/ml2/linuxbridge_agent.ini.bak
	# vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
	[linux_bridge]
	physical_interface_mappings = provider:<PROVIDER_INTERFACE_NAME(eth#)>
	
	[vxlan]
	enable_vxlan = False
	
	[securitygroup]
	...
	enable_security_group = True
	firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
	
### Configure the DHCP agent
edit ```/etc/neutron/dhcp_agent.ini```

	# cp /etc/neutron/dhcp_agent.ini /etc/neutron/dhcp_agent.ini.bak
	# vim /etc/neutron/dhcp_agent.ini
	[DEFAULT]
	...
	interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
	dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
	enable_isolated_metadata = True

## Configure the metadata agent
edit ```/etc/neutron/metadata_agent.ini```

	# cp /etc/neutron/metadata_agent.ini /etc/neutron/metadata_agent.ini.bak
	# vim /etc/neutron/metadata_agent.ini
	[DEFAULT]
	...
	nova_metadata_ip = controller
	metadata_proxy_shared_secret = METADATA_SECRET

Replace METADATA_SECRET with a suitable secret for the metadata proxy.

## Configure Compute to use Networking

	# vim /etc/nova/nova.conf
	[neutron]
	...
	url = http://controller:9696
	auth_url = http://controller:35357
	auth_type = password
	project_domain_name = default
	user_domain_name = default
	region_name = RegionOne
	project_name = service
	username = neutron
	password = NEUTRON_PASS

	service_metadata_proxy = True
	metadata_proxy_shared_secret = METADATA_SECRET

Replace METADATA_SECRET with the secret you chose for the metadata proxy.
	
## Finalize installation

	# ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
	
	# su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
    --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
  
	# systemctl restart openstack-nova-api.service
	# systemctl enable neutron-server.service \
	neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
	neutron-metadata-agent.service
	# systemctl restart neutron-server.service \
	neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
	neutron-metadata-agent.service

## Verity operation
Source the ```admin``` credentials to gain access to admin-only CLI commands:

	# source admin-openrc
	
List loaded extensions to verify successful launch of the neutron-server process:

	# neutron ext-list
	
The output should indicate three agents on the controller node and one agent on each compute node.

## Next steps

Your OpenStack environment now includes the core components necessary to launch a basic instance. You can [Launch an instance](http://docs.openstack.org/mitaka/install-guide-rdo/launch-instance.html#launch-instance) or add more OpenStack services to your environment.

---

# Dashboard - horizon
## Install and configure components
1. Install the packages:

		# yum install -y openstack-dashboard
	
2. edit ```/etc/openstack-dashboard/local_settings```

		# cp /etc/openstack-dashboard/local_settings /etc/openstack-dashboard/local_settings.bak
		# vim /etc/openstack-dashboard/local_settings
		OPENSTACK_HOST = "controller"
		ALLOWED_HOSTS = ['*', ]
		
		SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

		CACHES = {
			'default': {
				 'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
				 'LOCATION': 'controller:11211',
			}
		}
		
		OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
		
		OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
		
		OPENSTACK_API_VERSIONS = {
			"identity": 3,
			"image": 2,
			"volume": 2,
		}
		
		OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "default"
		
		OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
		
		OPENSTACK_NEUTRON_NETWORK = {
			...
			'enable_router': False,
			'enable_quotas': False,
			'enable_distributed_router': False,
			'enable_ha_router': False,
			'enable_lb': False,
			'enable_firewall': False,
			'enable_vpn': False,
			'enable_fip_topology_check': False,
		}
		
		TIME_ZONE = "Asia/Hong_Kong"

