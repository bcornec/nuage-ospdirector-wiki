# Introduction

This document outlines the architecture of the integration project of using ML2 with Nuage as mechanism driver with OSP Director 11.  

# OSP Director 11 integration with ML2 and Nuage as mechanism driver 
This document will focus on providing the information required to add and configure ML2 and Nuage.
 
The OSP Director is an image based installer. It uses a single image (named overcloud-full.qcow2) that is deployed on the Controller and Compute machines belonging to the overcloud OpenStack cluster. This image contains all the packages that are needed during the deployment. The deployment only creates the configuration files and databases required by the different services and starts the services in the correct order. Typically, there is no new software installation during the deployment phase. The packages/files required by ML2 will be added to this image as well.

The OSP Director architecture allows partners to create new templates to expose parameters specific to their modules and then the templates can be passed to the `openstack ovecloud deploy` command during the deployment. 
Additionally, changes to the puppet [manifests](http://git.openstack.org/cgit/openstack/tripleo-heat-templates/tree/puppet) are required to handle the new values in the Hiera database and act on them to deploy the partner software. ML2 options will be added to the existing Nuage templates.

# ML2 and SRIOV
This feature allows an OpenStack installation to support Single Root I/O Virtualization (SR-IOV)-attached VMs (https://wiki.openstack.org/wiki/SR-IOV-Passthrough-For-Networking) with VSP-managed VMs on the same KVM hypervisor cluster. It provides a Nuage ML2 mechanism driver that coexists with the sriovnicswitch mechanism driver.

Neutron ports attached through SR-IOV are configured by the sriovnicswitch mechanism driver. Neutron ports attached to Nuage VSD-managed networks are configured by the Nuage ML2 mechanism driver.

# Integration of Nuage VSP with OSP Director

The integration of Nuage VSP with OSP Director involves the following steps:

### OSP Director 11.0

Since OpenStack Ocata has capability for composable services, Nuage is added as mechanism driver with ML2 in a separate service to differentiate between Nuage as Neutron core plugin and Nuage as mechanism driver for ML2 as core plugin in tripleo-heat-templates at [this review](https://review.openstack.org/#/c/492224/3). This review contains Nuage mechanism driver as a composable service in tripleo-heat-templates. ID: https://review.openstack.org/#/c/492224/3. 

We have also removed the Compute specific Nuage parameters from extraconfig and made it part of an existing composable service. [This review](https://review.openstack.org/#/c/494239/) contains the required changes.The above mentioned changes are not in OSP Director 11.0 and need to be added MANUALLY since these changes are required on the Undercloud. Since this is WIP, the "heat_template_version:" in the file puppet/services/neutron-plugin-ml2-nuage.yaml and puppet/services/neutron-compute-plugin-nuage.yaml, needs to be changed to ocata from pike.

The following review provides support for non-default MTU configuration on the compute nodes. Allows to configure bridge MTU value. https://review.openstack.org/#/c/509284/

Horizon parameters - horizon::vhost_extra_params and horizon::customization_module are available but not configurable using horizon. [This change](https://review.openstack.org/#/c/511060/) exposes these parameters and makes them configurable.

Nova patching parameters are available in nova.conf but are not configurable from tripleo-heat-templates. [This change](https://review.openstack.org/#/c/511566/1) exposes these parameters from Nuage composable services to make them configurable. It enables setting the patching parameters in environment files. 

All the manual changes required are provided in the diff at [this link](https://github.com/nuagenetworks/nuage-ospdirector/tree/ML2-SRIOV/tripleo-heat-templates-diff). This contains the diff_OSPD11 file containing the differences that need to be applied. The steps for applying this patch are provided in the README [here](https://github.com/nuagenetworks/nuage-ospdirector/blob/ML2-SRIOV/tripleo-heat-templates-diff/README.md)

## Modification of overcloud-full image   
Since the typical deployment scenario of OSP Director assumes that all the packages are installed on the overcloud-full image, we need to patch the overcloud-full image with the following RPMs:  
* nuage-openstack-neutron  
* nuage-openstack-neutronclient    
* nuage-metadata-agent  
* nuage-openstack-heat
* nuage-openstack-horizon
* selinux-policy-nuage  
* nuage-puppet-modules-4.0 [link](https://github.com/nuagenetworks/nuage-ospdirector/blob/ML2-SRIOV/image-patching/nuage-puppet-modules-4.0.x86_64.rpm)   

Also, we need to un-install OVS and Install VRS
* Un-install OVS  
* Install VRS (nuage-openvswitch)  

The installation of packages and un-installation of OVS can be done via [this script](https://github.com/nuagenetworks/nuage-ospdirector/blob/ML2-SRIOV/image-patching/stopgap-script/nuage_overcloud_full_patch_w_ml2.sh).  
Since the files required to configure plugin.ini, neutron.conf and ml2_conf.ini are not in the OSP-Director codebase, the changes can be added to the image using the same [script](https://github.com/nuagenetworks/nuage-ospdirector/blob/ML2-SRIOV/image-patching/stopgap-script/nuage_overcloud_full_patch_w_ml2.sh). Copy the directory containing the 11_files at [this link](https://github.com/nuagenetworks/nuage-ospdirector/blob/ML2-SRIOV/image-patching/stopgap-script) and execute the script. For the next release this code will be upstreamed.

## Changes to openstack-tripleo-heat-templates
Some of the generic neutron.conf and nova.conf parameters need to be configured in the heat templates. Also, the metadata agent needs to be configured. The tripleo-heat-templates repository needs the extraconfig templates to configure the Nuage specific parameters. The values of these parameters are dependent on the configuration of Nuage VSP. The "Sample Templates" section contains some 'probable' values for these parameters in files neutron-nuage-config.yaml and nova-nuage-config.yaml.

## HA changes
For Nuage VSP with OpenStack HA, we need to disable the default services like openvswitch-agent and dhcp-agent from being controlled via Pacemaker. These services are also disabled in neutron-nuage-config.yaml file.

## Neutron Metadata configuration and VRS configuration  
A new puppet module is needed to create and populate the metadata agent config file and the VRS configuration in /etc/default/openvswitch. nuage-metadata-agent module will be included in Nuage-puppet-modules, along with other required Nuage packages. The section "Modification of overcloud-full image" mentions the steps for including Nuage-puppet-modules in the overcloud-full image used for Overcloud deployment.

# Deployment steps

## Modify overcloud-full.qcow2 to include Nuage components
The customer will receive all the RPMs and the script to patch the overcloud-full image with the RPMs. The user needs to create a local repo that is accessible from the machine that the script will run on and add all the RPMs to that repo. The machine also needs lib-guestfs-tools installed.
The script syntax is: `python nuage_overcloud_full_patch_w_ml2.py --RhelUserName=<value>  --RhelPassword='<value>' --RepoName=Nuage --RepoBaseUrl=http://IP/reponame --RhelPool=<value> --ImageName='<value>' --Version=11`  
This script takes in following input parameters:  
  RhelUserName: User name for the RHEL subscription    
  RhelPassword: Password for the RHEL subscription    
  RhelPool: RHEL Pool to subscribe to for base packages  
  RepoName: Name for the local repo hosting the Nuage RPMs  
  RepoBaseUrl: Base URL for the repo hosting the Nuage RPMs  
  ImageName: Name of the qcow2 image (overcloud-full.qcow2 for example)  
  Version: OSP-Director version (11)

## Deploy undercloud 
The undercloud deployment should proceed as per the OSP Director documentation. Follow all the steps before the `openstack overcloud deploy` command.  

#### Linux bonding with VLANs

Add network-environment.yaml file to /usr/share/openstack-tripleo-heat-templates/environments/
The sample is provided in the "Sample Templates" section

Nuage uses default linux bridge and linux bonds. For this to take effect, configuration changes need to be replicated to each of the node types that will be deployed, i.e. controller.yaml, compute.yaml etc. For these, the following network files are changed.
```
/usr/share/openstack-tripleo-heat-templates/network/config/bond-with-vlans/controller.yaml
```
and
```
/usr/share/openstack-tripleo-heat-templates/network/config/bond-with-vlans/compute.yaml
```

The changes that are required are:   
1. Remove ovs_bridge and move the containing members one level up   
2. Change ovs_bond to linux_bond with the right bonding_options (For example, bonding_options: 'mode=active-backup')   
3. Change the interface names under network_config and linux_bond to reflect the interface names of the baremetal machines that are being used. For the example below:   
  a. Name of provisioning interface on the baremetal machines in this case is "eno1"   
  b. The interfaces that will be bonded here are "eno2" and "eno3"   
```
Example
```
```
=========
Original
=========

    properties:
      group: os-apply-config
      config:
        os_net_config:
          network_config:
            -
              type: interface
              name: nic1
              use_dhcp: false
              dns_servers: {get_param: DnsServers}
              addresses:
                -
                  ip_netmask:
                    list_join:
                      - '/'
                      - - {get_param: ControlPlaneIp}
                        - {get_param: ControlPlaneSubnetCidr}
              routes:
                -
                  ip_netmask: 169.254.169.254/32
                  next_hop: {get_param: EC2MetadataIp}
                -
                  default: true
                  next_hop: {get_param: ControlPlaneDefaultRoute}
            -
              type: ovs_bridge
              name: {get_input: bridge_name}
              members:
                -
                  type: ovs_bond
                  name: bond1
                  ovs_options: {get_param: BondInterfaceOvsOptions}
                  members:
                    -
                      type: interface
                      name: nic2
                      primary: true
                    -
                      type: interface
                      name: nic3
                -
                  type: vlan
                  device: bond1
                  vlan_id: {get_param: InternalApiNetworkVlanID}
                  addresses:
                    -
                      ip_netmask: {get_param: InternalApiIpSubnet}
                -
                  type: vlan
                  device: bond1
                  vlan_id: {get_param: StorageNetworkVlanID}
                  addresses:
                    -
                      ip_netmask: {get_param: StorageIpSubnet}

```
```
==================================
Modified (changes are **marked**)
==================================

properties:
      group: os-apply-config
      config:
        os_net_config:
          network_config:
            -
              type: interface
              name: **eno1**
              use_dhcp: false
              dns_servers: {get_param: DnsServers}
              addresses:
                -
                  ip_netmask:
                    list_join:
                      - '/'
                      - - {get_param: ControlPlaneIp}
                        - {get_param: ControlPlaneSubnetCidr}
              routes:
                -
                  ip_netmask: 169.254.169.254/32
                  next_hop: {get_param: EC2MetadataIp}
                -
                  default: true
                  next_hop: {get_param: ControlPlaneDefaultRoute}
            -
              type: **linux_bond**
              name: bond1
              **bonding_options: 'mode=active-backup'**
              members:
                -
                  type: interface
                  name: **eno2**
                  primary: true
                -
                  type: interface
                  name: **eno3**
            -
              type: vlan
              device: bond1
              vlan_id: {get_param: InternalApiNetworkVlanID}
              addresses:
                -
                  ip_netmask: {get_param: InternalApiIpSubnet}
            -
              type: vlan
              device: bond1
              vlan_id: {get_param: StorageNetworkVlanID}
              addresses:
                -
                  ip_netmask: {get_param: StorageIpSubnet}
```

Since the deployment tries to reach the gateway of the OSP External Network VLAN, we need to configure a NIC on the undercloud with that IP and tag the packets to that NIC with the corresponding VLAN id.
```
Example:
In our example template the OSP External Network VLAN has a subnet 10.0.0.0/24 and so we add a NIC
sudo ifconfig enp0s7 10.0.0.1/24
```

### Generate CMS ID

For an Openstack installation, a CMS (Cloud Management System) ID needs to be generated to configure with Nuage VSD installation. The assumption is that Nuage VSD and Nuage VSC are already running before overcloud is deployed.

Steps to generate it:  
* Copy the [folder](https://github.com/nuagenetworks/nuage-ospdirector/tree/ML2-SRIOV/generate-cms-id) to a machine that can reach VSD (typically the undercloud node)  
* From the folder run the following command to generate CMS_ID:  
```
python configure_vsd_cms_id.py --server <vsd-ip-address>:<vsd-port> --serverauth <vsd-username>:<vsd-password> --organization <vsd-organization> --auth_resource /me --serverssl True --base_uri /nuage/api/<vsp-version>"  
example command : 
python configure_vsd_cms_id.py --server 0.0.0.0:0 --serverauth username:password --organization organization --auth_resource /me --serverssl True --base_uri "/nuage/api/v5_0"
```
* The CMS ID will be displayed on the terminal as well as a copy of it will be stored in a file "cms_id.txt" in the same folder.  
* This generated cms_id needs to be added to neutron-nuage-config.yaml template file for the parameter NeutronNuageCMSId  

## Overcloud Deployment commands
For OSP Director, tuskar deployment commands are recommended. But as part of Nuage integration effort, it was found that heat-templates provide more options and customization to overcloud deployment. The templates can be passed in "openstack overcloud deploy" command line options and can create or update an overcloud deployment.

### Non-HA
For non-HA overcloud deployment, following command was used for deploying with Nuage:

**openstack overcloud deploy --templates -e /usr/share/openstack-tripleo-heat-templates/environments/nova-nuage-config.yaml -e /usr/share/openstack-tripleo-heat-templates/environments/neutron-nuage-config.yaml --control-scale 1 --compute-scale 1 --ceph-storage-scale 0 --block-storage-scale 0 --swift-storage-scale 0**

For Virtual deployment, need to add --libvirt-type parameter as:

**openstack overcloud deploy --templates --libvirt-type qemu -e /usr/share/openstack-tripleo-heat-templates/environments/nova-nuage-config.yaml -e /usr/share/openstack-tripleo-heat-templates/environments/neutron-nuage-config.yaml --control-scale 1 --compute-scale 1 --ceph-storage-scale 0 --block-storage-scale 0 --swift-storage-scale 0**

where:  
neutron-nuage-config.yaml: Controller specific parameter values  
nova-nuage-config.yaml: Compute specific parameter values  

### HA
For HA deployment, following command was used for deploying with Nuage:

**openstack overcloud deploy --templates -e /usr/share/openstack-tripleo-heat-templates/environments/nova-nuage-config.yaml -e /usr/share/openstack-tripleo-heat-templates/environments/neutron-nuage-config.yaml --control-scale 2 --compute-scale 2 --ceph-storage-scale 0 --block-storage-scale 0 --swift-storage-scale 0 --ntp-server ntp.zam.alcatel-lucent.com**

For Virtual deployment, need to add --libvirt-type parameter as:

**openstack overcloud deploy --templates --libvirt-type qemu -e /usr/share/openstack-tripleo-heat-templates/environments/nova-nuage-config.yaml -e /usr/share/openstack-tripleo-heat-templates/environments/neutron-nuage-config.yaml --control-scale 2 --compute-scale 2 --ceph-storage-scale 0 --block-storage-scale 0 --swift-storage-scale 0 --ntp-server ntp.zam.alcatel-lucent.com**

where:  
neutron-nuage-config.yaml: Controller specific parameter values  
nova-nuage-config.yaml: Compute specific parameter values  


### Linux bonding Non-HA with Nuage
For linux bonding deployment with VLANs, following command was used for deploying with Nuage:

**openstack overcloud deploy --templates -e /usr/share/openstack-tripleo-heat-templates/environments/network-environment.yaml -e /usr/share/openstack-tripleo-heat-templates/environments/network-isolation.yaml -e /usr/share/openstack-tripleo-heat-templates/environments/net-bond-with-vlans.yaml -e /usr/share/openstack-tripleo-heat-templates/environments/nova-nuage-config.yaml -e /usr/share/openstack-tripleo-heat-templates/environments/neutron-nuage-config.yaml --control-scale 1 --compute-scale 1 --ceph-storage-scale 0 --block-storage-scale 0 --swift-storage-scale 0**

where:  
network-environment.yaml: Configures additional network environment variables  
network-isolation.yaml: Enables creation of networks for isolated overcloud traffic  
net-bond-with-vlans.yaml: Configures an IP address and a pair of bonded nics on each network  
neutron-nuage-config.yaml: Controller specific parameter values  
nova-nuage-config.yaml: Compute specific parameter values  

### Linux bonding HA with Nuage
For linux bonding deployment with VLANs for HA config, following command was used for deploying with Nuage:

**openstack overcloud deploy --templates -e /usr/share/openstack-tripleo-heat-templates/environments/network-environment.yaml -e /usr/share/openstack-tripleo-heat-templates/environments/network-isolation.yaml -e /usr/share/openstack-tripleo-heat-templates/environments/net-bond-with-vlans.yaml -e /usr/share/openstack-tripleo-heat-templates/environments/neutron-nuage-config.yaml -e /usr/share/openstack-tripleo-heat-templates/environments/nova-nuage-config.yaml --control-scale 2 --compute-scale 2 --ceph-storage-scale 0 --block-storage-scale 0 --swift-storage-scale 0 --ntp-server pool.ntp.org**

where:  
network-environment.yaml: Configures additional network environment variables  
network-isolation.yaml: Enables creation of networks for isolated overcloud traffic  
net-bond-with-vlans.yaml: Configures an IP address and a pair of bonded nics on each network  
neutron-nuage-config.yaml: Controller specific parameter values  
nova-nuage-config.yaml: Compute specific parameter values  

## Sample Templates
### network-environment.yaml
```
#This file is an example of an environment file for defining the isolated
#networks and related parameters.
resource_registry:
  # Network Interface templates to use (these files must exist)
  OS::TripleO::BlockStorage::Net::SoftwareConfig:
    ../network/config/bond-with-vlans/cinder-storage.yaml
  OS::TripleO::Compute::Net::SoftwareConfig:
    ../network/config/bond-with-vlans/compute.yaml
  OS::TripleO::Controller::Net::SoftwareConfig:
    ../network/config/bond-with-vlans/controller.yaml
  OS::TripleO::ObjectStorage::Net::SoftwareConfig:
    ../network/config/bond-with-vlans/swift-storage.yaml
  OS::TripleO::CephStorage::Net::SoftwareConfig:
    ../network/config/bond-with-vlans/ceph-storage.yaml

parameter_defaults:
  # This section is where deployment-specific configuration is done
  # CIDR subnet mask length for provisioning network
  ControlPlaneSubnetCidr: '24'
  # Gateway router for the provisioning network (or Undercloud IP)
  ControlPlaneDefaultRoute: 192.0.2.1
  EC2MetadataIp: 192.168.24.1  # Generally the IP of the Undercloud
  # Customize the IP subnets to match the local environment
  InternalApiNetCidr: 172.17.0.0/24
  StorageNetCidr: 172.18.0.0/24
  StorageMgmtNetCidr: 172.19.0.0/24
  TenantNetCidr: 172.16.0.0/24
  ExternalNetCidr: 10.0.0.0/24
  # Customize the VLAN IDs to match the local environment
  InternalApiNetworkVlanID: 20
  StorageNetworkVlanID: 30
  StorageMgmtNetworkVlanID: 40
  TenantNetworkVlanID: 50
  ExternalNetworkVlanID: 10
  # Customize the IP ranges on each network to use for static IPs and VIPs
  InternalApiAllocationPools: [{'start': '172.17.0.10', 'end': '172.17.0.200'}]
  StorageAllocationPools: [{'start': '172.18.0.10', 'end': '172.18.0.200'}]
  StorageMgmtAllocationPools: [{'start': '172.19.0.10', 'end': '172.19.0.200'}]
  TenantAllocationPools: [{'start': '172.16.0.10', 'end': '172.16.0.200'}]
  # Leave room if the external network is also used for floating IPs
  ExternalAllocationPools: [{'start': '10.0.0.10', 'end': '10.0.0.50'}]
  # Gateway router for the external network
  ExternalInterfaceDefaultRoute: 10.0.0.1
  # Uncomment if using the Management Network (see network-management.yaml)
  # ManagementNetCidr: 10.0.1.0/24
  # ManagementAllocationPools: [{'start': '10.0.1.10', 'end': '10.0.1.50'}]
  # Use either this parameter or ControlPlaneDefaultRoute in the NIC templates
  # ManagementInterfaceDefaultRoute: 10.0.1.1
  # Define the DNS servers (maximum 2) for the overcloud nodes
  DnsServers: ["8.8.8.8","8.8.4.4"]
  # List of Neutron network types for tenant networks (will be used in order)
  NeutronNetworkType: 'vxlan,vlan'
  # The tunnel type for the tenant network (vxlan or gre). Set to '' to disable tunneling.
  NeutronTunnelTypes: 'vxlan'
  # Neutron VLAN ranges per network, for example 'datacentre:1:499,tenant:500:1000':
  NeutronNetworkVLANRanges: 'datacentre:1:1000'
  # Customize bonding options, e.g. "mode=4 lacp_rate=1 updelay=1000 miimon=100"
  # for Linux bonds w/LACP, or "bond_mode=active-backup" for OVS active/backup.
  BondInterfaceOvsOptions: "bond_mode=active-backup"
```

### neutron-nuage-config.yaml
```
# A Heat environment file which can be used to enable a
# a Neutron Nuage backend on the controller, configured via puppet
resource_registry:
  OS::TripleO::Services::NeutronDhcpAgent: OS::Heat::None
  OS::TripleO::Services::NeutronL3Agent: OS::Heat::None
  OS::TripleO::Services::NeutronMetadataAgent: OS::Heat::None
  OS::TripleO::Services::NeutronOvsAgent: OS::Heat::None
  OS::TripleO::Services::ComputeNeutronOvsAgent: OS::Heat::None
  # Override the NeutronCorePlugin to use Nuage
  OS::TripleO::Services::NeutronCorePlugin: OS::TripleO::Services::NeutronCorePluginML2Nuage

parameter_defaults:
  NeutronNuageNetPartitionName: 'Nuage_Partition'
  NeutronNuageVSDIp: '192.0.2.190:8443'
  NeutronNuageVSDUsername: 'csproot'
  NeutronNuageVSDPassword: 'csproot'
  NeutronNuageVSDOrganization: 'csp'
  NeutronNuageBaseURIVersion: 'v4_0'
  NeutronNuageCMSId: 'e6a6d911-5f58-4641-b9c9-72d391c6e13a'
  UseForwardedFor: true
  NeutronServicePlugins: 'NuagePortAttributes,NuageAPI,NuageL3'
  NeutronDBSyncExtraParams: '--config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini --config-file /etc/neutron/plugins/nuage/plugin.ini'
  NeutronTypeDrivers: "vlan,vxlan,flat"
  NeutronNetworkType: 'vxlan,vlan,flat'
  NeutronMechanismDrivers: "nuage"
  NeutronPluginExtensions: "nuage_subnet,nuage_port,port_security"
  NeutronFlatNetworks: '*'
  NeutronTunnelIdRanges: "1:1000"
  NeutronNetworkVLANRanges: "physnet1:1:1000,physnet2:1:1000"
  NeutronVniRanges: "1001:2000"
  NovaOVSBridge: 'alubr0'
  NeutronMetadataProxySharedSecret: 'NuageNetworksSharedSecret'
  InstanceNameTemplate: 'inst-%08x'
  HorizonCustomizationModule: 'nuage_horizon.customization'
  HorizonVhostExtraParams:
    priority: 10
    access_log_format: '%a %l %u %t \"%r\" %>s %b \"%%{}{Referer}i\" \"%%{}{User-Agent}i\"'
    directories: [{'allow_override': ['None'], 'path': '/usr/lib/python2.7/site-packages/nuage_horizon', 'require': 'all granted', 'options': ['FollowSymLinks','MultiViews']}]
    add_listen: False
```

### nova-nuage-config.yaml for Virtual Setup
```
# Heat environment file which can be used to enable
# Nuage backend on the compute, configured via puppet
resource_registry:
  OS::TripleO::Services::ComputeNeutronCorePlugin: OS::TripleO::Services::ComputeNeutronCorePluginNuage

parameter_defaults:
  NuageActiveController: '192.0.2.191'
  NuageStandbyController: '0.0.0.0'
  NuageBridgeMTU: '9000'
  NovaOVSBridge: 'alubr0'
  NovaComputeLibvirtType: 'qemu'
  NovaIPv6: False
  NuageMetadataProxySharedSecret: 'NuageNetworksSharedSecret'
  NuageNovaApiEndpoint: 'internalURL'
```

### nova-nuage-config.yaml for Baremetal Setup
```
# Heat environment file which can be used to enable
# Nuage backend on the compute, configured via puppet
resource_registry:
  OS::TripleO::ComputeExtraConfigPre: ../puppet/extraconfig/pre_deploy/compute/nova-nuage.yaml
  OS::TripleO::Services::ComputeNeutronCorePlugin: ../puppet/services/neutron-compute-plugin-nuage.yaml

parameter_defaults:
  NuageActiveController: '192.0.2.191'
  NuageStandbyController: '0.0.0.0'
  NovaOVSBridge: 'alubr0'
  NuageBridgeMTU: '9000'
  NovaComputeLibvirtType: 'kvm'
  NovaIPv6: False
  NuageMetadataProxySharedSecret: 'NuageNetworksSharedSecret'
  NuageNovaApiEndpoint: 'internalURL'
```

## Parameter details
This section described the details of the parameters specified in the template files. Also, the configuration files where these parameters are set and used. See OpenStack Newton user guide install section for more details.

### Parameters on the Neutron Controller
The following parameters are mapped to values in /etc/neutron/plugins/nuage/plugin.ini file on the neutron controller

```
NeutronNuageNetPartitionName
Maps to default_net_partition_name parameter
```
```
NeutronNuageVSDIp
Maps to server parameter
```
```
NeutronNuageVSDUsername
NeutronNuageVSDPassword
Maps to serverauth as username:password
```
```
NeutronNuageVSDOrganization
Maps to organization parameter
```
```
NeutronNuageBaseURIVersion
Maps to the version in base_uri as /nuage/api/<version>
```
```
NeutronNuageCMSId
Maps to the cms_id parameter
```
The following parameters are mapped to values in /etc/neutron/neutron.conf file on the neutron controller
```
NeutronCorePlugin
Maps to core_plugin parameter in [DEFAULT] section
```
```
NeutronServicePlugins
Maps to service_plugins parameter in [DEFAULT] section
```
The following parameters are mapped to values in /etc/nova/nova.conf file on the neutron controller
```
UseForwardedFor
Maps to use_forwarded_for parameter in [DEFAULT] section
```
```
NeutronMetadataProxySharedSecret
Maps to metadata_proxy_shared_secret parameter in [neutron] section
```
```
InstanceNameTemplate
Maps to instance_name_template parameter in [DEFAULT] section
```
The following parameters are mapped to values in /etc/neutron/plugins/ml2/ml2_conf.ini file on the neutron controller
```
NeutronNetworkType
Maps to tenant_network_types in [ml2] section
```
```
NeutronPluginExtensions
Maps to extension_drivers in [ml2] section
```
```
NeutronTypeDrivers
Maps to type_drivers in [ml2] section
```
```
NeutronMechanismDrivers
Maps to mechanism_drivers in [ml2] section
```
```
NeutronFlatNetworks
Maps to flat_networks parameter in [ml2_type_flat] section
```
```
NeutronTunnelIdRanges
Maps to tunnel_id_ranges in [ml2_type_gre] section
```
```
NeutronNetworkVLANRanges
Maps to network_vlan_ranges in [ml2_type_vlan] section
```
```
NeutronVniRanges
Maps to vni_ranges in [ml2_type_vxlan] section
```
```
NeutronNuagePluginsML2FirewallDriver
Maps to firewall_driver in [securitygroup] section
```
The following parameters are used for setting/disabling services in undercloud's puppet code
```
OS::TripleO::Services::NeutronEnableDHCPAgent
OS::TripleO::Services::NeutronEnableL3Agent
OS::TripleO::Services::NeutronEnableMetadataAgent
OS::TripleO::Services::NeutronEnableOVSAgent
These parameters are used to disable the OpenStack default services as these are not used with Nuage integrated OpenStack cluster
```
The following parameter is used for setting values on the Controller using puppet code
```
NeutronNuageDBSyncExtraParams
String of extra command line parameters to append to the neutron-db-manage upgrade head command
```

### Parameters on the Nova Compute
The following parameters are mapped to values in /etc/default/openvswitch file on the nova compute

```
NuageActiveController
Maps to ACTIVE_CONTROLLER parameter
```
```
NuageStandbyController
Maps to STANDBY_CONTROLLER parameter
```
The following parameters are mapped to values in /etc/nova/nova.conf file on the nova compute
```
NovaOVSBridge
Maps to ovs_bridge parameter in [neutron] section
```
```
NovaComputeLibvirtType
Maps to virt_type parameter in [libvirt] section
```
```
NovaIPv6
Maps to use_ipv6 in [DEFAULT] section
```
The following parameters are mapped to values in /etc/default/nuage-metadata-agent file on the nova compute
```
NuageMetadataProxySharedSecret
Maps to METADATA_PROXY_SHARED_SECRET parameter. This need to match the setting in neutron controller above
```
```
NuageNovaApiEndpoint
Maps to NOVA_API_ENDPOINT_TYPE parameter. This needs to correspond to the setting for the Nova API endpoint as configured by OSP Director
```

# Appendix
## ML2 and SRIOV
This feature allows an OpenStack installation to support Single Root I/O Virtualization (SR-IOV)-attached VMs (https://wiki.openstack.org/wiki/SR-IOV-Passthrough-For-Networking) with VSP-managed VMs on the same KVM hypervisor cluster. It provides a Nuage ML2 mechanism driver that coexists with the sriovnicswitch mechanism driver.

Neutron ports attached through SR-IOV are configured by the sriovnicswitch mechanism driver. Neutron ports attached to Nuage VSD-managed networks are configured by the Nuage ML2 mechanism driver.

## Overcloud Deployment commands
For OSP Director, tuskar deployment commands are recommended. But as part of Nuage integration effort, it was found that heat-templates provide more options and customization to overcloud deployment. The templates can be passed in "openstack overcloud deploy" command line options and can create or update an overcloud deployment.

### Non-HA
For non-HA overcloud deployment, following command was used for deploying with Nuage:

**openstack overcloud deploy --templates -e /home/stack/templates/nova-nuage-config.yaml -e /home/stack/templates/neutron-nuage-config.yaml -e /home/stack/templates/neutron-sriov.yaml --control-scale 1 --compute-scale 1 --ceph-storage-scale 0 --block-storage-scale 0 --swift-storage-scale 0**

For Virtual deployment, need to add --libvirt-type parameter as:

**openstack overcloud deploy --templates --libvirt-type qemu -e /home/stack/templates/nova-nuage-config.yaml -e /home/stack/templates/neutron-nuage-config.yaml -e /home/stack/templates/neutron-sriov.yaml --control-scale 1 --compute-scale 1 --ceph-storage-scale 0 --block-storage-scale 0 --swift-storage-scale 0**

where:  
neutron-nuage-config.yaml: Controller specific parameter values  
nova-nuage-config.yaml: Compute specific parameter values  

### HA
For HA deployment, following command was used for deploying with Nuage:

**openstack overcloud deploy --templates -e /home/stack/templates/nova-nuage-config.yaml -e /home/stack/templates/neutron-nuage-config.yaml -e /home/stack/templates/neutron-sriov.yaml --control-scale 2 --compute-scale 2 --ceph-storage-scale 0 --block-storage-scale 0 --swift-storage-scale 0 --ntp-server ntp.zam.alcatel-lucent.com**

For Virtual deployment, need to add --libvirt-type parameter as:

**openstack overcloud deploy --templates --libvirt-type qemu -e /home/stack/templates/nova-nuage-config.yaml -e /home/stack/templates/neutron-nuage-config.yaml -e /home/stack/templates/neutron-sriov.yaml --control-scale 2 --compute-scale 2 --ceph-storage-scale 0 --block-storage-scale 0 --swift-storage-scale 0 --ntp-server ntp.zam.alcatel-lucent.com**

where:  
neutron-nuage-config.yaml: Controller specific parameter values  
nova-nuage-config.yaml: Compute specific parameter values  

### Sample Templates
### neutron-nuage-config.yaml
```
# A Heat environment file which can be used to enable a
# a Neutron Nuage backend on the controller, configured via puppet
resource_registry:
  OS::TripleO::Services::NeutronDhcpAgent: OS::Heat::None
  OS::TripleO::Services::NeutronL3Agent: OS::Heat::None
  OS::TripleO::Services::NeutronMetadataAgent: OS::Heat::None
  OS::TripleO::Services::NeutronOvsAgent: OS::Heat::None
  OS::TripleO::Services::ComputeNeutronOvsAgent: OS::Heat::None
  # Override the NeutronCorePlugin to use Nuage
  OS::TripleO::Services::NeutronCorePlugin: /usr/share/openstack-tripleo-heat-templates/puppet/services/neutron-plugin-ml2-nuage.yaml

parameter_defaults:
  NeutronNuageNetPartitionName: 'Nuage_Partition3'
  NeutronNuageVSDIp: '192.0.2.117:8443'
  NeutronNuageVSDUsername: 'csproot'
  NeutronNuageVSDPassword: 'csproot'
  NeutronNuageVSDOrganization: 'csp'
  NeutronNuageBaseURIVersion: 'v5_0'
  NeutronNuageCMSId: 'd1c4bcbd-ef9b-4d96-890c-44d6a4797a6d'
  UseForwardedFor: true
  NeutronServicePlugins: 'NuagePortAttributes,NuageAPI,NuageL3'
  NeutronDBSyncExtraParams: '--config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini --config-file /etc/neutron/plugins/nuage/plugin.ini'
  NeutronMechanismDrivers: 'nuage,sriovnicswitch,l2population,nuage_sriov'
  NeutronPluginExtensions: "nuage_subnet,nuage_port,port_security"
  NeutronFlatNetworks: '*'
  NovaOVSBridge: 'alubr0'
  NeutronMetadataProxySharedSecret: 'NuageNetworksSharedSecret'
  InstanceNameTemplate: 'inst-%08x'
  NeutronTunnelIdRanges: '1:1000'
  NeutronNetworkVLANRanges: 'physnet1:1:1000,physnet2:1:1000'
  NeutronVniRanges: '1001:2000'
  NovaPatchConfigMonkeyPatch: 'True'
  NovaPatchConfigMonkeyPatchModules: 'nova.network.neutronv2.api:nuage_nova_extensions.nova.network.neutronv2.api.decorator'
```

### nova-nuage-config.yaml for Virtual Setup
```
# Heat environment file which can be used to enable
# Nuage backend on the compute, configured via puppet
resource_registry:
  OS::TripleO::Services::ComputeNeutronCorePlugin: /usr/share/openstack-tripleo-heat-templates/puppet/services/neutron-compute-plugin-nuage.yaml

parameter_defaults:
  NuageActiveController: '192.0.2.191'
  NuageStandbyController: '0.0.0.0'
  NuageBridgeMTU: '9000'
  NovaPCIPassthrough: '[{"devname":"eno2","physical_network":"physnet1"},{"devname":"eno3","physical_network":"physnet2"}]'
  NovaIPv6: False
  NuageMetadataProxySharedSecret: 'NuageNetworksSharedSecret'
  NuageNovaApiEndpoint: 'internalURL'
  NovaOVSBridge: 'alubr0'
  NovaComputeLibvirtType: 'qemu'
  NovaComputeLibvirtVifDriver: 'nova.virt.libvirt.vif.LibvirtGenericVIFDriver'
```

### nova-nuage-config.yaml for Baremetal Setup
```
# Heat environment file which can be used to enable
# Nuage backend on the compute, configured via puppet
resource_registry:
  OS::TripleO::Services::ComputeNeutronCorePlugin: /usr/share/openstack-tripleo-heat-templates/puppet/services/neutron-compute-plugin-nuage.yaml

parameter_defaults:
  NuageActiveController: '192.0.2.191'
  NuageStandbyController: '0.0.0.0'
  NuageBridgeMTU: '9000'
  NovaPCIPassthrough: '[{"devname":"eno2","physical_network":"physnet1"},{"devname":"eno3","physical_network":"physnet2"}]'
  NovaIPv6: False
  NuageMetadataProxySharedSecret: 'NuageNetworksSharedSecret'
  NuageNovaApiEndpoint: 'internalURL'
  NovaOVSBridge: 'alubr0'
  NovaComputeLibvirtType: 'kvm'
  NovaComputeLibvirtVifDriver: 'nova.virt.libvirt.vif.LibvirtGenericVIFDriver'
```
### neutron-sriov.yaml
```
## A Heat environment that can be used to deploy SR-IOV
resource_registry:
  OS::TripleO::Services::NeutronSriovAgent: /usr/share/openstack-tripleo-heat-templates/puppet/services/neutron-sriov-agent.yaml

parameter_defaults:
  # Add PciPassthroughFilter to the scheduler default filters
  NovaSchedulerDefaultFilters: ['RetryFilter','AvailabilityZoneFilter','RamFilter','ComputeFilter','ComputeCapabilitiesFilter','ImagePropertiesFilter','ServerGroupAntiAffinityFilter','ServerGroupAffinityFilter','PciPassthroughFilter']
  NovaSchedulerAvailableFilters: ['nova.scheduler.filters.all_filters']
  
  # Provide the vendorid:productid of the VFs
  NeutronSupportedPCIVendorDevs: ['8086:154c','8086:10ca','8086:1520']
  NeutronPhysicalDevMappings: "datacentre:eno2,physnet2:eno3"

  # Number of VFs that needs to be configured for a physical interface
  NeutronSriovNumVFs: "eno2:5,eno3:7"
```


### Issues and Resolution
#### 1. In case one or more of the overcloud deployed nodes is stopped
Then for the node that was shutdown
```
nova start <node_name> as in overcloud-controller-0 
```

Once the node is up, execute the following on the node
```
pcs cluster start --all
pcs status
```

If the services do not come up, then try
```
pcs resource cleanup
```

#### 2. If the following issue is hit while running the patching script
```

virt-customize: error: libguestfs error: could not create appliance through 
libvirt.

Try running qemu directly without libvirt using this environment variable:
export LIBGUESTFS_BACKEND=direct
```

Run the following command before executing the script
```
export LIBGUESTFS_BACKEND=direct
```

#### 3. No valid host found error while registering nodes
```
openstack baremetal import --json instackenv.json
No valid host was found. Reason: No conductor service registered which supports driver pxe_ipmitool. (HTTP 404)
```

Workaround: Install python package python-dracclient and restart ironic-conductor service. Then try the command again
```
sudo yum install -y python-dracclient
exit (go to root user)
systemctl restart openstack-ironic-conductor
su - stack (switch to stack user)
source stackrc (source stackrc)
```

#### 4. ironic nde-list shows Instance UUID even after deleting the stack
```
[stack@instack ~]$ heat stack-list
WARNING (shell) "heat stack-list" is deprecated, please use "openstack stack list" instead
+----+------------+--------------+---------------+--------------+
| id | stack_name | stack_status | creation_time | updated_time |
+----+------------+--------------+---------------+--------------+
+----+------------+--------------+---------------+--------------+
[stack@instack ~]$ nova list
+----+------+--------+------------+-------------+----------+
| ID | Name | Status | Task State | Power State | Networks |
+----+------+--------+------------+-------------+----------+
+----+------+--------+------------+-------------+----------+
[stack@instack ~]$ ironic node-list
+--------------------------------------+------+--------------------------------------+-------------+--------------------+-------------+
| UUID                                 | Name | Instance UUID                        | Power State | Provisioning State | Maintenance |
+--------------------------------------+------+--------------------------------------+-------------+--------------------+-------------+
| 9e57d620-3ec5-4b5e-96b1-bf56cce43411 | None | 1b7a6e50-3c15-4228-85d4-1f666a200ad5 | power off   | available          | False       |
| 88b73085-1c8e-4b6d-bd0b-b876060e2e81 | None | 31196811-ee42-4df7-b8e2-6c83a716f5d9 | power off   | available          | False       |
| d3ac9b50-bfe4-435b-a6f8-05545cd4a629 | None | 2b962287-6e1f-4f75-8991-46b3fa01e942 | power off   | available          | False       |
+--------------------------------------+------+--------------------------------------+-------------+--------------------+-------------+
```

Workaround: Manually remove instance_uuid reference
```
ironic node-update <node_uuid> remove instance_uuid
E.g. ironic node-update 9e57d620-3ec5-4b5e-96b1-bf56cce43411 remove instance_uuid
```