# Introduction

This document outlines the architecture of the integration project to add Nuage VSP to OSP Director. 

# OSP Director Architecture
 
OSP Director is a tool set for installing and managing an OpenStack environment. OSP Director is primarily based on the TripleO project and uses an OpenStack deployment (known as undercloud) to deploy the OpenStack cluster (known as the overcloud). The OSP Director is an image based installer. It uses a single image (named overcloud-full.qcow2) that is deployed on the Controller and Compute machines belonging to the overcloud OpenStack cluster. This image contains all the packages that are needed during the deployment. The deployment only creates the configuration files and databases required by the different services and starts the services in the correct order.  Typically, there is no new software installation during the deployment phase.

An OpenStack cluster can be installed using OSP Director in 2 ways:
* Using a UI (tuskar) for basic deployments  
* Using command line for advanced deployments (template option)  
For integration of OSP Director with Nuage, the command line based template option will be used.


OSP Director uses Heat to orchestrate the deployment of the OpenStack environment. The actual deployment is done via Heat templates and Puppet. The relevant Heat templates are found in the following [module](http://git.openstack.org/cgit/openstack/tripleo-heat-templates/). The users provide any custom input via templates via the `openstack overcloud deploy` command. When this command is run, all the templates are parsed to create the Hiera database and then a set of puppet [manifests](http://git.openstack.org/cgit/openstack/tripleo-heat-templates/tree/puppet) are run to complete the deployment. The puppet code in turn uses the puppet modules developed to deploy different services of OpenStack (such as puppet-nova, puppet-neutron, puppet-cinder, etc).

The OSP Director architecture allows partners to create custom templates (known as extraconfig templates) found [here](http://git.openstack.org/cgit/openstack/tripleo-heat-templates/tree/puppet/extraconfig). Partners create new templates to expose parameters specific to their modules and then the templates can be passed to the `openstack ovecloud deploy` command during the deployment. 
Additionally, changes to the puppet [manifests](http://git.openstack.org/cgit/openstack/tripleo-heat-templates/tree/puppet) are required to handle the new values in the Hiera database and act on them to deploy the partner software.

# Integration of Nuage VSP with OSP Director

The integration of Nuage VSP with OSP Director involves the following steps:

## OSP Director 10.0
From OSP Director 10.0 firewall rules are added by default. So, for releases Newton and later, tripleo-heat-templates were modified to add firewall rules for VxLAN and Metadata agent and the changes are at [this review](https://review.openstack.org/#/c/452932/). This review contains the changes required to puppet files that enable Nuage specific code. ID: https://review.openstack.org/#/c/452932/. This change is not in OSP-Director 10.0 yet.

## Modification of overcloud-full image   
Since the typical deployment scenario of OSP Director assumes that all the packages are installed on the overcloud-full image, we need to patch the overcloud-full image with the following RPMs:  
* nuagenetlib  
* nuage-openstack-neutron  
* nuage-openstack-neutronclient  
* nuage-metadata-agent  
* nuage-puppet-modules-3.0  

Also, we need to un-install OVS and Install VRS
* Un-install OVS  
* Install VRS (nuage-openvswitch)  

The installation of packages and un-installation of OVS can be done via [this script](https://github.com/nuagenetworks/nuage-ospdirector/blob/master/image-patching/stopgap-script/nuage_overcloud_full_patch.sh).  
Since the files required to configure plugin.ini are not in the OSP-Director codebase, the changes can be added to the image using the same [script](https://github.com/nuagenetworks/nuage-ospdirector/blob/master/image-patching/stopgap-script/nuage_overcloud_full_patch.sh). Copy the directory containing the files and the script at [this link](https://github.com/nuagenetworks/nuage-ospdirector/tree/master/image-patching/stopgap-script) and execute the script.

## Changes to openstack-tripleo-heat-templates
Some of the generic neutron.conf and nova.conf parameters need to be configured in the heat templates. Also, the metadata agent needs to be configured. The tripleo-heat-templates repository needs the extraconfig templates to configure the Nuage specific parameters. The values of these parameters are dependent on the configuration of Nuage VSP. The "Sample Templates" section contains some 'probable' values for these parameters in files neutron-nuage-config.yaml and nova-nuage-config.yaml.

## HA changes
For Nuage VSP with OpenStack HA, we need to disable the default services like openvswitch-agent and dhcp-agent from being controlled via Pacemaker. These services are also disabled in neutron-nuage-config.yaml file.

## Neutron Metadata configuration and VRS configuration  
A new puppet module is needed to create and populate the metadata agent config file and the VRS configuration in /etc/default/openvswitch. nuage-metadata-agent module will be included in Nuage-puppet-modules, along with other required Nuage packages. The section "Modification of overcloud-full image" mentions the steps for including Nuage-puppet-modules in the overcloud-full image used for Overcloud deployment.

# Deployment steps

## Modify overcloud-full.qcow2 to include Nuage components
The RPMs are available through normal Nuage download process. The script to patch the overcloud-full image with the RPMs is linked above in the "Modification of overcloud-full image" section along with the details. The user needs to create a local repo that is accessible from the machine that the script will run on and add all the RPMs to that repo. The machine also needs lib-guestfs-tools installed.
The script syntax is: `source nuage_overcloud_full_patch.sh --RhelUserName=<value>  --RhelPassword='<value>' --RepoName=Nuage --RepoBaseUrl=http://IP/reponame --RhelPool=<value> --ImageName='<value>' --Version=10`  
This script takes in following input parameters:  
  RhelUserName: User name for the RHEL subscription    
  RhelPassword: Password for the RHEL subscription    
  RhelPool: RHEL Pool to subscribe to for base packages  
  RepoName: Name for the local repo hosting the Nuage RPMs  
  RepoBaseUrl: Base URL for the repo hosting the Nuage RPMs  
  ImageName: Name of the qcow2 image (overcloud-full.qcow2 for example)  
  Version: OSP-Director version (10)

## Deploy undercloud 
The undercloud deployment should proceed as per the OSP Director documentation. Follow all the steps before the `openstack overcloud deploy` command.  

### Optional Features   
You can chose to enable the following features   
#### Linux bonding with VLANs

Add network-environment.yaml file to /usr/share/openstack-tripleo-heat-templates/environments/
The sample is provided in the "Sample Templates" section

Nuage uses default linux bridge and linux bonds. For this to take effect, following network files are changed.
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
3. Change the interface names under network_config and linux_bond to reflect the interface names of the baremetal machines that are being used.   
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

Since Nuage uses alubr0 bridge for connectivity and does not rely on the default br-ex bridge, we need to add a route for external network VLAN on the undercloud using br-ctlplane IP as the gateway
```
Example:
sudo route add -net 10.0.0.0/16 gw 192.0.2.1
```

#### Load Balancing As a Service (LBaaS)   
Load Balancing-as-a-Service (LBaaS) enables OpenStack Networking to distribute incoming requests evenly between designated instances.   
Since OSP Director 8.0 does not have LBaaS support, following manual configuration steps are required to use OpenStack LBaaS with Nuage Neutron Plugin   

1. Changes to overcloud-full.qcow2 image   
   a. Add "python-neutron-lbaas", "neutron-lbaas-agent" and "iputils-arping" packages   
   b. Add service_providers parameter to /etc/puppet/modules/neutron/manifests/server.pp   

2. Changes to /usr/share/openstack-tripleo-heat-templates/puppet/manifests/overcloud_controller_pacemaker.pp   
   a. In core_plugin as NuagePlugin section, add VRS and LBaaS   
   ```
   if  hiera('neutron::core_plugin') == 'nuage_neutron.plugins.nuage.plugin.NuagePlugin' {
     include ::neutron::plugins::nuage

     class {'::nuage::vrs':
       active_controller => '<VSC_IP>'
     }

     class {'::neutron::agents::lbaas':
       interface_driver       => 'nuage_neutron.lbaas.agent.nuage_interface.NuageInterfaceDriver',
       manage_haproxy_package => false
     }
   }

   ```
   where 
   `<VSC_IP> is VSC Active Controller IP`   

3. Make neutron::server::service_providers parameter configurable   
   a. Addition to /usr/share/openstack-tripleo-heat-templates/puppet/extraconfig/pre_deploy/controller/neutron-nuage.yaml   
   Addition to parameters section   
   ```
   NeutronNuageLBaaSServicesProviders:
       description: Defines providers for LBaaS service using the format - "<service_type>:<name>:<driver>[:default]"
       type: comma_delimited_list
       default: ''
   ```
   Addition to NeutronNuageConfig section   
   ```
   neutron::server::service_providers: {get_input: NuageLBaaSServicesProviders}
   ```
   Addition to NeutronNuageDeployment section   
   ```
   NuageLBaaSServicesProviders: {get_param: NeutronNuageLBaaSServicesProviders}
   ```
4. Addition to environments yaml files   
   a. Addition to /usr/share/openstack-tripleo-heat-templates/environments/neutron-nuage-config.yaml   
   ```
   NeutronNuageLBaaSServicesProviders: 'LOADBALANCERV2:Haproxy:neutron_lbaas.drivers.haproxy.plugin_driver.HaproxyOnHostPluginDriver:default'
   ```

   b. Addition to /usr/share/openstack-tripleo-heat-templates/environments/neutron-generic.yaml   
   ```
   NeutronServicePlugins: 'neutron_lbaas.services.loadbalancer.plugin.LoadBalancerPluginv2'
   ```

5. Changes to Controller/Network node post-deployment
   a. In the default section of /etc/neutron/neutron.conf file add   
   ```
   [DEFAULT]
   ovs_integration_bridge = alubr0
   ```
   b. Under the default section of /etc/neutron/lbaas_agent.ini file add   
   ```
   [DEFAULT]
   ovs_use_veth=False
   interface_driver=nuage_neutron.lbaas.agent.nuage_interface.NuageInterfaceDriver
   ```
   c. Start the neutron-lbaas-agent service
   ```
   systemctl start neutron-lbaasv2-agent
   ```

This should configure LBaaS V2 on the overcloud

### Generate CMS ID

For an Openstack installation, a CMS (Cloud Management System) ID needs to be generated to configure with Nuage VSD installation.

Steps to generate it:  
* Copy the [folder](https://github.com/nuagenetworks/nuage-ospdirector/tree/master/generate-cms-id) to a machine that can reach VSD (typically the undercloud node)  
* From the folder run the following command to generate CMS_ID:  
```
python configure_vsd_cms_id.py --server <vsd-ip-address>:<vsd-port> --serverauth <vsd-username>:<vsd-password> --organization <vsd-organization> --auth_resource /me --serverssl True --base_uri /nuage/api/<vsp-version>"  
example command : 
python configure_vsd_cms_id.py --server 0.0.0.0:0 --serverauth username:password --organization organization --auth_resource /me --serverssl True --base_uri "/nuage/api/v4_0"
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
# This template configures additional network environment variables specific
# to the deployment environment.
resource_registry:
  OS::TripleO::BlockStorage::Net::SoftwareConfig: ../network/config/bond-with-vlans/cinder-storage.yaml
  OS::TripleO::Compute::Net::SoftwareConfig: ../network/config/bond-with-vlans/compute.yaml
  OS::TripleO::Controller::Net::SoftwareConfig: ../network/config/bond-with-vlans/controller.yaml
  OS::TripleO::ObjectStorage::Net::SoftwareConfig: ../network/config/bond-with-vlans/swift-storage.yaml
  OS::TripleO::CephStorage::Net::SoftwareConfig: ../network/config/bond-with-vlans/ceph-storage.yaml

# We use parameter_defaults instead of parameters here because Tuskar munges
# the names of top level and role level parameters with the role name and a
# version. Using parameter_defaults makes it such that if the parameter name is
# not defined in the template, we don't get an error.
parameter_defaults:
  # Gateway router for the provisioning network (or Undercloud IP)
  ControlPlaneDefaultRoute: 192.0.2.254
  # Generally the IP of the Undercloud
  EC2MetadataIp: 192.0.2.1
  # Define the DNS servers (maximum 2) for the overcloud nodes
  DnsServers: ['8.8.8.8','8.8.4.4']
  # Customize bonding options if required (ignored if bonds are not used)
  # BondInterfaceOvsOptions: 'mode=active-backup'
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
  OS::TripleO::Services::NeutronCorePlugin: OS::TripleO::Services::NeutronCorePluginNuage

parameter_defaults:
  NeutronNuageNetPartitionName: 'Nuage_Partition3'
  NeutronNuageVSDIp: '192.0.2.190:8443'
  NeutronNuageVSDUsername: 'csproot'
  NeutronNuageVSDPassword: 'csproot'
  NeutronNuageVSDOrganization: 'csp'
  NeutronNuageBaseURIVersion: 'v4_0'
  NeutronNuageCMSId: 'a26b8bc6-59bf-41df-8d54-4b8b21ea31c5'
  UseForwardedFor: true
  NeutronCorePlugin: 'nuage_neutron.plugins.nuage.plugin.NuagePlugin'
  NeutronServicePlugins: 'nuage_neutron.plugins.common.service_plugins.port_attributes.service_plugin.NuagePortAttributesServicePlugin'
  NovaOVSBridge: 'alubr0'
  NeutronMetadataProxySharedSecret: 'NuageNetworksSharedSecret'
  InstanceNameTemplate: 'inst-%08x'
  controllerExtraConfig:
    neutron::api_extensions_path: '/usr/lib/python2.7/site-packages/neutron/plugins/nuage/'
```

### nova-nuage-config.yaml for Virtual Setup
```
# A Heat environment file which can be used to enable
# Nuage backend on the compute, configured via puppet
resource_registry:
  OS::TripleO::ComputeExtraConfigPre: ../puppet/extraconfig/pre_deploy/compute/nova-nuage.yaml
  OS::TripleO::Services::ComputeNeutronCorePlugin: ../puppet/services/neutron-compute-plugin-nuage.yaml

parameter_defaults:
  NuageActiveController: '192.0.2.201'
  NuageStandbyController: '0.0.0.0'
  NovaOVSBridge: 'alubr0'
  NovaComputeLibvirtType: 'qemu'
  NuageMetadataProxySharedSecret: 'NuageNetworksSharedSecret'
  NuageNovaApiEndpoint: 'internalURL'
```

### nova-nuage-config.yaml for Baremetal Setup
```
# A Heat environment file which can be used to enable
# Nuage backend on the compute, configured via puppet
resource_registry:
  OS::TripleO::ComputeExtraConfigPre: ../puppet/extraconfig/pre_deploy/compute/nova-nuage.yaml
  OS::TripleO::Services::ComputeNeutronCorePlugin: ../puppet/services/neutron-compute-plugin-nuage.yaml

parameter_defaults:
  NuageActiveController: '192.0.2.201'
  NuageStandbyController: '0.0.0.0'
  NovaOVSBridge: 'alubr0'
  NovaComputeLibvirtType: 'kvm'
  NuageMetadataProxySharedSecret: 'NuageNetworksSharedSecret'
  NuageNovaApiEndpoint: 'internalURL'
```

## Parameter details
This section described the details of the parameters specified in the template files. Also, the configuration files where these parameters are set and used. See OpenStack Liberty user guide install section for more details.

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
The following parameters are used for setting/disabling services in undercloud's puppet code
```
OS::TripleO::Services::NeutronEnableDHCPAgent
OS::TripleO::Services::NeutronEnableL3Agent
OS::TripleO::Services::NeutronEnableMetadataAgent
OS::TripleO::Services::NeutronEnableOVSAgent
These parameters are used to disable the OpenStack default services as these are not used with Nuage integrated OpenStack cluster
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
The following parameters are mapped to values in /etc/neutron/neutron.conf file on the nova compute
```
NeutronCorePlugin
Maps to core_plugin parameter in [DEFAULT] section
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