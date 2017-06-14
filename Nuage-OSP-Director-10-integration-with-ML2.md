# Introduction

This document outlines the architecture of the integration project of using ML2 with Nuage as mechanism driver with OSP Director 10.  

# OSP Director 10 integration with Ml2 and Nuage as mechanism driver 
This document will focus on providing the information required to add and configure ML2 and Nuage.
 
The OSP Director is an image based installer. It uses a single image (named overcloud-full.qcow2) that is deployed on the Controller and Compute machines belonging to the overcloud OpenStack cluster. This image contains all the packages that are needed during the deployment. The deployment only creates the configuration files and databases required by the different services and starts the services in the correct order. Typically, there is no new software installation during the deployment phase. The packages/files required by ML2 will be added to this image as well.

The OSP Director architecture allows partners to create new templates to expose parameters specific to their modules and then the templates can be passed to the `openstack ovecloud deploy` command during the deployment. 
Additionally, changes to the puppet [manifests](http://git.openstack.org/cgit/openstack/tripleo-heat-templates/tree/puppet) are required to handle the new values in the Hiera database and act on them to deploy the partner software. ML2 options will be added to the existing Nuage templates.

# ML2 and SRIOV
This feature allows an OpenStack installation to support Single Root I/O Virtualization (SR-IOV)-attached VMs (https://wiki.openstack.org/wiki/SR-IOV-Passthrough-For-Networking) with VSP-managed VMs on the same KVM hypervisor cluster. It provides a Nuage ML2 mechanism driver that coexists with the sriovnicswitch mechanism driver.

Neutron ports attached through SR-IOV are configured by the sriovnicswitch mechanism driver. Neutron ports attached to Nuage VSD-managed networks are configured by the Nuage ML2 mechanism driver.



# Integration of Nuage VSP with OSP Director

The integration of Nuage VSP with OSP Director involves the following steps:

### OSP Director 9.0
For OSP Director 9.0, the changes required to create and modify the plugin.ini file is upstreamed at [this review](https://review.openstack.org/#/c/372757/). This review contains new code in manifests/plugins directory with the associated tests and custom resources. ID:  https://review.openstack.org/#/c/372757/. This change is not in OSP-Director 9.0 yet. The patching script mentioned below will take care of this change.

Secondly, since Nuage neutron package name changed for releases Mitaka and later, tripleo-heat-templates were modified and the changes are at [this review](https://review.openstack.org/#/c/372749/). This review contains the changes required to puppet files that enable Nuage specific code. ID: https://review.openstack.org/#/c/372749/. This change is also not in OSP-Director 9.0 yet.

## Modification of overcloud-full image   
Since the typical deployment scenario of OSP Director assumes that all the packages are installed on the overcloud-full image, we need to patch the overcloud-full image with the following RPMs:  
* nuagenetlib  
* nuage-openstack-neutron  
* nuage-openstack-neutronclient  
* nuage-nova-extensions  
* nuage-metadata-agent  
* nuage-puppet-modules-2.0  
* openstack-neutron-sriov-nic-agent  
* lldpad

Also, we need to un-install OVS and Install VRS
* Un-install OVS  
* Install VRS (nuage-openvswitch)  

The installation of packages and un-installation of OVS can be done via [this script](https://github.com/nuagenetworks/nuage-ospdirector/blob/ML2-SRIOV/image-patching/stopgap-script/nuage_overcloud_full_patch.sh).  
Since the files required to configure plugin.ini, neutron.conf, ml2_conf.ini, ml2_conf_sriov.ini, nova.conf and sriov_agent.ini are not in the OSP-Director codebase, the changes can be added to the image using the same [script](https://github.com/nuagenetworks/nuage-ospdirector/blob/ML2-SRIOV/image-patching/stopgap-script/nuage_overcloud_full_patch.sh). Copy the directory containing the 9_files and the script at [this link](https://github.com/nuagenetworks/nuage-ospdirector/tree/ML2-SRIOV/image-patching/stopgap-script) and execute the script. For the next release this code will be upstreamed.

## Generic changes to openstack-tripleo-heat-templates   
Changes are required in controller and compute puppet files to enable ML2 as core plugin and Nuage and SRIOV as mechanism drivers. These changes are done in [openstack-tripleo-heat-templates/puppet/manifests/overcloud_controller_pacemaker.pp](https://github.com/nuagenetworks/nuage-ospdirector/blob/ML2-SRIOV/openstack-tripleo-heat-templates/puppet/manifests/overcloud_controller_pacemaker.pp#L868-L878) for Controller nodes and for Compute nodes in [openstack-tripleo-heat-templates/puppet/manifests/overcloud_compute.pp](https://github.com/nuagenetworks/nuage-ospdirector/blob/ML2-SRIOV/openstack-tripleo-heat-templates/puppet/manifests/overcloud_compute.pp#L179-L215). Some of the generic neutron.conf and nova.conf parameters need to be configured in the heat templates. Also, the metadata agent needs to be configured. All the generic neutron and nova parameters and their 'probable' values are specified in files neutron-generic.yaml and nova-generic.yaml under the "Sample Templates" section below.

## Changes to openstack-tripleo-heat-templates specific to Nuage
The tripleo-heat-templates repository needs the extraconfig templates to configure the Nuage specific parameters. Also, ML2 and SRIOV specific parameters are added to these files, till they are upstreamed in tripleo-heat-templates repository. The Controller node extraconfig parameter file is at [openstack-tripleo-heat-templates/puppet/extraconfig/pre_deploy/controller/neutron-nuage.yaml](https://github.com/nuagenetworks/nuage-ospdirector/blob/ML2-SRIOV/openstack-tripleo-heat-templates/puppet/extraconfig/pre_deploy/controller/neutron-nuage.yaml). The compute node extraconfig parameter file is at [openstack-tripleo-heat-templates/puppet/extraconfig/pre_deploy/compute/nova-nuage.yaml](https://github.com/nuagenetworks/nuage-ospdirector/blob/ML2-SRIOV/openstack-tripleo-heat-templates/puppet/extraconfig/pre_deploy/compute/nova-nuage.yaml). The values of these parameters are dependent on the configuration of Nuage VSP and SRIOV. The "Sample Templates" section contains some 'probable' values that can be specified for Nuage specific parameters in files neutron-nuage-config.yaml and nova-nuage-config.yaml.

## HA changes
For Nuage VSP with OpenStack HA, we need to disable the default services like openvswitch-agent and dhcp-agent from being controlled via Pacemaker. The flags to disable these services are also present in the neutron-generic.yaml file.

## Neutron Metadata configuration and VRS configuration  
A new puppet module is needed to create and populate the metadata agent config file and the VRS configuration in /etc/default/openvswitch. nuage-metadata-agent module will be included in Nuage-puppet-modules, along with other required Nuage packages. The section "Modification of overcloud-full image" mentions the steps for including Nuage-puppet-modules in the overcloud-full image used for Overcloud deployment.

# Deployment steps

## Modify overcloud-full.qcow2 to include Nuage components
The customer will receive all the RPMs and the script to patch the overcloud-full image with the RPMs. The user needs to create a local repo that is accessible from the machine that the script will run on and add all the RPMs to that repo. The machine also needs lib-guestfs-tools installed.
The script syntax is: `source nuage_overcloud_full_patch.sh --RhelUserName=<value>  --RhelPassword='<value>' --RepoName=Nuage --RepoBaseUrl=http://IP/reponame --RhelPool=<value> --ImageName='<value>' --Version=9`  
This script takes in following input parameters:  
  RhelUserName: User name for the RHEL subscription    
  RhelPassword: Password for the RHEL subscription    
  RhelPool: RHEL Pool to subscribe to for base packages  
  RepoName: Name for the local repo hosting the Nuage RPMs  
  RepoBaseUrl: Base URL for the repo hosting the Nuage RPMs  
  ImageName: Name of the qcow2 image (overcloud-full.qcow2 for example)  
  Version: OSP-Director version (9)

## Deploy undercloud 
The undercloud deployment should proceed as per the OSP Director documentation. Follow all the steps before the `openstack overcloud deploy` command.  


### Generate CMS ID

For an Openstack installation, a CMS (Cloud Management System) ID needs to be generated to configure with Nuage VSD installation. The assumption is that Nuage VSD and Nuage VSC are already running before overcloud is deployed.

Steps to generate it:  
* Copy the [folder](https://github.com/nuagenetworks/nuage-ospdirector/tree/ML2-SRIOV/generate-cms-id) to a machine that can reach VSD (typically the undercloud node)  
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

**openstack overcloud deploy --templates -e neutron-nuage-config.yaml -e neutron-generic.yaml -e nova-nuage-config.yaml -e nova-generic.yaml--control-scale 1 --compute-scale 1 --ceph-storage-scale 0 --block-storage-scale 0 --swift-storage-scale 0**

For Virtual deployment, need to add --libvirt-type parameter as:

**openstack overcloud deploy --templates --libvirt-type qemu -e neutron-nuage-config.yaml -e neutron-generic.yaml -e nova-nuage-config.yaml -e nova-generic.yaml--control-scale 1 --compute-scale 1 --ceph-storage-scale 0 --block-storage-scale 0 --swift-storage-scale 0**

where:  
neutron-nuage-config.yaml: Nuage specific controller parameter values  
neutron-generic.yaml: Values for neutron config parameters as CorePlugin and ServicePlugins  
nova-nuage-config.yaml: Nuage specific compute parameter values  
nova-generic.yaml: Values for nova config parameters as LibvirtVifDriver, OVSBridge, SecurityGroupApi, etc.  

### HA
For HA deployment, following command was used for deploying with Nuage:

**openstack overcloud deploy --templates -e neutron-nuage-config.yaml -e neutron-generic.yaml -e nova-nuage-config.yaml -e nova-generic.yaml--control-scale 2 --compute-scale 2 --ceph-storage-scale 0 --block-storage-scale 0 --swift-storage-scale 0 --ntp-server ntp.zam.alcatel-lucent.com**

For Virtual deployment, need to add --libvirt-type parameter as:

**openstack overcloud deploy --templates --libvirt-type qemu -e neutron-nuage-config.yaml -e neutron-generic.yaml -e nova-nuage-config.yaml -e nova-generic.yaml--control-scale 2 --compute-scale 2 --ceph-storage-scale 0 --block-storage-scale 0 --swift-storage-scale 0 --ntp-server ntp.zam.alcatel-lucent.com**

where:  
neutron-nuage-config.yaml: Nuage specific controller parameter values as well as services disabling parameters  
neutron-generic.yaml: Values for neutron config parameters as CorePlugin and ServicePlugins  
nova-nuage-config.yaml: Nuage specific compute parameter values  
nova-generic.yaml: Values for nova config parameters as LibvirtVifDriver, OVSBridge, SecurityGroupApi, etc.  

## TripleO Heat Template code changes
Changes to openstack-tripleo-heat-templates will be provided in a tar file till these are upstreamed

## Sample Templates

### neutron-nuage-config.yaml
```
# A Heat environment file which can be used to enable a
# a Neutron Nuage backend on the controller, configured via puppet
resource_registry:
  OS::TripleO::ControllerExtraConfigPre: ../puppet/extraconfig/pre_deploy/controller/neutron-nuage.yaml

parameter_defaults:
  NeutronNuageOSControllerIp: '0.0.0.0'
  NeutronNuageNetPartitionName: 'Nuage_Partition'
  NeutronNuageVSDIp: '192.0.2.112:8443'
  NeutronNuageVSDUsername: 'csproot'
  NeutronNuageVSDPassword: 'csproot'
  NeutronNuageVSDOrganization: 'csp'
  NeutronNuageBaseURIVersion: 'v4_0'
  NeutronNuageCMSId: 'e52717dd-251c-475a-b1aa-7bb689d1c9de'
  UseForwardedFor: true
  NeutronNuageDBSyncExtraParams: '--config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini --config-file /etc/neutron/plugins/nuage/plugin.ini'
  NeutronFlatNetworks: '*'
  NeutronNuagePluginsML2FirewallDriver: ''
  NeutronNuagePluginsML2SupportedPCIVendorDevs: "8086:1572, 8086:1521"
  NeutronNuagePluginsML2SriovAgentRequired: true
  NovaNuageMonkeyPatch: true
  NovaNuageMonkeyPatchModules: 'nova.network.neutronv2.api:nuage_nova_extensions.nova.network.neutronv2.api.decorator'
```

### nova-nuage-config.yaml
```
# A Heat environment file which can be used to enable
# Nuage backend on the compute, configured via puppet
resource_registry:
  OS::TripleO::ComputeExtraConfigPre: ../puppet/extraconfig/pre_deploy/compute/nova-nuage.yaml

parameter_defaults:
  NuageActiveController: '192.0.2.120'
  NuageStandbyController: '0.0.0.0'
  NuageSriovInterfaceNames: 'eno2,eno3'
  NuageNumberOfVFs: 7
  NuageNovaComputePCIPassthroughWhitelist: "'[{\"devname\":\"eno2\",\"physical_network\":\"physnet1\"},{\"devname\":\"eno3\",\"physical_network\":\"physnet2\"}]'"
  NuageNeutronAgentsML2PhysicalDeviceMappings: 'physnet1:eno2, physnet2:eno3'
  NuageNeutronAgentsML2FirewallDriver: 'neutron.agent.firewall.NoopFirewallDriver'
```

### neutron-generic.yaml
```
resource_registry:
  OS::TripleO::ControllerDeployment: /usr/share/openstack-tripleo-heat-templates/puppet/controller.yaml

parameter_defaults:
  NeutronCorePlugin: 'neutron.plugins.ml2.plugin.Ml2Plugin'
  NeutronServicePlugins: 'NuagePortAttributes,NuageNetTopology'
  NeutronNetworkType: 'vxlan,vlan,flat'
  NeutronPluginExtensions: "nuage_subnet,nuage_port"
  NeutronTypeDrivers: "vlan,vxlan,flat"
  NeutronMechanismDrivers: "nuage,nuage_hwvtep,sriovnicswitch"
  NeutronTunnelIdRanges: "1:1000"
  NeutronNetworkVLANRanges: "physnet1:1:1000,physnet2:1:1000"
  NeutronVniRanges: "1001:2000"
  NeutronEnableDHCPAgent: 'false'
  NeutronEnableL3Agent: 'false'
  NeutronEnableMetadataAgent: 'false'
  NeutronEnableOVSAgent: 'false'
  NeutronMetadataProxySharedSecret: 'NuageNetworksSharedSecret'
  InstanceNameTemplate: 'inst-%08x'
```

### nova-generic.yaml for Virtual Setup
```
resource_registry:
  OS:TripleO:Compute: /usr/share/openstack-tripleo-heat-templates/puppet/compute.yaml

parameter_defaults:
  NeutronCorePlugin: 'neutron.plugins.ml2.plugin.Ml2Plugin'
  NeutronMechanismDrivers: 'nuage,nuage_hwvtep,sriovnicswitch'
  NovaOVSBridge: 'alubr0'
  NovaSecurityGroupAPI: 'neutron'
  NovaComputeLibvirtType: 'qemu'
  NovaIPv6: True
  NuageMetadataProxySharedSecret: 'NuageNetworksSharedSecret'
  NuageNovaApiEndpoint: 'internalURL'
```

### nova-generic.yaml for Baremetal Setup
```
resource_registry:
  OS:TripleO:Compute: /usr/share/openstack-tripleo-heat-templates/puppet/compute.yaml

parameter_defaults:
  NeutronCorePlugin: 'neutron.plugins.ml2.plugin.Ml2Plugin'
  NeutronMechanismDrivers: 'nuage,nuage_hwvtep,sriovnicswitch'
  NovaOVSBridge: 'alubr0'
  NovaSecurityGroupAPI: 'neutron'
  NovaComputeLibvirtType: 'kvm'
  NovaIPv6: True
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
```
NeutronNuageOSControllerIp
This is a deprecated parameter, but some of OSP Director 8 code still considers it as mandatory parameter. So, it needs to be specified but the value is not used anymore.
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
```
NovaNuageMonkeyPatch
Maps to monkey_patch parameter in [DEFAULT] section
```
```
NovaNuageMonkeyPatchModules
Maps to monkey_patch_modules parameter in [DEFAULT] section
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
The following parameters are mapped to values in /etc/neutron/plugins/ml2/ml2_conf_sriov.ini file on the neutron controller
```
NeutronNuagePluginsML2SupportedPCIVendorDevs
Maps to supported_pci_vendor_devs parameter in [ml2_sriov] section
```
```
NeutronNuagePluginsML2SriovAgentRequired
Maps to agent_required parameter in [ml2_sriov] section
```
The following parameters are used for setting/disabling values in undercloud's puppet code
```
ControlVirtualInterface
PublicVirtualInterface
These parameters map to the management interface name of the undercloud node
```
```
NeutronEnableDHCPAgent
NeutronEnableL3Agent
NeutronEnableMetadataAgent
NeutronEnableOVSAgent
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
NovaSecurityGroupAPI
Maps to security_group_api in [DEFAULT] section
```
```
NovaComputeLibvirtType
Maps to virt_type parameter in [libvirt] section
```
```
NuageNovaComputePCIPassthroughWhitelist
Maps to pci_passthrough_whitelist in [DEFAULT] section
White list of PCI devices available to VMs
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
The following parameters are mapped to values in /etc/neutron/plugins/ml2/sriov_agent.ini on the nova compute
```
NuageNeutronAgentsML2PhysicalDeviceMappings
Maps to physical_device_mappings in [sriov_nic] section
```
```
NuageNeutronAgentsML2FirewallDriver
Maps to firewall_driver in [securitygroup] section
```
The following parameters are used to configure LLDP and Virtual Functions (VF) on SRIOV supported NICs
```
NuageSriovInterfaceNames
Interface names on which SRIOV needs to be configured
```
```
NuageNumberOfVFs
Number of Virtual Functions to be supported by each SRIOV supported NIC
```
The following parameters are used for setting/disabling values in undercloud's puppet code
```
NeutronMechanismDrivers
Used for verification of Nuage as one of the Mechanism Drivers when core_plugin is ML2
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