# Introduction

This document outlines the architecture of the integration project to add Nuage VSP to OSP Director. 

# OSP Director Architecture
 
OSP Director is a tool set for installing and managing an OpenStack environment. OSP Director is primarily based on the TripleO project and uses an OpenStack deployment (known as undercloud) to deploy the OpenStack cluster (known as the overcloud). The OSP Director is an image based installer. It uses a single image (named overcloud-full.qcow2) that is deployed on the Controller and Compute machines belonging to the overcloud OpenStack cluster. This image contains all the packages that are needed during the deployment. The deployment only creates the configuration files and databases required by the different services and starts the services in the correct order.  Typically, there is no new software installation during the deployment phase.

An OpenStack cluster can be installed using OSP Director in 2 ways:
* Using a UI (tuskar) for basic deployments
* Using command line for advanced deployments (template option)  
For integration of OSP Director with Nuage, the command line based template option will be used.


OSP Director uses Heat to orchestrate the deployment of the OpenStack environment. The actual deployment is done via Heat templates and Puppet. The relevant Heat templates are found in the [tripleo](https://github.com/nuagenetworks/ospd-experimental/tree/master/tripleo/) folder in the repository. The users provide any custom input via templates via the `openstack overcloud deploy` command. When this command is run, all the templates are parsed to create the Hiera database and then a set of puppet manifests are run to complete the deployment. The puppet code in turn uses the puppet modules developed to deploy different services of OpenStack (such as puppet-nova, puppet-neutron, puppet-cinder, etc).

The OSP Director architecture allows partners to create custom templates (known as extraconfig templates) found [here](https://github.com/nuagenetworks/ospd-experimental/tree/master/tripleo/puppet/extraconfig). Partners create new templates to expose parameters specific to their modules and then the templates can be passed to the `openstack ovecloud deploy` command during the deployment. 
Additionally, changes to the puppet manifests are required to handle the new values in the Hiera database and act on them to deploy the partner software.

# Integration of Nuage VSP with OSP Director

The integration of Nuage VSP with OSP Director involves the following changes:

## Modification of overcloud-full image
Since the typical deployment scenario of OSP Director assumes that all the packages are installed on the overcloud-full image, we need to patch the overcloud-full image with the following RPMs:  
* Nuagenetlib  
* Nuage-openstack-neutron
* Nuage-openstack-neutron-client
* Nuage-metadata-agent  
* Uninstall OVS  
* Install VRS   
* Nuage-puppet-module

The patching can be done via [this script](https://github.com/nuagenetworks/ospd-experimental/blob/master/image-patching/nuage_overcloud_full_patch.sh). Nuage netlib, openstack-neutron, openstack-neutron-client and metadata-agent RPMs are provided separately. nuage-puppet-modules RPM can be found [here](https://github.com/nuagenetworks/ospd-experimental/blob/master/image-patching/nuage-puppet-modules-3.2-R4.x86_64.rpm). This RPM is compatible with both Nuage R3 and R4 RPMs.

In addition to this puppet-neutron and puppet-nova changes need to be manually patched to the overcloud-full.qcow2 image using guestfish.

### puppet-neutron changes
Puppet-Neutron directory is provided [here](https://github.com/nuagenetworks/ospd-experimental/tree/master/puppet-neutron). The files that need to be added to overcloud-full.qcow2 image are:

* lib/puppet/provider/neutron_plugin_nuage/ini_setting.rb
* lib/puppet/type/neutron_plugin_nuage.rb
* manifests/plugins/nuage.pp

The files that need to be modified are:

* manifests/config.pp
* manifests/params.pp

### puppet-nova changes
Puppet-Nova changes are present [here](https://github.com/nuagenetworks/ospd-experimental/tree/master/puppet-nova). Following file needs to be modified to include instance_name_template parameter.

* manifests/api.pp

## Patch the openstack-tripleo-heat-templates package
As part of undercloud installation, the openstack-tripleo-heat-templates rpm gets installed at `/usr/share/openstack-tripleo-heat-templates`. This RPM needs to be patched with the Nuage changes. All the changes have been patched and are available in this [repository](https://github.com/nuagenetworks/ospd-experimental/tree/master/tripleo). The repository can be used directly on the undercloud machine.

# Customer Deployment steps

## Modify overcloud-full.qcow2 to include Nuage components
The customer will receive all the RPMs and the script to patch the overcloud-full image with the RPMs. The user needs to create a local repo that is accessible from the machine that the script will run on and add all the RPMs to that repo. The machine also needs lib-guestfs-tools installed.
The script syntax is: `source nuage_overcloud_customize.sh --RhelUserName=<value>  --RhelPassword='<value>' --RepoName=<value> --RepoBaseUrl='<value>' --RhelPool=<value> --ImageName='<value>'`
This script takes in following input parameters:
  RhelUserName: User name for the RHEL subscription
  RhelPassword: Password for the RHEL subscription
  RhelPool    : RHEL Pool to subscribe
  RepoName    : Name of the local repository
  RepoBaseUrl : Base URL of the local repository


## Deploy undercloud
The undercloud deployment should proceed as per the OSP Director documentation. Follow all the steps until the `openstack overcloud deploy` command.

## Change the overcloud-resource-registry-puppet.yaml file

Open the file on the undercloud machine. It can be found at /usr/share/openstack-tripleo-heat-templates/

Change the following lines as shown below:
```
OS::TripleO::Controller::Net::SoftwareConfig: net-config-noop.yaml
OS::TripleO::ControllerExtraConfigPre: puppet/extraconfig/pre_deploy/controller/neutron-nuage.yaml
OS::TripleO::ComputeExtraConfigPre: puppet/extraconfig/pre_deploy/compute/nova-nuage.yaml
```
## Changes for linux bonding with VLANs

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

The changes include to remove ovs_bridge and change ovs_bond to linux_bond with the right bonding_options. Also, the interface names need to change to reflect the interface names of the baremetal machines that are being used.

All these changes can be found at:

for controller [here](https://github.com/nuagenetworks/ospd-experimental/blob/master/tripleo/network/config/bond-with-vlans/controller.yaml)

for compute [here](https://github.com/nuagenetworks/ospd-experimental/blob/master/tripleo/network/config/bond-with-vlans/compute.yaml)

where the interfaces names are:
eno1: PxE boot interface
eno2 and eno3: Bonded interfaces

## Overcloud Deployment commands
For OSP Director, tuskar deployment commands are recommended. But as part of Nuage integration effort, it was found that heat-templates provide more options and customization to overcloud deployment. The templates can be passed in "openstack overcloud deploy" command line options and can create or update an overcloud deployment.

### Non-HA
For non-HA overcloud deployment, following command was used for deploying with Nuage:

**openstack overcloud deploy --templates -e neutron-nuage-config.yaml -e neutron-generic.yaml -e nova-nuage-config.yaml -e nova-generic.yaml --control-scale 1 --compute-scale 1 --ceph-storage-scale 0 --block-storage-scale 0 --swift-storage-scale 0**

For Virtual deployment, need to add --libvirt-type parameter as:

**openstack overcloud deploy --templates --libvirt-type qemu -e neutron-nuage-config.yaml -e neutron-generic.yaml -e nova-nuage-config.yaml -e nova-generic.yaml --control-scale 1 --compute-scale 1 --ceph-storage-scale 0 --block-storage-scale 0 --swift-storage-scale 0**

where:
neutron-nuage-config.yaml: Nuage specific controller parameter values
neutron-generic.yaml: Values for neutron config parameters as CorePlugin and ServicePlugins
nova-nuage-config.yaml: Nuage specific compute parameter values
nova-generic.yaml: Values for nova config parameters as OVSBridge, SecurityGroupApi, etc.

### HA
For HA deployment, following command was used for deploying with Nuage:

**openstack overcloud deploy --templates -e neutron-nuage-config.yaml -e neutron-generic.yaml -e nova-nuage-config.yaml -e nova-generic.yaml --control-scale 2 --compute-scale 2 --ceph-storage-scale 0 --block-storage-scale 0 --swift-storage-scale 0 --ntp-server pool.ntp.org**

For Virtual deployment, need to add --libvirt-type parameter as:

**openstack overcloud deploy --templates --libvirt-type qemu -e neutron-nuage-config.yaml -e neutron-generic.yaml -e nova-nuage-config.yaml -e nova-generic.yaml --control-scale 2 --compute-scale 2 --ceph-storage-scale 0 --block-storage-scale 0 --swift-storage-scale 0 --ntp-server pool.ntp.org**

where:
neutron-nuage-config.yaml: Nuage specific controller parameter values as well as services disabling parameters
neutron-generic.yaml: Values for neutron config parameters as CorePlugin and ServicePlugins
nova-nuage-config.yaml: Nuage specific compute parameter values
nova-generic.yaml: Values for nova config parameters as LibvirtVifDriver, OVSBridge, SecurityGroupApi, etc.


### Linux bonding Non-HA with Nuage
For linux bonding deployment with VLANs, following command was used for deploying with Nuage:

**openstack overcloud deploy --templates -e network-environment.yaml -e network-isolation.yaml -e net-bond-with-vlans.yaml -e neutron-nuage-config.yaml -e neutron-generic.yaml -e nova-nuage-config.yaml -e nova-generic.yaml --control-scale 1 --compute-scale 1 --ceph-storage-scale 0 --block-storage-scale 0 --swift-storage-scale 0**


### Linux bonding HA with Nuage
For linux bonding deployment with VLANs for HA config, following command was used for deploying with Nuage:

**openstack overcloud deploy --templates -e network-environment.yaml -e network-isolation.yaml -e net-bond-with-vlans.yaml -e neutron-nuage-config.yaml -e neutron-generic.yaml -e nova-nuage-config.yaml -e nova-generic.yaml--control-scale 2 --compute-scale 2 --ceph-storage-scale 0 --block-storage-scale 0 --swift-storage-scale 0 --ntp-server pool.ntp.org**

## Sample Templates

### network-environment.yaml
included [here](https://github.com/nuagenetworks/ospd-experimental/blob/master/tripleo/environments/network-environment.yaml)
```
# This template configures each role to use a pair of bonded nics (nic2 and
# nic3) and configures an IP address on each relevant isolated network
# for each role. This template assumes use of network-isolation.yaml.
#
# FIXME: if/when we add functionality to heatclient to include heat
# environment files we should think about using it here to automatically
# include network-isolation.yaml.
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
included [here](https://github.com/nuagenetworks/ospd-experimental/blob/master/tripleo/environments/neutron-nuage-config.yaml)
```
# A Heat environment file which can be used to enable a
# a Neutron Nuage backend, configured via puppet
resource_registry:
  OS::TripleO::ControllerExtraConfigPre: /usr/share/openstack-tripleo-heat-templates/puppet/extraconfig/pre_deploy/controller/neutron-nuage.yaml

parameter_defaults:
  NeutronNuageOSControllerIp: '192.0.2.13'
  NeutronNuageNetPartitionName: 'Nuage_Partition'
  NeutronNuageVSDIp: '192.0.2.100:8443'
  NeutronNuageVSDUsername: 'csproot'
  NeutronNuageVSDPassword: 'csproot'
  NeutronNuageVSDOrganization: 'csp'
  NeutronNuageBaseURIVersion: 'v3_2'
  NeutronNuageCMSId: ''
  UseForwardedFor: true
```
where NeutronNuageCMSId needs to be generated and provided here for Nuage version 3.2R4 onwards

### nova-nuage-config.yaml
included [here](https://github.com/nuagenetworks/ospd-experimental/blob/master/tripleo/environments/nova-nuage-config.yaml)
```
# A Heat environment file which can be used to enable a
# a Nova Nuage backend, configured via puppet
resource_registry:
  OS::TripleO::ComputeExtraConfigPre: /usr/share/openstack-tripleo-heat-templates/puppet/extraconfig/pre_deploy/compute/nova-nuage.yaml

parameter_defaults:
  NuageActiveController: '192.0.2.101'
  NuageStandbyController: '192.0.2.102'
```

### neutron-generic.yaml for non-HA
```
resource_registry:
  OS:TripleO:ControllerDeployment: /usr/share/openstack-tripleo-heat-templates/puppet/controller-puppet.yaml

parameter_defaults:
  NeutronCorePlugin: 'neutron.plugins.nuage.plugin.NuagePlugin'
  NeutronServicePlugins: ''
  ControlVirtualInterface: 'eth0'
  PublicVirtualInterface: 'eth0'
  NeutronMetadataProxySharedSecret: 'NuageNetworksSharedSecret'
  InstanceNameTemplate: 'inst-%08x'
```

### neutron-generic.yaml for HA
```
resource_registry:
  OS::TripleO::ControllerDeployment: /usr/share/openstack-tripleo-heat-templates/puppet/controller-puppet.yaml

parameter_defaults:
  NeutronCorePlugin: 'neutron.plugins.nuage.plugin.NuagePlugin'
  NeutronServicePlugins: ''
  ControlVirtualInterface: 'eth0'
  PublicVirtualInterface: 'eth0'
  NeutronEnableDHCPAgent: 'false'
  NeutronEnableL3Agent: 'false'
  NeutronEnableMetadataAgent: 'false'
  NeutronEnableOVSAgent: 'false'
  NeutronMetadataProxySharedSecret: 'NuageNetworksSharedSecret'
  InstanceNameTemplate: 'inst-%08x'
  
```


### nova-generic.yaml
```
resource_registry:
  OS:TripleO:Compute: /usr/share/openstack-tripleo-heat-templates/puppet/compute-puppet.yaml

parameter_defaults:
  NeutronCorePlugin: 'neutron.plugins.nuage.plugin.NuagePlugin'
  NovaOVSBridge: 'alubr0'
  NovaSecurityGroupAPI: 'neutron'
  NovaComputeLibvirtType: 'qemu'
  NuageMetadataProxySharedSecret: 'NuageNetworksSharedSecret'
```


