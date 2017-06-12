This step-by-step howto describes installation and configuration of OVSvAPP
Neutron agent. Please, refer to following link for main description of
OVSvAPP architecture and network mechanisms:

https://wiki.openstack.org/wiki/Neutron/Networking-vSphere


### Configure Neutron server

0. Setup prerequisites
    ```
    sudo apt-get install python python-pip git
    ```

1. Setup Networking-vSphere package
    ```
    git clone https://github.com/openstack/networking-vsphere.git
    cd ~/networking-vsphere
    git fetch
    git checkout liberty-eol
    sudo pip install -r requirements.txt
    sudo python setup.py install
    ```

2. Allow OVSvAPP network mechanism on Neutron/Network server/s. Edit `/etc/neutron/plugins/ml2/ml2_conf.ini`
    ```
    ...

    [ml2]

    ...

    # (ListOpt) Ordered list of networking mechanism driver entrypoints
    # to be loaded from the neutron.ml2.mechanism_drivers namespace.
    # mechanism_drivers =
    # Example: mechanism_drivers = openvswitch,mlnx
    # Example: mechanism_drivers = arista
    # Example: mechanism_drivers = cisco,logger
    # Example: mechanism_drivers = openvswitch,brocade
    # Example: mechanism_drivers = linuxbridge,brocade
    mechanism_drivers = openvswitch,ovsvapp

    ...
    ```

3. Restart Neutron service:
    ```
    sudo service neutron-server restart
    ```


### Configure VMware vSphere

1. Create distributed virtual switch `int-dvs`, one per each ESX/ESXi host, **without** any VNICs added into uplinks (only host should be added).

2. Create port group `int-dvs-trunk` on DVS `int-dvs` configured as a VLAN trunk with VLAN IDs range of tenant VLANs (500-2000).

3. Create distributed virtual switch `ext-dvs` with configured uplink, connected to physical tenants network.


4. Create port group `ext-dvs-trunk` configured as VLAN trunk with VLAN IDs range of tenant VLANs (500-2000).

5. Create VM (OVSvAPP), one per each ESX/ESXi host, with following network configuration:
    - network adapter 1 (VM NIC eth0) bound to port group `int-dvs-trunk`
    - network adapter 2 (VM NIC eth1) bound to port group `ext-dvs-trunk`
    - network adapter 3 (VM NIC eth2) bound to management network


6. Install clean Ubuntu 14.04 server on each OVSvAPP VM.


### Setup OVSvAPP VM and Neutron agent (one per ESX/ESXi host in cluster)

Following instructions should be executed on OVSvAPP VM.

0. Setup prerequisites
    ```
    sudo apt-get install python python-pip git
    ```

1. Setup required packages
    ```
    sudo apt-get install neutron-common neutron-plugin-ml2 openvswitch-switch -y
    ```

2. Setup Networking-vSphere package
    ```
    git clone https://github.com/openstack/networking-vsphere.git
    cd ~/networking-vsphere
    git fetch
    git checkout liberty-eol
    sudo pip install -r requirements.txt
    sudo python setup.py install
    ```

3. Configure Neutron and OVSvAPP agent

    - Edit `/etc/neutron/neutron.conf`; make the config the same as on other network nodes (Keystone auth info, RabbitMQ connection and etc)

    - Edit `/etc/neutron/plugins/ml2/ovsvapp_agent.ini`; add following options:
      ```
      [vmware]
      # Provide IP address for vCenter.
      vcenter_ip = <vcenter_host_or_ip>

      # Provide vCenter Credentials.
      vcenter_username = <vcenter_username>
      vcenter_password = <vcenter_password>

      # SSL communication between OVSvApp Agent and vCenter Server
      # cert_check = False
      # Provide Certificate file location.
      # cert_path =

      # Customized https_port for OVSvApp agent and vCenter Server communication
      # https_port = 443

      # Number of retries by OVSvApp Agent to connect to vCenter server
      # vcenter_api_retry_count = 2

      # wsdl_location =
      # vCenter server wsdl location
      # Example:
      # wsdl_location=https://<vCenter_ip>:<https_port>/sdk/vimService.wsdl
      wsdl_location = <wsdl_location>

      # Provide Cluster to DVS/vDS mapping.
      # cluster_dvs_mapping =
      #
      # Example:
      # cluster_dvs_mapping = <DatacenterName>/host/<ClusterName>:<vDSName>
      #
      # NOTE #1: 'host' is a simple literal in DVS mapping, not hostname or IP!
      #
      # NOTE #2: vDSName is the name of **internal** switch (`int-dvs`)
      #
      cluster_dvs_mapping = <cluster_dvs_mapping>

      # Provide ESX host name or IP address on which this OVSvAPP resides.
      esx_hostname = <esx_hostname>

      # To set the ESX host in maintenance mode during Fault Management.
      # esx_maintenance_mode = True


      [ovsvapp]

      tunnel_types = vlan

      # Network type for tenant networks.
      # tenant_network_type =
      # OVSvApp Agent supports either VLAN or VXLAN.
      # Example:
      tenant_network_type = vlan

      # Local IP address of VXLAN tunnel endpoint.
      #local_ip = 10.21.0.21

      # OVS integration bridge
      integration_bridge = br-int

      # Provide bridge mappings for VLAN networks.
      #
      # Example:
      # bridge_mappings = physnet:br-eth1
      # where `eth1` is data interface and `physnet` is name of **physical**
      # network as it defined in Neutron.
      #
      bridge_mappings = <bridge_mappings>


      # IP address for monitoring OVS Status.
      # monitoring_ip =

      [securitygroup]
      # Provide bridge mapping for security groups.
      security_bridge_mapping = br-sec:eth0
      #
      # Example:
      # security_bridge_mapping = br-sec:eth2
      # where br-sec is default security bridge and eth2 is trunk interface.

      # Firewall driver for OVSvApp.
      ovsvapp_firewall_driver = networking_vsphere.drivers.ovs_firewall.OVSFirewallDriver

      # For disbaling security groups.
      # ovsvapp_firewall_driver = neutron.agent.firewall.NoopFirewallDriver

      ```

