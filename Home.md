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
This can be done via [this script](https://github.com/nuagenetworks/ospd-experimental/blob/master/image-patching/nuage_overcloud_full_patch.sh).

In addition to this puppet-neutron and puppet-nova changes need to be manually patched to the overcloud-full.qcow2 image using guestfish.

### puppet-neutron changes
Puppet-Neutron directory is provided [here](https://github.com/nuagenetworks/ospd-experimental/tree/master/puppet-neutron). The files that need to be added to overcloud-full.qcow2 image are:

* lib/puppet/provider/neutron_plugin_nuage/ini_setting.rb
* lib/puppet/type/neutron_plugin_nuage.rb
* manifests/plugins/nuage.pp

The files that need to be modified are:

* manifests/config.pp
* manifests/params.pp

## Configuring plugin.ini on the Controller
Puppet-neutron is a puppet module that configures Neutron and Neutron plugins. This module already has code to configure and maintain the /etc/neutron/neutron.conf file.
New code needs to be added to configure the plugin.ini. The changes to create and modify the plugin.ini file is being upstreamed at [this review](https://review.openstack.org/#/c/214798/). This review contains new code in manifests/plugins/nuage directory with the associated tests and custom resources. ID:  https://review.openstack.org/#/c/214798/

## Generic changes to openstack-tripleo-heat-templates
Some of the generic nova.conf parameters need to be exposed in the heat templates. This can found at [this review](https://review.openstack.org/#/c/229649/). ID: https://review.openstack.org/#/c/229649
The metadata agent needs instance_name_template configured. This is at [this review](https://review.openstack.org/#/c/234965/). ID: https://review.openstack.org/#/c/234965/

The neutron::core_plugin is already exposed via the parameter NeutronCorePlugin in the template files. So there is no change required for that.


## Changes to openstack-tripleo-heat-templates specific to Nuage
The tripleo-heat-templates repo needs the extraconfig templates to configure the Nuage specific parameters. Additionally, the puppet orchestration files (controller.pp, controller-pacemaker.pp and compute.pp) need changes to perform Nuage configuration. These changes are at [this review] (https://review.openstack.org/#/c/230116/). ID: https://review.openstack.org/#/c/230116/.
The Nuage metadata agent also needs to be configured via the Heat Templates and the puppet orchestration files. These changes can be found at [this review](https://review.openstack.org/#/c/234970/) ID: https://review.openstack.org/#/c/234970/


## Modification to puppet-nova
Puppet-nova is a puppet module that configures Nova on the controller and computes. All the parameters except the instance_name_template are exposed in puppet-nova already. The change to expose this parameter is at [this review](https://review.openstack.org/#/c/234972/) ID: https://review.openstack.org/#/c/234972/

## HA changes
For Nuage VSP with OpenStack HA, we need to disable the default services like openvswitch-agent and dhcp-agent from being controlled via Pacemaker. These change are at [this review](https://review.openstack.org/#/c/230764/). ID: https://review.openstack.org/#/c/230764/

## Neutron Metadata configuration and VRS configuration
A new puppet module is need to create and populate the metadata agent config file and the VRS configuration in /etc/default/openvswitch. This new puppet module will be called from the orchestration layer described in the "Changes to openstack-tripleo-heat-templates specific to Nuage" section earlier in this document. The code for the puppet module can be found at [this link](https://github.com/nuagenetworks/NuagePuppetModules). Note: Copy paste this link if the hyperlink does not work https://github.com/nuagenetworks/NuagePuppetModules

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

Change the following 2 lines as shown below:
```
OS::TripleO::ControllerExtraConfigPre: puppet/extraconfig/pre_deploy/controller/neutron-nuage.yaml
OS::TripleO::ComputeExtraConfigPre: puppet/extraconfig/pre_deploy/compute/nova-nuage.yaml
```


## Overcloud Deployment commands
For OSP Director, tuskar deployment commands are recommended. But as part of Nuage integration effort, it was found that heat-templates provide more options and customization to overcloud deployment. The templates can be passed in "openstack overcloud deploy" command line options and can create or update an overcloud deployment.

### Non-HA
For non-HA overcloud deployment, following command was used for deploying with Nuage:

**openstack overcloud deploy --templates -e neutron-nuage-config.yaml -e neutron-generic.yaml -e nova-nuage-config.yaml -e nova-generic.yaml--control-scale 1 --compute-scale 1 --ceph-storage-scale 0 --block-storage-scale 0 --swift-storage-scale 0**

where:
neutron-nuage-config.yaml: Nuage specific controller parameter values
neutron-generic.yaml: Values for neutron config parameters as CorePlugin and ServicePlugins
nova-nuage-config.yaml: Nuage specific compute parameter values
nova-generic.yaml: Values for nova config parameters as OVSBridge, SecurityGroupApi, etc.

### HA
For HA deployment, following command was used for deploying with Nuage:

**openstack overcloud deploy --templates -e neutron-nuage-config.yaml -e neutron-generic.yaml -e nova-nuage-config.yaml -e nova-generic.yaml--control-scale 2 --compute-scale 2 --ceph-storage-scale 0 --block-storage-scale 0 --swift-storage-scale 0**

where:
neutron-nuage-config.yaml: Nuage specific controller parameter values as well as services disabling parameters
neutron-generic.yaml: Values for neutron config parameters as CorePlugin and ServicePlugins
nova-nuage-config.yaml: Nuage specific compute parameter values
nova-generic.yaml: Values for nova config parameters as LibvirtVifDriver, OVSBridge, SecurityGroupApi, etc.

## Sample Templates
### neutron-nuage-config.yaml
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
```

### nova-nuage-config.yaml
```
# A Heat environment file which can be used to enable a
# a Nova Nuage backend, configured via puppet
resource_registry:
  OS::TripleO::ComputeExtraConfigPre: /usr/share/openstack-tripleo-heat-templates/puppet/extraconfig/pre_deploy/compute/nova-nuage.yaml

parameter_defaults:
  NuageActiveController: '192.0.2.101'
  NuageMetadataPort: '9697'
  NuageMetadataProxySharedSecret: 'NuageNetworksSharedSecret'
  NuageNovaMetadataPort: '8775'
  NuageNovaClientVersion: '2'
  NuageNovaOsUsername: 'nova'
  NuageMetadataAgentStartWithOvs: true
  NuageNovaApiEndpoint: 'publicURL'
  NuageNovaRegionName: 'regionOne'
```

### neutron-generic.yaml for non-HA
```
resource_registry:
  OS:TripleO:ControllerDeployment: /usr/share/openstack-tripleo-heat-templates/puppet/controller-puppet.yaml

parameter_defaults:
  NeutronCorePlugin: 'neutron.plugins.nuage.plugin.NuagePlugin'
  NeutronServicePlugins: ''
```

### neutron-generic.yaml for HA
```
resource_registry:
  OS::TripleO::ControllerDeployment: /usr/share/openstack-tripleo-heat-templates/puppet/controller-puppet.yaml

parameter_defaults:
  NeutronCorePlugin: 'neutron.plugins.nuage.plugin.NuagePlugin'
  NeutronServicePlugins: ''
  EnableDHCPAgent: False
  EnableL3Agent: False
  EnableMetadataAgent: False
  EnableOvsAgent: False
```


### nova-generic.yaml
```
resource_registry:
  OS:TripleO:Compute: /usr/share/openstack-tripleo-heat-templates/puppet/compute-puppet.yaml

parameter_defaults:
  NovaOVSBridge: 'alubr0'
  NovaSecurityGroupAPI: 'neutron'
```