4. Preconfigure OVS bridges
    ```
    ovs-vsctl add-br br-sec
    ovs-vsctl add-br br-int
    ovs-vsctl add-br br-eth1

    # add trunk port group interface
    ovs-vsctl add-port br-sec eth0

    # configure patch-integration
    ovs-vsctl add-port br-sec patch-integration
    ovs-vsctl set interface patch-integration type=patch options:peer=patch-security

    # configure patch-security
    ovs-vsctl add-port br-int patch-security
    ovs-vsctl set interface patch-security type=patch options:peer=patch-integration

    # configure int-br-eth1
    ovs-vsctl add-port br-int int-br-eth1
    ovs-vsctl set interface int-br-eth1 type=patch options:peer=phy-br-eth1

    # configure phy-br-eth1
    ovs-vsctl add-port br-eth1 phy-br-eth1
    ovs-vsctl set interface phy-br-eth1 type=patch options:peer=int-br-eth1

    # add data port group inetrface
    ovs-vsctl add-port br-eth1 eth1
    ```
    NOTE: Ignore error messages on steps of creation of `patch-integration`, `patch-security`, `int-br-eth1` and `phy-br-eth1` ports.
    
    *Be aware: bridges names are mandatory!*


5. Start Neutron agent service
    ```
    sudo neutron-ovsvapp-agent --config-file /etc/neutron/neutron.conf \
    --config-file /etc/neutron/plugins/ml2/ovsvapp_agent.ini
    ```

    After this step new OVSvAPP agent should appear in list of Neutron agents.


### Configure Nova VMware compute host (one per vSphere cluster)

0. Setup prerequisites
    ```
    sudo apt-get install python python-pip git
    ```

1. Add CloudArchive/Liberty repo
    ```
    sudo add-apt-repository cloud-archive:liberty
    sudo apt-get update -y && sudo apt-get upgrade -y && sudo apt-get dist-upgrade -y
    ```

2. Setup Nova compute VMware plugin
    ```
    apt-get install nova-compute-vmware -y
    ```

3. Setup Networking-vSphere package
    ```
    git clone https://github.com/openstack/networking-vsphere.git
    cd ~/networking-vsphere
    git fetch
    git checkout liberty-eol
    sudo pip install -r requirements.txt
    sudo python setup.py install
    ```

    NOTE: OVSvAPP vCenter driver from `liberty-eol` branch might need
    following patch to be applied after install to the file
    `/usr/local/lib/python2.7/dist-packages/networking_vsphere/nova/virt/vmwareapi/ovsvapp_vc_driver.py`:

    ```
    --- ovsvapp_vc_driver.py.orig	2017-05-20 15:43:06.000000000 -0700
    +++ ovsvapp_vc_driver.py	2017-06-12 12:57:05.406469045 -0700
    @@ -22,7 +22,11 @@
     from nova import objects
     from nova.virt.vmwareapi import driver as vmware_driver
     from nova.virt.vmwareapi import images
    -from nova.virt.vmwareapi import ovsvapp_vmops as vmops  # noqa
    +try:
    +	from nova.virt.vmwareapi import ovsvapp_vmops as vmops  # noqa
    +except:
    +	import ovsvapp_vmops as vmops
    +
     from nova.virt.vmwareapi import vm_util

     LOG = log.getLogger(__name__)
     ```

4. Configure Nova compute

  - Edit `/etc/nova/nova.conf`; make the config as on other compute nodes (Keystone authorization, RabbitMQ connection, etc)

  - Edit `/etc/nova/nova-compute.conf`, add follwing options:
    ```
    [DEFAULT]
    compute_driver = networking_vsphere.nova.virt.vmwareapi.ovsvapp_vc_driver.OVSvAppVCDriver

    [vmware]
    host_ip = <vCenter_IP>
    host_username = <vCenter_username>
    host_password = <vCenter_password>
    cluster_name = <vCenter_cluster>
    #datastore_regex=
    #
    # wsdl_location = https://<vCenter_ip>:<https_port>/sdk/vimService.wsdl
    wsdl_location = <vCenter_wdsl_location>
    insecure = True
    vnc_port = 5900
    vnc_port_total = 10000
    ```

5. Restart Nova service:

    ```
    service nova-compute restart
    ```

    After this step, the new Nova compute service with `vmware` type should appear
    in list of Nova hypervisors.
