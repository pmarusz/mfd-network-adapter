> [!IMPORTANT]  
> This project is under development. All source code and features on the main branch are for the purpose of testing or evaluation and not production ready.


# MFD Network Adapter

## Table of contents
1. [Usage](#usage)
2. [Exceptions](#exceptions-raised-by-mfd-network-adapter-module)
3. [Classes](#classes)
   * [NetworkAdapterOwner](#networkadapterowner)
     * [Methods](#methods)
     * [Available features](#available-features)
       * [VLAN](#vlan)
       * [VxLAN](#vxlan)
       * [GRE](#gre)
       * [ARP](#arp)
       * [CPU](#cpu)
       * [DCB](#dcb)
       * [Driver](#driver)
       * [Firewall](#firewall)
       * [Interrupt](#interrupt)
       * [IP](#ip)
       * [Route](#route)
       * [Network Manager](#network-manager---nmcli)
       * [Virtualization](#virtualization)
       * [Queue](#queue)
       * [Utils](#utils)
       * [IPTables](#iptables)
       * [LinkAggregation](#link-aggregation-owner)
       * [MAC](#mac)
   * [NetworkInterface](#networkinterface)
     * [Common fields](#common-fields-of-networkinterface-)
     * [Linux fields](#additional-fields-of-linux-network-interface-)
     * [Windows fields](#additional-fields-of-windows-networkinterface-)
     * [Skipped interfaces](#what-type-of-interfaces-are-intentionally-skipped--linux-)
     * [Unsupported interfaces](#unsupported-interfaces--linux--)
     * [When to refresh interfaces?](#what-kind-of-actions-should-force-users-to-re-create-list-of-interfaces)
     * [Methods](#methods-1)
     * [Linux Methods](#additional-methods---linux)
     * [ESXi Methods](#additional-methods---esxi)
     * [Features](#features)
       * [Link](#link)
       * [IP](#ip-1)
       * [MTU](#mtu)
       * [Driver](#driver-1)
       * [Utils](#utils-1)
       * [Memory](#memory)
       * [NUMA](#numa)
       * [InterFrame](#interframe)
       * [RSS](#rss)
       * [Stats](#stats)
       * [LLDP](#lldp)
       * [Wol](#wol)
       * [Queue](#queue-1)
       * [Virtualization](#virtualization-1)
       * [StatChecker](#statchecker)
       * [Buffers](#buffers)
       * [DMA](#dma)
       * [Capture](#capture)
       * [VLAN](#vlan-1)
       * [Offload](#offload)
       * [ENS](#ens)
       * [NIC Team](#nic-team)
       * [MAC](#mac-1)
     * [Data structures](#networkinterface-data-structures-)
4. [Common Data structures](#common-data-structures)
5. [OS supported](#os-supported-)
6. [Static API](#static-api)
7. [Poolmon](#poolmon)
8. [Issue reporting](#issue-reporting)


## Usage
- More detailed examples:
  - [linux example](examples/linux_example.py)
  - [interface refresh example](examples/interface_refresh_example.py)

```python
from mfd_connect import SSHConnection
from mfd_network_adapter.network_adapter_owner.linux import NetworkAdapterOwner

connection = SSHConnection(ip='10.10.10.10', username='***', password='***')
owner = NetworkAdapterOwner(connection=connection)
interfaces = owner.get_interfaces()
```
## Exceptions raised by MFD-Network-Adapter module
- related to module:  `NetworkAdapterModuleException`
- related to Network Interface:  `InterfaceNameNotFound`, `IPException`, `IPAddressesNotFound`, `NetworkQueuesException`, `RDMADeviceNotFound`, `NumaNodeException`, `DriverInfoNotFound`, `FirmwareVersionNotFound`
- related to NetworkInterface's features: `VirtualizationFeatureException`
- 
## Classes

## `NetworkAdapterOwner`
Gathers system-level features related to networking. 
It implements methods for detecting and filtering NICs of the system. 

#### Class diagram:

```mermaid
classDiagram

NetworkAdapterOwner <|-- ESXiNetworkAdapterOwner
NetworkAdapterOwner <|-- FreeBSDNetworkAdapterOwner
LinuxNetworkAdapterOwner
LinuxNetworkAdapterOwner <|-- IPULinuxNetworkAdapterOwner
NetworkAdapterOwner <|-- LinuxNetworkAdapterOwner
NetworkAdapterOwner <|-- WindowsNetworkAdapterOwner
```

#### Methods:

- `get_interfaces`:  returns list of all detected Network Interfaces on the system.

To filter out specific Network Interfaces you can use following combinations of filters:
1) `pci_address`
2) (`pci_device`|`family`|`speed`) + `interface_indexes`
3) (`pci_device`|`family`|`speed`|`family`+`speed`) + (`random_interface`|`all_interfaces`)
4) (`random_interface`|`all_interfaces`) 
5) `interface_names`

- `family`:
  - key of `DEVICE_IDS` from `mfd-const` e.g. `CPK`, `FVL`
  - or `Family` Enum member from `mfd-const` e.g. `Family.FVL`, `Family.CPK`

- `speed`:
  - key of `SPEED_IDS` from `mfd-const` e.g. `@40G`, `@100G`
    - available `speed` str formats: `@40G`, `@40g`, `40`, `40G`, `40g`, `40giga`, `40Giga`, `40GIGA`, `40Gb`
    - basically it needs to match speed number with such regex: `pattern = r"@{0,1}(?P<speed>\d+)\D*"`
  - or `Speed` Enum member from `mfd-const` e.g. `Speed.G40`, `Speed.G100`

* source code:
```python
def get_interfaces(
        self,
        *,
        pci_address: Optional[PCIAddress] = None,
        pci_device: Optional[PCIDevice] = None,
        family: Optional[Union[str, Family]] = None,
        speed: Optional[Union[str, Speed]] = None,
        interface_indexes: Optional[List[int]] = None,
        interface_names: Optional[List[str]] = None,
        random_interface: Optional[bool] = None,
        all_interfaces: Optional[bool] = None,
        namespace: Optional[str] = None,
    ) -> List["NetworkInterface"]
```

* `Sorted Interfaces`: To get interfaces in the sorted order use sorted() built-in function to list of interfaces.
```python
list_of_interfaces = owner.get_interfaces()
sorted_list_of_interfaces = sorted(interfaces)
```

- `get_interface`:  returns single interface of network adapter.

Expected combinations are:
1) interface_name
2) pci_address
3) pci_device / family / speed + interface_index

* source code:
```python
def get_interface(
        self,
        *,
        pci_address: Optional[PCIAddress] = None,
        pci_device: Optional[PCIDevice] = None,
        family: Optional[Union[str, Family]] = None,
        speed: Optional[Union[str, Speed]] = None,
        interface_index: Optional[int] = None,
        interface_name: Optional[str] = None,
        namespace: Optional[str] = None,
    ) -> "NetworkInterface"
```

- `is_management_interface(ip: IPv4Interface)`: Validate if passed IP address is used by management interface.

[L]
- `load_driver_file(driver_filepath: 'Path', params: Optional[Dict])`: load file with driver to kernel using insmod, available usege of parameters to insmod

[L]
- `load_driver_module(driver_name: str, params: Optional[Dict])`: load module with driver to kernel using modprobe, available usege of parameters to modprobe

[L]
- `unload_driver_module(driver_name: str)`: unload driver using modprobe -r

[L]
- `reload_driver_module(driver_name: str, reload_time: float = 5, params: Optional[Dict])`: unload and load driver module with `reload_time` inactivity

[L]
- `create_vfs(interface_name: str, vfs_count: int)`: assign specified number of Virtual Functions to the Physical Function

[L]
- `delete_vfs(interface_name: str)`: delete all Virtual Functions assigned to the Physical Function.

- `get_pci_addresses_by_pci_device(self, pci_device: PCIDevice, namespace: Optional[str] = None) -> List[PCIAddress]`: Translate PCI Device to PCI Addresses.

- `get_pci_device_by_pci_address(self, pci_address: PCIAddress, namespace: Optional[str] = None) -> PCIDevice`: Translate PCI Address to PCI Device.

[W]
-`get_log_cpu_no(self) -> int`: Get the number of logical cpus.

[ESXi]
- `wait_for_interfaces_up(self, interfaces: list["NetworkInterface"], timeout: int = 30) -> None`: Wait for all interfaces become up.

[FreeBSD]
- `create_vfs(interface_name: str, vfs_count: int, config_dir: "Path | str", config_name: str = None)`: assign specified number of Virtual Functions to the Physical Function.

- `delete_vfs(interface_name: str, config_dir: "Path | str", remove_conf: bool, config_name: str = None)`: delete all Virtual Functions assigned to the Physical Function.

  - `add_vfs_to_config_file(
      self,
      interface_name: str,
      vfs_count: int,
      passthrough: bool = False,
      max_vlan_allowed: int | None = None,
      max_mac_filters: int | None = None,
      allow_promiscuous: bool = False,
      num_queues: int | None = None,
      mdd_auto_reset_vf: bool = False,
      config_dir: "Path | str" = "/home/user",
      mirror_src_vsi: int | None = None,
      config_name: str | None = None,
      mac_addr: tuple[str, ...] | None = None,
      allow_set_mac: bool = False,
      mac_anti_spoof: bool = True,
      **kwargs, 
        ) -> None:
  `: Add specified number of Virtual Functions to the conf file for the Physical Function. 

### Available features

#### VLAN
[Linux]
```python
create_vlan(        
    self,
    vlan_id: int,
    interface_name: str,
    vlan_name: Optional[str] = None,
    protocol: Optional[str] = None,
    reorder: bool = True,
    namespace_name: Optional[str] = None,
) -> ConnectionCompletedProcess
```
[Windows]
```python
create_vlan(
    self,
    vlan_id: int,
    method: str,
    interface_name: Optional[str] = None,
    interface_index: Optional[str] = None,
    nic_team_name: Optional[str] = None,
) -> ConnectionCompletedProcess
```
[FreeBSD]
```python
create_vlan(self, vlan_id: int, interface_name: str) -> ConnectionCompletedProcess
```

[Linux]
```python
remove_vlan(
    self,
    vlan_name: Optional[str] = None,
    vlan_id: Optional[int] = None,
    interface_name: Optional[str] = None,
    namespace_name: Optional[str] = None,
) -> ConnectionCompletedProcess:
```
[Windows]
```python
remove_vlan(self, vlan_id: int, method: str, interface_name: str, interface_index: Optional[str]) -> ConnectionCompletedProcess
```
[FreeBSD]
```python
remove_vlan(self, vlan_id: int) -> ConnectionCompletedProcess
```

[Linux]
```python
remove_all_vlans(self) -> None
```

[Linux]
```python
create_macvlan(self, interface_name: str, mac: MACAddress, macvlan_name: str) -> ConnectionCompletedProcess
```

[Linux]
```python
set_ingress_egress_map(self, interface_name: str, priority_map: str, direction: str, verify: bool = True) -> None
```

[Windows]
```python
list_vlan_ids(self, interface_name: str) -> list - Returns list of all VLAN IDs on provided interface.
```
[Windows]
```python
modify_vlan(self, vlan_id: int, nic_team_name: str, new_vlan_id: int, new_vlan_name: str) -> ConnectionCompletedProcess
```
#### VxLAN

[Linux] Create a VxLAN Tunnel.

```python
create_setup_vxlan(vxlan_name: str, ip_addr: Union[IPv4Interface, IPv6Interface], vni: int, group_addr: Union[IPv4Interface, IPv6Interface], interface_name: str, dstport: int, namespace_name: str | None = None) -> None
```

[Linux] Delete a VxLAN Tunnel.

```python
delete_vxlan(vxlan_name: str, namespace_name: str | None = None) -> None
```

[FreeBSD] Create a VxLAN Tunnel.

```python
create_setup_vxlan(local_ip_addr: Union[IPv4Interface, IPv6Interface], vni: int, group_addr: Union[IPv4Interface, IPv6Interface], interface_name: str, vxlan_ip_addr: Union[IPv4Interface, IPv6Interface]) -> Union[str, None]:
```

[FreeBSD] Delete a VxLAN Tunnel.

```python
delete_vxlan(vxlan_name: str) -> None:
```

#### GRE

[Linux] Create a GRE Tunnel.

```python
create_setup_gre(gre_tunnel_name: str, local_ip_addr: IPv4Interface | IPv6Interface, remote_ip_addr: IPv4Interface | IPv6Interface, interface_name: str, key_id: int, namespace_name: str | None = None) -> None
```

[Linux] Delete a GRE Tunnel.

```python
delete_gre(gre_tunnel_name: str, namespace_name: str | None = None) -> None
```

### ARP

[FreeBSD]

```python
get_arp_table(self, ip_ver: IPVersion = IPVersion.V4) -> Dict[Union[IPv4Interface, IPv6Interface], MACAddress]
```

[Linux] [Windows]

```python
get_arp_table(
    self, ip_ver: IPVersion = IPVersion.V4, allowed_states: Optional[List[str]] = None
) -> Dict[Union[IPv4Interface, IPv6Interface], MACAddress]
```

[FreeBSD]

```python
add_arp_entry(self, ip: Union[IPv4Interface, IPv6Interface], mac: MACAddress) -> "ConnectionCompletedProcess"
```

[Linux]

```python
add_arp_entry(
    self,
    interface: "LinuxNetworkInterface",
    ip: Union[IPv4Interface, IPv6Interface],
    mac: MACAddress,
) -> "ConnectionCompletedProcess"
```

[Windows]

```python
add_arp_entry(
    self, interface: "WindowsNetworkInterface", ip: Union[IPv4Interface, IPv6Interface], mac: MACAddress
) -> "ConnectionCompletedProcess"
```

[FreeBSD]

```python
del_arp_entry(self, ip: Union[IPv4Interface, IPv6Interface]) -> "ConnectionCompletedProcess"
```

[Linux]

```python
del_arp_entry(
    self, interface: "LinuxNetworkInterface", ip: Union[IPv4Interface, IPv6Interface], mac: MACAddress
) -> "ConnectionCompletedProcess"
```

[Windows]

```python
del_arp_entry(self, interface: "WindowsNetworkInterface", ip: Union[IPv4Interface, IPv6Interface]) -> "ConnectionCompletedProcess"
```

[ESXi]
```python
del_arp_entry(self, ip: IPv4Interface | IPv6Interface) -> "ConnectionCompletedProcess": - Delete an entry from ARP table
```

[Linux]

```python
send_arp(self, interface: "LinuxNetworkInterface", destination: IPv4Interface, count: int = 1) -> "ConnectionCompletedProcess"
```

[FreeBSD]

```python
send_arp(self, interface: "FreeBSDNetworkInterface", destination: IPv4Interface, count: int = 1) -> "ConnectionCompletedProcess"
```

[Windows] [arp-ping.exe](https://www.elifulkerson.com/projects/arp-ping.php) is required to run this method.

```python
send_arp(
    self,
    interface: "WindowsNetworkInterface",
    destination: IPv4Interface,
    arp_ping_path: Union[str, Path],
    count: int = 1,
) -> "ConnectionCompletedProcess"
```

[Linux]

```python
flush_arp_table(self, interface: "LinuxNetworkInterface") -> "ConnectionCompletedProcess"
```

[Linux]

```python
delete_permanent_arp_table(
    self, interface: "LinuxNetworkInterface", ip_ver: IPVersion = IPVersion.V4
) -> None
```

[Linux]

```python
set_arp_response(self, interface: "LinuxNetworkInterface", state: State) -> "ConnectionCompletedProcess"
```

[Linux]

```python
check_arp_response_state(self, interface: "LinuxNetworkInterface") -> State
```

[Windows]

**Methods**
- `read_arp_table() -> str`

  Reads all lines in arp table.

  **Parameters:**
  * `None`
  
  **Returns:**
  * `str` - command output (ConnectionCompletedProcess.stdout)

- `read_ndp_neighbors(self, ip: "IPv4Interface | IPv6Interface") -> str`

  Reads neighbor discovery table (ND Table).

  **Parameters:**

  * `ip`: IP address of entry (IPv4 or IPv6)
    **Returns:**
  * `str` - command output (ConnectionCompletedProcess.stdout)


### Bonding
[Linux] Load bonding module
```python
load(self, mode: str = "active-backup", miimon: int = 100, max_bonds: int = 1) -> None
```

[Linux] Get list of bond interfaces
```python
get_bond_interfaces(self) -> list[str]
```

[Linux] Attach network interface to bonding interface using ifenslave command
```python
connect_interface_to_bond(self, network_interface: str | LinuxNetworkInterface, bonding_interface: str | LinuxNetworkInterface) -> None
```

[Linux] Detach network interface from bonding interface using ifenslave command
```python
disconnect_interface_from_bond(self, network_interface: str | LinuxNetworkInterface, bonding_interface: str | LinuxNetworkInterface) -> None
```

[Linux] Attach network interface to bonding interface using alternative commands
```python
connect_interface_to_bond_alternative(self, network_interface: str | LinuxNetworkInterface, bonding_interface: str | LinuxNetworkInterface, mode: str = None, miimon: int = None) -> None
```

[Linux] Detach network interface from bonding interface using alternative commands
```python
disconnect_interface_from_bond_alternative(self, network_interface: str | LinuxNetworkInterface, bonding_interface: str | LinuxNetworkInterface) -> None
```

[Linux] Create bond interface
```python
create_bond_interface(self, bonding_interface: str | LinuxNetworkInterface) -> None
```

[Linux] Set bonding params
```python
set_bonding_params(self, bonding_interface: str | LinuxNetworkInterface, params: dict[BondingParams, str]) -> None:
```

[Linux] Set active child
```python
set_active_child(self, bonding_interface: str | LinuxNetworkInterface, network_interface: str | LinuxNetworkInterface) -> None
```

[Linux] Get active child
```python
get_active_child(self, bonding_interface: str | LinuxNetworkInterface) -> str
```

[Linux] Get bonding mode
```python
get_bonding_mode(self, bonding_interface: str | LinuxNetworkInterface) -> str
```

[Linux] Delete bond interface
```python
delete_bond_interface(self, bonding_interface: str | LinuxNetworkInterface, child_interfaces: list[str | LinuxNetworkInterface]) -> None
```

[Linux] Verify if provided network_interface is active child
```python
verify_active_child(self, bonding_interface: str | LinuxNetworkInterface, network_interface: str | LinuxNetworkInterface) -> bool
```

[Linux] Get children
```python
get_children(self, bonding_interface: str | LinuxNetworkInterface) -> list[str]
```

### CPU

[ESXi] Initiate ESXi performance statistic gathering. Samples are collected every 2 seconds.
```python
start_cpu_usage_measure(self, file_path: str = "cpu.csv") -> "RemoteProcess"
```

[ESXi] Ensure statistic collection process termination.
```pyhton
stop_cpu_measurement(self, process: "RemoteProcess") -> bool
```

[ESXi] Extract from esxtop batch file data regarding particular VM vCPU usage.
```pyhton
parse_cpu_measurement_output(self, name_vm: str, file_path: str) -> int
```

### DCB

DCB feature is an object of mfd-dcb, so it provides you all the API that mfd-dcb does.

[API](https://github.com/intel/mfd-dcb)

### Driver

[Linux] Load driver by module name using modprobe.
```python
load_module(*, module_name: str, params: Optional[str] = None) -> "ConnectionCompletedProcess"
```

[Linux] Load driver file using insmod.
```python
load_module_file( *, module_filepath: "Path", params: Optional[str] = None) -> "ConnectionCompletedProcess"
```

[Linux] Unload driver from kernel via modprobe.
```python
unload_module(*, module_name: str, params: Optional[str] = None, with_dependencies: bool = False) -> "ConnectionCompletedProcess"
```

[Linux] Reload module using modprobe.
```python
reload_module(*, module_name: str, reload_time: float = 5, params: Optional[str] = None, with_dependencies: bool = False) -> None
```

[ESXi] Load module with configuration parameters.
```python
load_module(*, module_name: str, params: str = None) -> "ConnectionCompletedProcess"
```

[ESXi] Load module with configuration parameters.
```python
unload_module(module_name: str) -> "ConnectionCompletedProcess"
```

[ESXi] Reload module in system.
```python
reload_module(*, module_name: str, reload_time: float = 5, params: str = None) -> None
```

[ESXi] Get module params.
```python
get_module_params(module_name: str) -> str
```

[ESXi] Get module params as dictionary, e.g.: {"vmdq": "1,1,0,0"}.
```python
get_module_params_as_dict(module_name: str) -> Dict[str, str]
```

[ESXi] Prepare string for module settings in format required to reload module command.
```python
prepare_module_param_options(module_name: str, param: str, values: List[str]) -> str)
```

[ESXi] Prepare a string with multiple param options for the reload module command.
```python
prepare_multiple_param_options(self, *, param_dict: Dict, module_name: str) -> str
```

[ESXi] Prepare values for interfaces which share same driver needed for driver reload with them. 
Value of param will be prepared for update for all interfaces using <driver_name>.
```python
prepare_values_sharing_same_driver(*, driver_name: str, param: str, value: int) -> str
```

[Windows]

`change_state_family_interfaces(*, driver_filename: str, enable: State.ENABLED) -> None`

Change state of all interfaces with the same driver - belong to the same NIC family.

**Parameters:**

- `driver_filename (str)`: driver filename to be used for changing state, e.g. 'v40e65.sys'

- `enable (bool):` `State.ENABLED` if enable NICs, `State.DISABLED` otherwise

**Returns:**

- `None`

[Esxi] Wait for all interfaces become loaded.
```python
wait_for_all_interfaces_load(self, driver_name: str) -> None
```

### Firewall

[Windows]
 - The structure created for firewall purpose to structurized possible options for Inbound and OutBound actions:
```python
class DefInOutBoundActions(Enum):
    """Available def_in_bound and def_out_bound actions for set firewall feature on Windows Owner."""

    NOTCONFIGURED = "NotConfigured"
    ALLOW = "Allow"
    BLOCK = "Block"
```


**Methods**
- `set_firewall_default_action(profile: list[str] = ["Domain", "Public", "Private"], def_inbound_action: DefInOutBoundActions = DefInOutBoundActions.ALLOW, 
def_outbound_action: DefInOutBoundActions = DefInOutBoundActions.ALLOW) -> str`

  Sets firewall default Inbound and Outbound action settings on given profile(s).

  **Parameters:**
  * `profile`: FW profile to set. default: ['Domain', 'Public', 'Private']
    `def_inbound_action`: Default Inbound Action. Possible values are stored in DefInOutBoundActions structure: NOTCONFIGURED, ALLOW, BLOCK.
    `def_outbound_action`: Default Outbound Action. Possible values: NotConfigured, Allow, Block
  
  **Returns:**
  * `str` - command output (ConnectionCompletedProcess.stdout)

- `set_firewall_profile(profile: list[str] = ("Domain", "Public", "Private"), enabled: bool | State = State.ENABLED) -> str`

  Enables or Disable the firewall on given profile(s).

  **Parameters:**

  * `profile`: FW profile to set. default: `['Domain', 'Public', 'Private']`
  * `enabled`: State.ENABLED or True for on, State.DISABLED or False for off
    **Returns:**
  * `str` - command output (ConnectionCompletedProcess.stdout)

### Interrupt

[ESXi] Set Interrupt moderation rate.
```python
set_interrupt_moderation_rate(self, *, driver_name: str, rxvalue: Optional[int] = None, txvalue: Optional[int] = None) -> None
```

### IP

[L] Create bridge.
```python
create_bridge(self, bridge_name: str) -> None
```

[L] Add interface to bridge.
```python
add_to_bridge(self, bridge_name: str, interface_name: str) -> None
```

[L] Create namespace.
```python
create_namespace(self, namespace_name: str) -> None
```

[L] Delete namespace.
```python
delete_namespace(self, namespace_name: str) -> None
```

[L] Add interface to namespace.
```python
add_to_namespace(self, namespace_name: str, interface_name: str) -> None
```

[L] Add virtual link (interface/device with type - e.g. bridge, vlan)
```python
add_virtual_link(self, device_name: str, device_type: str, namespace: Optional[str] = None) -> None
```

[L] Delete virtual link (interface/device)
```python
delete_virtual_link(self, device_name: str, namespace: Optional[str] = None) -> None
```

[L] Add Virtual Ethernet interface
```python
create_veth_interface(self, interface_name: str, peer_name: str, namespace: Optional[str] = None) -> None
```

[L] Kill processes used in namespace
```python
kill_namespace_processes(self, namespace: str) -> None
```

[OS Agnostic] Remove conflicting IPs with tested interface
```python
remove_conflicting_ip(tested_interface: "NetworkInterface", all_interfaces: list["NetworkInterface"] | None = None) -> None:
```

[OS Agnostic] Remove duplicated IPs in system
```python
remove_duplicate_ip(ip_to_compare: IPv4Interface | IPv6Interface, interface_to_skip: "NetworkInterface | None" = None, all_interfaces: list["NetworkInterface"] | None = None) -> None
```

[L] Get output from ip link show bridge
```python
get_ip_link_show_bridge_output(self) -> str:
```

[L] Get list of net namespaces
```python
get_namespaces(self) -> list[str]:
```

[L] Delete all net namespaces
```python
delete_all_namespaces(self) -> None:
```

[L] Rename an interface
```python
rename_interface(self, current_name: str, new_name: str, namespace: str | None = None) -> None:
```
___
### Route
[Linux] Add ip route
```python
add_route(self, ip_network: "IPv4Interface", device: str, namespace: Optional[str] = None)
```

[Linux] Add ip route via remote address
```python
add_route_via_remote(
        self,
        ip_network: "IPv4Interface",
        remote_ip: "IPv4Address",
        device: str,
        set_onlink: bool = False,
        namespace: Optional[str] = None,
    ):
```
[Linux] Add default ip route
```python
add_default_route(self, remote_ip: "IPv4Address", device: str, namespace: Optional[str] = None) -> None:
```

[Linux] Change ip route
```python
change_route(
        self, ip_network: "IPv4Interface", remote_ip: "IPv4Address", device: str, namespace: Optional[str] = None
    ) -> None:
```

[Linux] Delete ip route
```python
delete_route(self, ip_network: "IPv4Interface", device: str, namespace: Optional[str] = None) -> None:
 ```

[Linux] Clear routing table for interface
```python
clear_routing_table(self, device: str, namespace: str | None = None) -> None:
```

### Network Manager - nmcli

[Linux] Set managed state
```python
set_managed(self, device: str, state: State) -> None:
```

[Linux] Remove device - set managed state to `no`
```python
remove_device(self, device: str) -> None:
```

[Linux] Get managed state
```python
get_managed_state(self, device: str) -> State:
```

[Linux] Verify managed state
```python
verify_managed(self, device: str, expected_state: State) -> bool:
```

[Linux] Prepare configuration file of interface for network manager
```python
prepare_adapter_config_file_for_network_manager(self, interface_name: str) -> None:
```
___
### Virtualization

[Linux] Create mediated device
```python
create_mdev(self, mdev_uuid: Union[str, "UUID"], pci_address: "PCIAddress", driver_name: str) -> None:
```

[Linux] Remove mediated device
```python
remove_mdev(self, mdev_uuid: Union[str, "UUID"]) -> None:
```

[Linux] Enable mediated device
```python
enable_mdev(self, mdev_uuid: Union[str, "UUID"]) -> None:
```

[Linux] Disable mediated device
```python
disable_mdev(self, mdev_uuid: Union[str, "UUID"]) -> None:
```

[Linux] Get list of all mdevs
```python
get_all_mdev_uuids(self) -> list[str]:
```

[Linux] Get PCI address of the PF that mdev is created on
```python
get_pci_address_of_mdev_pf(self, mdev_uuid: Union[str, "UUID"]) -> PCIAddress:
```

[Linux] Assign queue pairs to the mdev
```python
assign_queue_pairs(self, mdev_uuid: Union[str, "UUID"], queue_pairs: dict[str, int]) -> None:
```

[Linux] Set VMDQ (Virtual Machine Device Queues) parameter for driver.
```python
set_vmdq(driver_name: str, value: int, reload_time: float = 5) -> None
```

[ESXi] Set VMDQ (Virtual Machine Device Queues) parameter for all interfaces sharing <driver_name>.
```python
set_vmdq(driver_name: str, value: int, reload_time: float = 5) -> None
```

[ESXi] Set VMDQ (Virtual Machine Device Queues) parameter on provided interface only.
```python
set_vmdq_on_interface(*, interface: "ESXiNetworkInterface", value: int, reload_time: float = 10) -> None
```

[ESXi] Set NumQPsPerVF parameter for all interfaces sharing <driver_name>.
```python
set_num_queue_pairs_per_vf(*, driver_name: str, value: int, reload_time: float = 10) -> None
```

[ESXi]  Verify whether VMDQ is set as expected.
```python
verify_vmdq(interface: "NetworkInterface", desired_value: int) -> None
```

[ESXi]  Get the list of VFs (IDs) of specified physical device used by VM.
```python
get_vm_vf_ids(self, vm_name: str, interface: "ESXiNetworkInterface") -> list[int]
```

### Queue

[L] Get number of queues from proc interrupts
```python
get_queue_number_from_proc_interrupts(self, interface_name: str) -> str:
```

### Utils

[OS Agnostic] Check if port is used.

```python
is_port_used(self, port_num: int) -> bool
```

[L] Get bridge interfaces
```python
get_bridge_interfaces(self, all_interfaces: list["NetworkInterface"] | None = None) -> list[NetworkInterface]:
```

[OS Agnostic] Get interfaces on the same bus as passed interface
```python
get_same_pci_bus_interfaces(self, interface: "NetworkInterface") -> list["NetworkInterface"]:
```


[L] Capture some meminfo results
```python
get_memory_values(self) -> dict[str, int]
```

[W] Get memory value based on poolmons values: available, paged and non-paged memor
```python
get_memory_values(self, poolmon_dir_path: "Path | str", *, cleanup_logs: bool = True) -> dict[str, str | int]:
```

[FreeBSD]

- Convert ConfigParser string to FreeBSD Virtual Function config format.  
example input string:  
  [VF-0]
passthrough = true  
max-vlan-allowed = 1  
max-mac-filters = 1  
allow-set-mac = true  
mac-addr = 00:00:00:00:00:00  
allow-promisc = true  
num-queues = 4  
mdd-auto-reset-vf = true  
mac-anti-spoof = false  
mirror-src-vsi = 3  
example output string:  
DEFAULT {  
}  
VF-0 { 
passthrough : true  
max-vlan-allowed : 16  
max-mac-filters : 16  
allow-set-mac : true  
mac-addr : 00:00:00:00:00:00  
allow-promisc : true  
num-queues : 4  
mdd-auto-reset-vf : true  
mac-anti-spoof : true  
mirror-src-vsi : 4  
}  

```python
convert_to_vf_config_format(config: str) -> str
```

- Update num_vfs in config string. 
```python
update_num_vfs_in_config(config: str, vfs_num: int) -> str
```

### IPTables

[L] Set snat rule in iptables
```python
set_snat_rule(self, source_interface_ip: "IPv4Address", destination_ip: "IPv4Address", new_source_ip: "IPv4Address")  -> None:
```

[L] Set dnat rule in iptables
```python
set_dnat_rule(self, original_destination_ip: "IPv4Address", new_destination_ip: "IPv4Address") -> None:
```

### DDP
[ESXi] Load DDP Package
```python
load_ddp_package(self, vmnic: str, package_name: str, force: bool = False, expect_error: bool = False) -> None:
```

[ESXi] Rollback loaded DDP package
```python
rollback_ddp_package(self, vmnic: str, force: bool = False) -> None:
```

[ESXi] List currently loaded ddp packages
```python
list_ddp_packages(self, csv_format: bool = False) -> str:
```

[ESXi] Check if DDP is loaded
```python
is_ddp_loaded(self, vmnic: str, package_name: str | None = "default package") -> bool:
```

## Link Aggregation Owner

**Methods**

[Windows]
`create_nic_team(        
        interfaces: "list[WindowsNetworkInterface] | WindowsNetworkInterface",
        team_name: str,
        *,
        teaming_mode: TeamingMode = TeamingMode.SWITCHINDEPENDENT,
        lb_algorithm: LoadBalancingAlgorithm = LoadBalancingAlgorithm.DYNAMIC,
    ) -> None`
Create NIC team.

**Parameters**:

`interfaces`: interface or list of interfaces to be added to the new NIC team
`team_name`: name of the NIC team
`teaming_mode`: team operating mode: {LACP, Static, SwitchIndependent}
`lb_algorithm`: load balancing algorithm: {Dynamic, TransportPorts, IPAddresses, MacAddresses, HyperVPort}

**Raises:**

`NICTeamFeatureProcessException`: if the return code is not expected

`wait_for_nic_team_status_up(team_name: str, *, count: int = 4, tout: int = 10) -> bool`
Wait for NIC team status change to up.

**Parameters:**

`team_name`: name of the NIC team
`count`: number of checks
`tout`: timeout in seconds between subsequent tries

**Raises:**

`NICTeamFeatureException:` if NIC team does not exist

**Returns:**

`True` if NIC team status is up, `False` otherwise


`get_nic_teams()-> dict[str, dict[str, str]]`
Get a dictionary of all existing NIC teams on host with name as a key and value is a dictionary of other field-value pairs.

**Returns:**

dictionary of existing NIC teams, e.g. {"TeamBlue": {"Name": TeamBlue, "Members": Ethernet 9 ...}}

**Raises:**

`NICTeamFeatureException:` if the return code is not expected

`remove_nic_team(team_name: str) -> None`
Remove specified NIC team from the host.

**Parameters:**

`team_name:` name of the NIC team

**Raises:**

`NICTeamFeatureException:` if the return code is not expected

**Returns:**

Nothing

`get_nic_team_interfaces(team_name: str) -> list[str]`
Get list of network interfaces which are members of the specified NIC team.

**Parameters:**

`team_name:` name of the NIC team

**Raises:**

`NICTeamFeatureException:` if the return code is not expected

**Returns:**

team members of specified NIC team, e.g. `['Ethernet 1', 'Ethernet 2']`

### MAC
MAC Feature

[Linux, Windows, FreeBSD]
- `set_mac(interface_name: str, mac: MACAddress) -> None` : Set MAC address for the interface.

[Linux]
- `delete_mac(interface_name: str, mac: MACAddress) -> None` : Delete MAC address from the interface.
- `get_default_mac(interface_name: str) -> MACAddress` : Get permanent HW MAC address of the interface.

## `NetworkInterface`

Class reflecting single Network Interface. List of supported NICs Types varies between OSes. 
To check what Interface Types are supported go to [mfd-typing.network_interface](https://github.com/intel/mfd-typing/blob/main/mfd_typing/network_interface.py#L19).
It offers plenty of features which are described in the section below. 
Every feature is kept in separate folder, e.g. `ip`, `link`, `mtu`. 
Feature code is divided into separate files - one per OS supported: `esxi.py`, `freebsd.py`, `linux.py`, `windows.py`).

#### Common fields of NetworkInterface: 

- `name` - Interface name
- `mac_address` - MAC Address
- `pci_address` - PCI Address
- `interface_type` - One of multiple available types (`InterfaceType` enum from `mfd-typing`): 
  - `GENERIC` : default
  - `ETH_CONTROLLER` : network controller listed on pci (default for network device without loaded driver)
  - `VIRTUAL_DEVICE` : interface located in path ../devices/virtual/net/ (bridge, macvlan, loopback)
  - `PF` : regular physical interface; located on PCI bus (../devices/pci0000/..) (eth)
  - `VF` : virtual inteface (SRIOV); described as 'Virtual Interface' in lspci detailed info
  - `VPORT` : IPU-specific interface which shares PCI Address with other interfaces (extra VSI Info stored in `VsiInfo`)
  - `VMNIC` : ESXi-specific interface or Windows Hyper-V interface (VNIC associated with SR-IOV interface), not used in Linux 
  - `VMBUS` : Hyper-V specific for Linux Guests (https://docs.kernel.org/virt/hyperv/vmbus.html)
  - `MANAGEMENT` : interface having IPv4 Address in range of management network (`from mfd_const.mfd_const import MANAGEMENT_NETWORK`)
  - `VLAN` : virtual device which is assigned to 802.1Q VLAN (extra VLAN details stored in `VlanInterfaceInfo`)
  - `CLUSTER_MANAGEMENT` : Windows ASHCI cluster management interface type
  - `CLUSTER_STORAGE` : Windows ASHCI storage / compute interfaces in cluster nodes, marked as `vSMB` in system
  - `BTS` : Linux: BTS shares PCI bus, device ID and index, we will mark it based on name starting with `nac`
- `installed` - Boolean flag telling us whether there is driver loaded for particular interface or not
- `branding_string` - Friendly name of network adapter
- `vlan_info` - `VlanInterfaceInfo` structure from `mfd-typing` holds details like: `vlan_id` and `parent` (name of parent interface)
- `switch_info` - `SwitchInfo` structure from `data_structures` holds details like: `switch` (MFD-Switchmanagement object) and `port` (literal representing switch port name of connected interface)
- `speed` - Python's property field that returns Speed Enum member (from mfd-const) based on PCI Device
- `family` - Python's property field that returns Family Enum member (from mfd-const) based on PCI Device

#### Additional Fields of Linux Network Interface:
- `namespace` - Linux network namespace name
- `vsi_info` - VPORT specific attribute which holds info about VSI Info (`fn_id`, `host_id`, `is_vf`, `vsi_id`, `vport_id`, `is_created`, `is_enabled`)

#### Additional Fields of Windows NetworkInterface:
- `description` - Description
- `index` - Index info
- `manufacturer` - Manufacturer name
- `net_connection_status` - Status of connection
- `pnp_device_id` - PnP Device ID
- `product_name` - Product Name
- `service_name` - Service Name
- `guid` - GUID
- `win32_speed` - Advertised speed of the interface
- `cluster_info` - Structure of Cluster Info (holds information about `node` & `network`), e.g.: 
  ```python
  from mfd_typing.network_interface import ClusterInfo
  
  ClusterInfo(node="NODE-1", network="Cluster Network 2")
  ```

#### What type of interfaces are intentionally skipped? (Linux)
- :x: Loopback interface
- :x: 40G FCoE
- :x: Tunnel interface

#### Unsupported interfaces (Linux):
- :question: USB ??? -> they might be listed on sys/class/net but is not supported

Network Interface object shall remain stateless.
It means that any action which significantly affect the lifecycle of Interface object like:
- driver reload 
- host reboot
- vlan created/deleted
- namespace created/deleted
- VFs created/deleted

should trigger the re-creation process.

> :warning: It is forbidden to modify its core properties like name, pci_address (all stored as InterfaceInfo dataclass).

#### What kind of actions should force users to re-create list of interfaces?
- adding/removing interface to/from namespace
- adding/removing interface to/from vlan
- loading/unloading driver
- binding/unbinding driver
- creating/destroying virtual interfaces
- attaching/deattaching interfaces to/from VM
- flashing MAC Address (adding alternate MAC Address)
- renaming interface


#### Class diagram:

```mermaid
classDiagram

NetworkInterface <|-- NetworkInterfaceBase
NetworkInterface <|-- ESXiNetworkInterface
NetworkInterfaceBase <|-- FreeBSDNetworkInterface
NetworkInterfaceBase <|-- LinuxNetworkInterface
NetworkInterfaceBase <|-- WindowsNetworkInterface

```

* source code:

```
class NetworkInterface(ABC):
    def __init__(
      self,
      *,
      connection: "Connection",
      interface_info: "InterfaceInfo",
      topology: "NetworkInterfaceModelBase | None" = None,
    )
 ```

#### Methods
- `get_branding_string() -> str` - Get branding string. Raise `BrandingStringException` if branding string not found.

- `get_stats() -> Dict` - Get specific Network Interface statistic or get all the statistics.

- `get_mac_address()` - Get MAC Address of interface

- `get_ring_settings() -> "RingBufferSettings"` - Get ring buffer settings.

- `set_ring_settings(settings: "RingBuffer") -> None` - Set ring buffer settings.

- `get_firmware_version() -> str` - Get firmware version of Adapter.

- `get_driver_info() -> 'DriverInfo'` - Get information about driver name and driver version of Network Adapter. Raises `AdapterDriverInfoNotFound` if driver info is incomplete or absent.

- `get_number_of_ports() -> int'` - Get number of ports in tested adapter.

- `restart() -> None` - Restart interface.

#### Additional methods - Linux

- `get_device_string() -> str` - Get device string. Raise `DeviceStringException` if device string not found. Linux's implementation only.

- `get_rdma_device_name()` - Get RDMA device name for Network Interface, requires `ibv_devices` tool available on machine.

- `get_network_queues() -> Dict` - Get network queues. Raise `NetworkQueuesException` if failed. Linux's implementation only.

- `set_network_queues(rx: int, tx: int, optional: int, combined: int) -> None` - Set network queues. Raise `NetworkQueuesException` if no values provided or failed. Linux's implementation only.

- `get_numa_node() -> int` - Get the Non-Uniform Memory Architecture (NUMA) node of interface Raise `NumaNodeException` if failed.

#### Additional methods - ESXi

- `update_name_mac_branding_string()` - Update Name, MAC Address & Branding string of the interface.
- `set_hw_capabilities(capability: str, capability_value: int) -> None` - Set HW capabilities.
- `get_hw_capability(capability: str) -> int` - Get HW capabilities.

### Additional methods - IPU
IPUInterface is a mixin class used for gathering extra details about IPU interfaces.
#### Class diagram

```mermaid
classDiagram
IPUInterface <|-- IPULinuxNetworkAdapterOwner
NetworkAdapterOwner <|-- LinuxNetworkAdapterOwner
LinuxNetworkAdapterOwner
LinuxNetworkAdapterOwner <|-- IPULinuxNetworkAdapterOwner
```
- `_update_vsi_info`: update VSI Info for all VPORT and VF interfaces.


#### Features
Network interface features are implemented in directory `mfd_network_adapter/network_interface/feature`.
Every feature has implementation for multiple operating systems. **Note that scope of provided methods in features may vary between OSes.**

Feature objects are created automatically in `NetworkInterface`. You can access them via instance variable, e.g.:
```python
linux_network_interface = LinuxNetworkInterface(connection=RPyCConnection("1.1.1.1"), pci_address=PCIAddress(0000, 18, 00, 1))
linux_network_interface.ip.get_ip()
```
#### Link

`set_link(self, state: LinkState) -> None` - Set link up or down for network port.

`get_link(self) -> LinkState` - Get link status for network port.

`wait_for_link(self, state: LinkState = LinkState.UP, retries: int = 3, interval: int = 5) -> bool` - Wait for link to be in desired state.

`get_speed_duplex(self) -> (str, str)` - Get speed and duplex.

`set_speed_duplex(self, speed: Speed, duplex: DuplexType, autoneg: AutoNeg = AutoNeg.NONE) -> None` - Set speed, duplex and autonegotation.

- `speed`:
  - `Speed` Enum member from `mfd_network_adapter/network_interface/feature/link/data_structures` e.g. `Speed.G40`, `Speed.G100`
- `duplex`:
  - `DuplexType` Enum member from `mfd_network_adapter/network_interface/feature/link/data_structures` e.g. `DuplexType.AUTO`, `DuplexType.FULL`, `DuplexType.HALF`
- `autoneg`:
  - `AutoNeg` Enum member from `mfd_network_adapter/network_interface/feature/link/data_structures` e.g. `AutoNeg.ON`, `AutoNeg.OFF`, `AutoNeg.NONE`

```python
interface = WindowsNetworkInterface(connection=RPyCConnection("1.1.1.1"), interface_info=WindowsInterfaceInfo(name="SLOT 4 Port 2"))
interface.link.set_speed_duplex(Speed.G25, DuplexType.FULL)
```

`get_available_speed(self) -> List[str]` - Get all available speeds of the interface.

`get_link_speed(self) -> str` - Get link speed.

[Linux]

`get_link_speed(self) -> Union[str, None]` - Get link speed.

`get_index(self) -> int` - Parse ip link command output to receive index.

`link_off_xdp(self)` - Link off XDP application from device.

`reset_interface()` - Reset interface via PCI device 

[Windows]

`get_speed_duplex(self) -> Dict[str, Union[Speed, DuplexType]]` - Get speed and duplex.

`set_speed_duplex(self, speed: Speed, duplex: DuplexType, autoneg: AutoNeg = AutoNeg.NONE) -> None` - Set speed, duplex and autonegotation.

`get_available_speed(self) -> List[str]` - Get all available speeds of the interface.


[FreeBSD]

`get_speed_duplex(self) -> Dict[str, Union[Speed, DuplexType]]` - Get speed and duplex.

[ESXi]

`reset_interface()` - Reset interface
`set_speed_duplex(self, speed: Optional[Speed], duplex: Optional[DuplexType], autoneg: bool = True) -> None` - Set speed, duplex and auto-negotiation.
`get_supported_speeds_duplexes(self) -> List[SpeedDuplex]` - Set supported list of speed and duplex
`set_administrative_privileges(self, state: State) -> None` - Set administrative link privileges.
`get_administrative_privileges(self) -> State` - Get administrative link privileges.
`get_fec(self) -> FECModes` - Get FEC setting values.
`set_fec(self, fec_setting: FECMode) -> None` - Set FEC value. 

[ESXi, Windows, Linux]

`is_auto_negotiation(self) -> bool` - Check if interface is in auto-negotiation mode

#### Link's data structures:
```python
class LinkState(Enum):
    """Enum for Link states."""

    UP = auto()
    DOWN = auto()

class Speed(Enum):
    """Enum class for Speeds."""

    AUTO = "auto"
    M10 = "10 mbps"
    M100 = "100 mbps"
    G1 = "1.0 gbps"
    G2_5 = "2.5 gbps"
    G5 = "5 gbps"
    G10 = "10 gbps"
    G20 = "20 gbps"
    G25 = "25 gbps"
    G40 = "40 gbps"
    G50 = "50 gbps"
    G56 = "56 gbps"
    G100 = "100 gbps"
    G200 = "200 gbps"


LINUX_SPEEDS = {
    Speed.AUTO: "auto",
    Speed.M10: "10",
    Speed.M100: "100",
    Speed.G1: "1000",
    Speed.G2_5: "2500",
    Speed.G5: "5000",
    Speed.G10: "10000",
    Speed.G20: "20000",
    Speed.G25: "25000",
    Speed.G40: "40000",
    Speed.G50: "50000",
    Speed.G56: "56000",
    Speed.G100: "100000",
    Speed.G200: "200000",
}


@dataclass
class SpeedDuplexInfo:
    """Dataclass for Speed."""

    SPEEDDUPLEX = "*SpeedDuplex"

class DuplexType(Enum):
    """Enum class for Duplex Type."""

    AUTO = "auto"
    FULL = "full"
    HALF = "half"

class AutoNeg(Enum):
    """Enum class for Duplex Type."""

    NONE = "None"
    ON = "on"
    OFF = "off"

class FECMode(Enum):
    """Enum class for FEC modes."""

    NO_FEC = "No-FEC"
    RS_FEC = "RS-FEC"
    AUTO_FEC = "Auto-FEC"
    FC_FEC_BASE_R = "FC-FEC/BASE-R"

FECModes = namedtuple("FECModes", "requested_fec_mode, fec_mode")
```
#### IP

* `[OS currently supported] short description`
  
    `method declaration`

* `W - Windows, L - Linux, E - ESXI, F - FreeBSD, ALL - not os related method`

[W, L, F] Get IPs from the interface.
```python
get_ips(self) -> "IPs"
```

[W, L, F] Add IP to interface.
```python
add_ip(self, ip: Union[IPv4Interface, IPv6Interface]) -> None
```

[W, L, F] Del IP from interface.
```python
del_ip(self, ip: Union[IPv4Interface, IPv6Interface]) -> None
```

[ALL] Del all IPs from interface.
```python
del_all_ips(self) -> None
```

[W] Set DNS for interface.
```python
configure_dns(self) -> None
```

[W, L, F] Enable DHCP.
```python
enable_dynamic_ip(self, ip_version: IPVersion, ip6_autoconfig: bool = True) -> None
```

[W, L] Remove DHCP address.
```python
release_ip(self, ip_version: IPVersion) -> None
```

[W, L, F] Set ipv6 autoconfiguration.
```python
set_ipv6_autoconf(self, state: State = State.ENABLED) -> None
```

[ALL] Wait for IP.
```python
wait_for_ip(self, ip: Union[IPv4Interface, IPv6Interface], timeout: int = 30) -> None
```

[ALL] Release existing IP addresses and set new one.
```python
set_new_ip_address(self, ip: Union[IPv4Interface, IPv6Interface]) -> None
```

[W, L] Refresh Ip address.
```python
renew_ip(self) -> None
```

[W, L] Get the type of IPv6 dynamic IP.
```python
get_dynamic_ip6(self) -> "DynamicIPType"
```

[W, L] Remove IPsec rules from firewall (Windows) or ip-xfrm (Linux).
```
remove_ip_sec_rules(self, rule_name: str = "*") -> None
```

[W, L] Add IPsec rules for given IP addresses.
```python
add_ip_sec_rules(
        self,
        local_ip: Union[IPv4Interface, IPv6Interface],
        remote_ip: Union[IPv4Interface, IPv6Interface],
        rule_name_spi: str = "",
        reqid: str = "10",
        config: Optional[str] = None,
    ) -> None
```

[W] Set state of given IPsec rule. It can be only one enabled.
```python
set_ip_sec_rule_state(self, rule_name: str = "", state: State = State.DISABLED) -> None
```

[W] Get IPsec rule state setting from firewall.
```python
get_ip_sec_rule_state(self, rule_name: str = "ESP_GCM") -> State
```

[W, L, F] Check whether a tentative IP address is present on the adapter.
```python
has_tentative_address(self) -> bool
```

[W, L, F] Wait till the given address will exit tentative state.
```python
wait_till_tentative_exit(self, ip: Union[IPv4Interface, IPv6Interface], timeout: int = 15) -> None
```

[L, F] Get ipv6 autoconfiguration state.
```python
get_ipv6_autoconf(self) -> State
```

[F] Add ip to a vlan.
```python
add_vlan_ip(self, vlan_ip: str, vlan_id: int, mask: int) -> None
```

[L] Add ip neighbor.
```python
add_ip_neighbor(self, neighbor_ip: Union[IPv4Interface, IPv6Interface], neighbor_mac: MACAddress) -> "ConnectionCompletedProcess"
```

[L] Delete ip neighbor.
```python
del_ip_neighbor(self, neighbor_ip: Union[IPv4Interface, IPv6Interface], neighbor_mac: MACAddress) -> "ConnectionCompletedProcess"
```

[L] Set allmulti parameter 
```python
set_all_multicast(self, turned_on: bool = True) -> None:
```

#### IP's data structures:
```python
@dataclass
class IPs:
    """IP dataclass to keep addresses and masks."""

    v4: List["IPv4Interface"] = field(default_factory=list)
    v6: List["IPv6Interface"] = field(default_factory=list)


class IPVersion(Enum):
    """IP Version Enum."""

    V4 = "4"
    V6 = "6"


class DynamicIPType(Enum):
    """Dynamic IP type."""

    OFF = auto()
    DHCP = auto()
    AUTOCONF = auto()

```
#### IP's required tools:

* FreeBSD: ifconfig

#### MTU
**OS supported:**
 - ESXi
 - FreeBSD
 - Linux
 - Windows

`is_mtu_set(self, mtu: int) -> bool` - Check if MTU set on interface same as passed to method.

`convert_str_mtu_to_int(self, mtu: str) -> int` - Converts string ("4k", "9k", "default" or custom) to MTU integer value.

`get_mtu(self) -> int` - Get current MTU value for interface.

`set_mtu(self, mtu: int) -> None` - Set MTU value on interface.

You can set MTU_CUSTOM value to whatever value it is needed.
If custom value is read from interface, it will automatically set MtuSize.MTU_CUSTOM to this value.

#### MTU's data structures:
```python
@dataclass
class MtuSize:
    """Dataclass for MTU sizes."""

    MTU_CUSTOM: int = 0
    MTU_DEFAULT: int = 1500
    MTU_4K: int = 4074
    MTU_9K: int = 9000
    MTU_MIN_IP4: int = 576
    MTU_MIN_IP6: int = 1280
    MTU_MAX: int = MTU_9K

```

#### Driver

[ALL] `get_driver_info(self) -> DriverInfo` - Get information about driver name and version.

[ALL] `get_module_dir(self) -> str` - Get the folder in the driver disk that contains the driver

[ALL] `is_interface_affected_by_driver_reload(self) -> bool` - Check if driver will be affected by reloading driver

[Linux] `get_formatted_driver_version() -> Dict` - Get current driver version and normalize the output into a dictionary.

#### Utils

Miscellaneous functionalities of network interface that don't have its place yet.

[ALL] Check if speed is equal to requested one.
```python
is_speed_eq(self, speed: Speed) -> bool
```

[ALL] Check if speed is equal or higher than requested one.
```python
is_speed_eq_or_higher(self, speed: Speed) -> bool
```

[ALL] Check if family is equal to requested one.
```python
is_family_eq(self, family: Family) -> bool
```

[ESXi] Get value of param from generic configuration of a network device.
```python
get_param(self, param: str) -> str
```

[ESXi] Set the debug level for the network device.
```python
set_debug_level(self, lvl: int = 0) -> None
```

[Windows] Get interface index from Powershell NetAdapter command.
```python
get_interface_index(self) -> str:
```

[Linux] Add tunnel endpoint.
```python
add_tunnel_endpoint(
        self,
        tun_name: str,
        tun_type: TunnelType,
        remote: "IPv4Address | None" = None,
        vni: int | None = None,
        group: "IPv4Address | None" = None,
        dst_port: int | None = None,
        ttl: int | None = None,
        interface_name: str | None = None,
        local_ip: "IPv4Address | None" = None,
    ) -> None:
```
[Linux] Query the specified network interface for coalescing information.
```python
get_coalescing_information(self) -> type[dataclass]
```

[Linux] Change the coalescing settings of the specified network device.
```python
set_coalescing_information(
    option: str, value: str, expected_return_codes=frozenset({0, ETHTOOL_RC_VALUE_UNCHANGED})
) -> str:
```

[Linux]  Change eeprom options.
```python
change_eeprom(self, option: EepromOption, value: str) -> str:
```

Available EepromOption:
```python
class EepromOption(Enum):
    """EEPROM option for the interface."""

    MAGIC = "magic"
    OFFSET = "offset"
    LENGTH = "length"
    VALUE = "value"
```

[Linux] Blink LEDs. Get output from --identify
```python
blink(duration: int = 3) -> str:
```
___
### Memory

[Linux] Get memory leak value based on poolmons diff value.
```python
get_memory_values(self, poolmon_dir_path: "Path | str", cleanup_logs: bool = True) -> dict[str, str | int]:
```
##### NetAdapterAdvanceProperties [W]

`get_advanced_properties() -> List[Dict]` -  Get interface advanced properties..

`get_advanced_property(advanced_property: str, use_registry: bool) -> str` -  Get specified advanced property from interface

`get_advanced_property_valid_values(registry_keyword: str) -> List` - Get interface advanced property valid values.

`set_advanced_property(registry_keyword: str, registry_value: Union[str, int]) -> None` - Set interface advanced property accessed by registry_keyword.

`reset_advanced_properties() -> None` - Reset all the interface advanced properties to default values.


#### NUMA
[Windows]
```python
set_numa_node_id(
    self, node_id: str
) -> None
```
#### InterFrame

[Windows]

`set_adaptive_ifs(self, enabled: State) -> None` - Set configuration of inter-frame spacing: enabled/disabled.

`get_adaptive_ifs(self) -> str` - Read setting of inter-frame spacing.

#### RSS

[Windows, Linux, FreeBSD] Set Queues

```python
set_queues(self, queue_number: int) -> None - Set Queues
```

[Windows, FreeBSD] Enable/Disable RSS

```python
set_rss(self, enabled: State) -> None - Enable/Disable RSS
```

[Windows] Set Max Processors

```python
set_max_processors(self, max_proc: int) -> None - Set MAX processors usage.
```

[Windows] Set Base Processors

```python
set_base_processor_num(self, base_proc_num: int) -> None - Set base processors.
```

[Windows] Set Max queues per VPORT

```python
set_max_queues_per_vport(self, max_queues_vport: int) -> None - Set MAX queues per virtual port usage.
```

[Windows, Linux, FreeBSD] Get Queues Information

```python
get_queues(self) -> str - Get Queues Information, amount of queues.
```

[Windows] Get Max processors

```python
get_max_processors(self) -> str - Get Max processors usage.
```

[Windows] Get base processors

```python
get_base_processor_num(self) -> str - Get base processors number.
```

[Windows] Get profile value

```python
get_profile(self) -> str - Get profile value on the interface.
```

[Windows, Linux, FreeBSD] Get Channels Information

```python
get_max_channels(self) -> str - Get Channels Information.
```

[Windows] Get Adapter Information

```python
get_adapter_info(self) -> Dict[str, str] - Get Adapter Information on the interface.
```

[Windows] Get processors information.

```python
get_proc_info(self) -> Dict[str, str] - Get processors information.
```

[Windows] Get maximum of available processors

```python
get_max_available_processors(self) -> int - Get maximum of available processors that can be assigned
```

[Windows] Get processors on Indirection table

```python
get_indirection_table_processor_numbers(self) -> List[None | str] - Get processors on Indirection table.
```

[Windows] Get Processor numbers of given numa distance

```python
get_numa_processor_array(self, numa_distance: int = 0) -> List[None | str] - Get Processor numbers of given numa distance
```

[Windows] Set profile

```python
set_profile(self, rss_profile: RSSProfileInfo) -> None - Set profile.
```

- `rss_profile`:
  - `RSSProfileInfo` Enum member from `mfd_network_adapter/network_interface/feature/rss/data_structures` e.g. `RSSProfileInfo.CLOSESTPROCESSOR`, `RSSProfileInfo.NUMASCALING`


```python
interface = WindowsNetworkInterface(connection=RPyCConnection("1.1.1.1"), interface_info=WindowsInterfaceInfo(name="SLOT 4 Port 2"))
interface.rss.set_profile(RSSProfileInfo.CLOSESTPROCESSOR)
```

[Windows] Set profile via AdapterRss command

```python
set_profile_command(self, rss_profile: RSSProfileInfo) -> None - Set profile via AdapterRss command.
```

- `rss_profile`:
  - `RSSProfileInfo` Enum member from `mfd_network_adapter/network_interface/feature/rss/data_structures` e.g. `RSSProfileInfo.CLOSESTPROCESSOR`, `RSSProfileInfo.NUMASCALING`


```python
interface = WindowsNetworkInterface(connection=RPyCConnection("1.1.1.1"), interface_info=WindowsInterfaceInfo(name="SLOT 4 Port 2"))
interface.rss.set_profile_command(RSSProfileInfo.CLOSESTPROCESSOR)
```

[Windows] Set Preferred NUMA Node Id via RSS.

```python
set_numa_node_id(self, node_id: int) -> None - Set Preferred NUMA Node Id via RSS
```

[Windows] Enable via AdapterRss

```python
enable(self) -> None - To enable via AdapterRss
```

[Windows] Disable via AdapterRss

```python
disable(self) -> None - To disable via AdapterRss
```

[Windows, Linux] Get State Information

```python
get_state(self) -> State - Get State Information
```

[Windows, Linux, FreeBSD] Get Maximum number of queues

```python
get_max_queues(self) -> int - Get Maximum number of queues
```

[Linux] Get actual number of queues

```python
get_actual_queues(self) -> int - Get actual number of queues.
```

[Linux] Get the indirection table count

```python
get_indirection_count(self) -> int - Get the indirection table count.
```

[Linux] Get Hash Options.

```python
get_hash_options(self, flow_type: FlowType) -> List[Optional[str]] - Get Hash Options.
```

[Linux] Get number of individual Tx and Rx queues

```python
get_rx_tx_queues(self, is_10g_adapter: bool) -> List[int] - Get number of individual Tx and Rx queues.
```

[Linux] Set RX, TX queues individual.

```python
set_queues_individual(self, tx_queues: str = "", rx_queues: str = "", is_10g_adapter: bool = False, is_100g_adapter: bool = False) -> None - Set RX, TX queues individual.
```

[Linux, FreeBSD] Add queues statistics

```python
add_queues_statistics(self, queue_number: int) -> None - Add queues statistics.
```

```python
interface = FreeBSDNetworkInterface(connection=connection, interface_info=LinuxInterfaceInfo(name="ixl1"))
interface.rss.add_queues_statistics(queue_number=10)
```

```python
interface = LinuxNetworkInterface(connection=connection, interface_info=LinuxInterfaceInfo(name="enp59s0f1"))
interface.rss.add_queues_statistics(queue_number=10)
```

[Linux, FreeBSD] Validate Statistics

```python
validate_statistics(self, traffic_duration: int = 30) -> None - Validate Statistics.
```

```python
interface = FreeBSDNetworkInterface(connection=connection, interface_info=LinuxInterfaceInfo(name="ixl1"))
interface.rss.validate_statistics(traffic_duration=45)
```

```python
interface = LinuxNetworkInterface(connection=connection, interface_info=LinuxInterfaceInfo(name="enp59s0f1"))
interface.rss.validate_statistics(traffic_duration=45)
```

[Windows] Validate Statistics

```python
validate_statistics(self, is_10g_adapter: bool = False, traffic_duration: int = 30) -> None - Validate statistics.
```

```python
interface = WindowsNetworkInterface(connection=RPyCConnection("1.1.1.1"), interface_info=WindowsInterfaceInfo(name="SLOT 4 Port 2"))
interface.rss.validate_statistics(is_10g_adapter=True, traffic_duration=15)
```

[Windows] To get max number of queues used

```python
get_num_queues_used(self, traffic_duration: int = 30) -> int - To get max number of queues used.
```

```python
interface = WindowsNetworkInterface(connection=RPyCConnection("1.1.1.1"), interface_info=WindowsInterfaceInfo(name="SLOT 4 Port 2"))
interface.rss.get_num_queues_used(traffic_duration=15)
```

[Windows] To get CPU IDs

```python
get_cpu_ids(self, traffic_duration: int = 30) -> List[str] - To get CPU IDs.
```

```python
interface = WindowsNetworkInterface(connection=RPyCConnection("1.1.1.1"), interface_info=WindowsInterfaceInfo(name="SLOT 4 Port 2"))
interface.rss.get_cpu_ids(traffic_duration=15)
```

[ESXi] Get privstats for Pkts rx queues using localcli command.

```python
get_rx_pkts_stats(self) -> dict[str]
```

[ESXi] Get info for RSS/DRSS modules in icen driver by using intnet tool.

```python
get_rss_info_intnet(self) -> dict[str]
```

[ESXi] get_queues_for_rss_engine.

```python
get_queues_for_rss_engine(self) -> dict[str, list[str]]
```

[ESXi] Get all DefQ or NetQ RSS queues.

```python
get_netq_defq_rss_queues(self, netq_rss: bool) -> list
```

#### Stats


[Linux] 

- `get_system_stats(name: Optional[str]) -> Dict` - Get a specific or all statistics from a specific network interface using system method.

- `get_stats_and_sys_stats(name: Optional[str]) -> Dict` - Get all or a specific statistics from specific interface using system and ethtool method.

- `read_and_sum_stats(name: str) -> int` - Get sum for similar statistics.

- `get_system_stats_errors() -> Dict` - Aggregate system error statistics from system statistics path.

- `get_per_queue_stat_string(direction: str, stat: str) -> str` - Get the properly formatted per-queue statistics string for the adapter.

- `generate_default_stat_checker() -> StatChecker` - Generate StatChecker class with standard statistics to verify.

- `start_statistics(names: list, stat_trend: list, stat_threshold: list) -> None` - Start to gather statistics on adapter, before starting traffic.

- `verify_statistics(stat_checker: StatChecker) -> bool` - Compare the statistics to when they were captured before and report any errors.

- `add_cso_statistics(rx_enabled: bool, tx_enabled: bool, proto: Protocol, ip_ver: str, direction: Direction, min_stats: int, max_err: int) -> None:` - Adding additional statistics to the interface statchecker object.

[Windows]

> [!IMPORTANT]  
>  This feature is under development. All source code and features on the main branch are for the purpose of testing or evaluation and not production ready. Method requires DLLs in `c:\NET_ADAPTER` directory to read OIDs.

- `get_stats(names: Optional[str] = None) -> Dict` - Get a specific or all statistics from a specific network interface.

- `add_default_stats() -> None` - Adding default statistics to the interface stat_checker object.

- `add_cso_statistics(rx_enabled: bool, tx_enabled: bool, proto: Protocol, ip_ver: str, direction: Direction, min_stats: int, max_err: int) -> None:` - Adding additional statistics to the interface statchecker object.

- `check_statistics_errors() -> bool` - Compare the statistics on the interface statschecker obj captured before and report any errors.

[FreeBSD]

- `get_stats(self, name: Optional[str] = None) -> Dict[str, str]: - Get a specific or all statistics from a specific network interface.

[ESXi]

- `get_stats(self, name: Optional[str] = None) -> Dict[str, str]: - Get a specific or all statistics from a specific network interface.
- `verify_stats(self, stats: Optional[Dict] = None) -> bool` - Check correctness of stats (if no errors or drops)
#### Stats data structures:
```python
class Protocol(Enum):
    """Enum class for Protocol Type."""

    IP = "ip"
    TCP = "tcp"
    UDP = "udp"
    SCTP = "sctp"


class Direction(Enum):
    """Enum class for Direction Type."""

    TX = auto()
    RX = auto()


```

#### LLDP

[Windows]

```python
set_fwlldp(self, enabled: State) -> None: - Set fw-lldp feature enabled/disabled.
```

[Linux]

```python
set_fwlldp(self, enabled: State) -> None: - Set fw-lldp feature enabled/disabled.
is_fwlldp_enabled(self) -> bool: - Check fw-lldp is enabled or not
```

[FreeBsd]

```python
set_fwlldp(self, enabled: State) -> str: - Set fw-lldp feature enabled/disabled
get_fwlldp(self) -> bool: - Get fw-lldp status
```

#### Wol

[Windows]

```python
set_wol_option(self, state: State) -> None: - Set Wake on LAN option.
get_wol_option(self) -> str: - Get wake on LAN option.
```

[Linux]

```python
get_supported_wol_options(self) -> List[WolOptions]: - Get supported wake on lan options.
get_wol_options(self) -> List[WolOptions]: - Get wake on lan options
set_wol_options(self, options: List[WolOptions]) -> None: - set wake on lan options
set_wake_from_magicpacket(self, state: State) -> None: - Toggling of wake from magicpacket
send_magic_packet(self, host_mac_address: MACAddress, broadcast: State = State.DISABLED, password: str = None) -> None: - Send magic packet to wake the system
```

#### Interrupt

[ESXi]

```python
get_interrupt_moderation_rate(self, is_ens: bool = False, pci_address: Optional[PCIAddress] = None,) -> Tuple[int, int]: - Get current interrupt moderation rate values.
get_available_interrupt_moderation_parameters() -> InterruptModeration: - Get available interrupt moderation parameters.
```

[Windows]
```python
set_interrupt_moderation(self, enabled: State) -> None: - Set Interrupt Moderation.
get_interrupt_moderation(self) -> str: - Get Interrupt Moderation value.
get_num_interrupt_vectors(self) -> int: - Get number of interrupt vectors per adapter.
get_interrupt_moderation_rate(self) -> str: -> Get Interrupt Moderation Rate.
set_interrupt_moderation_rate(self, setting: InterruptModerationRate) -> None: Set Interrupt Moderation Rate.
set_adaptive_interrupt_mode(self, mode: State) -> None: -> Set Adaptive Interrupt Mode
get_interrupt_mode(self) -> Tuple[InterruptMode, int | None]: -> Get interrupt mode
get_expected_max_interrupts(self, itr_val: ITRValues, virtual: bool = False) -> int: -> Get expected max interrupts.
check_itr_value_set(self, expected_value: int) -> bool: -> Check ITR value.
get_per_queue_interrupts_per_sec(self, interval: int = 5, samples: int = 5) -> dict[str, int]: -> Get per queue interrupts per second data.
set_interrupt_mode(self, mode: InterruptMode, interrupt_limit: int | None = None) -> None: - Set interrupt mode for the interface.
get_rsc_operational_enabled(self, ip_flag: IPFlag, status_to_query: StatusToQuery) -> bool: - Get the RSC/LRO operational state or enabled status through Get-NetAdapterRsc cmdlet.
```

[Linux]
```python
get_interrupt_mode(self) -> InterruptMode: -> Get interrupt mode
is_interrupt_mode_msix(self) -> State: -> Check interrupt mode is msix
check_interrupt_throttle_rate(self, itr_threshold: int, duration: int = 10) -> bool: - Check interrupt throttle rate from /proc/interrupts.
set_adaptive_interrupt_mode(self, mode: State) -> None: - Set adaptive interrupt mode.
get_interrupt_moderation_rate(self) -> str: - Get interrupt moderation rate (rx-usecs) value.
get_per_queue_interrupts_per_sec(self, interval: int = 5) -> dict[str, int]: -> Get the interface per queue interrupts per second data.
get_per_queue_interrupts_delta(self, interval: int = 5) -> InterruptsData: -> Get the interface per queue interrupts delta.
get_expected_max_interrupts(self, itr_val: ITRValues) -> int: - Get expected max interrupts.
set_interrupt_moderation_rate(self, rxvalue: str, txvalue: str | None = None) -> None: -> Set Interrupt Moderation rate.
```

[FreeBsd]
```python
get_interrupts_info_per_que(self) -> list[dict[str]]: -> Get interrupt information
get_interrupts_per_second(self, interval:int=10) -> int: -> Get the IRQ per second
get_interrupts_rate_active_avg(self) -> int: -> Get an average interrupts rate on active queues since the last call.
```

#### Queue

[Linux]

`get_per_queue_packet_stats(self) -> Dict` - Get existing Tx Rx per queue packets counters.

[Windows]

`get_hw_queue_number(self) -> int` - Get the number of available hardware acceleration queues.

`set_sriov_queue_number(self, value: int) -> None` - Set number of hardware queues that need to be assigned to SRIOV adapters.

`split_hw_queues(self) -> None` - Set 1/2 of HW queues to SRIOV and 1/2 to VMQ.

`get_vmq_queue(self) -> str` - Get adapter vmq allocation using Get-NetadapterVmqQueue cmdlet.

`get_queues_in_use(self, traffic_duration: int = 5, sampling_interval: int = 1) -> int` - Get number of queues used with enabled/disabled RSS.

[ESXi]

`get_queues_info(self, queues: str) -> dict[str, str]` - Get queues information for interface. Raises QueueFeatureInvalidValueException if provided queues are incorrect.

`get_queues(self, queues: str) -> str` - Get queues information for interface as raw output. Raises QueueFeatureInvalidValueException if provided queues are incorrect.

`get_rx_sec_queues(self, primary_queue: str) -> list[str]` - Get rxSecQueues of rxqueues for primary_queue.

`read_primary_or_secondary_queues_vsish(raw_vsish_output: str) -> list[str]` - Read primary or secondary queues from vsish.

`_parse_lcores_order(output: str) -> list[str]` - Parse lcores rx/tx order. Raises NetworkAdapterModuleException on operation error.

`_parse_lcores_values(output: str, mac: "MACAddress") -> list[int]` - Parse lcores values for rx/tx. Raises NetworkAdapterModuleException on operation error.

`get_assigned_ens_lcores(self) -> type(RxTx)` - Get rx and tx lcores which are assigned to given MAC address.

`get_ens_flow_table(self, lcore: int = None) -> list[dict[str, str | Any]]` - Return ENS flow table dump. When lcore is provided return flow table only for specific lcore.

`get_ens_fpo_stats(self, lcore: int) -> dict[str, AnyStr]` - Get ENS FPO statistics. Raises NetworkAdapterModuleException on error.

#### Virtualization
Virtualization related functionalities.

[Windows, Linux]
- `set_sriov(sriov_enabled: bool, no_restart: bool = False) -> None` - Set network interface SRIOV.

[ESXi]
- `get_enabled_vfs(self, interface: str) -> int` - Get number of VFs enabled on PF.
- `get_possible_options_intnet_sriovnic_vf_set(self) -> list` - Get possible options to be used with esxcli intnet sriovnic vf set command.
- `set_intnet_sriovnic_options(self, vf_id: int, interface: str, **kwargs) -> bool` - Use intnet tool to set sriovnic options like Trusted Mode or Spoof or Floating VEB options on VF.
- `get_intnet_sriovnic_options(self, vf_id: int, interface: str) -> dict` - Get sriovnic options like Trusted, Spoof and Floating VEB status on VF.
- `set_intnet_vmdq_loopback(self, interface: str, **kwargs) -> None` - Use intnet tool to set VMDQ loopback option.
- `get_intnet_vmdq_loopback(self, interface: str) -> bool` - Get VMDQ loopback status on interface.
- `get_connected_vfs_info(self) -> list[VFInfo]` - Get list of used vfs

[Windows]
- `set_vmq(vmq_enabled: bool, no_restart: bool = False) -> None` - Set network interface VMQ.
- `is_vmq_enabled() -> bool` - Check VMQ is enabled on PF.
- `is_sriov_enabled() -> bool` - Check SRIOV is enabled on PF.
- `is_sriov_supported() -> bool` - Check SRIOV is enabled on PF.
- `enable_pf_npcap_binding() -> None` - Enable PF NPCAP after creating vSwitch.

[Linux]

:warning: All of the methods listed below can be executed only on PF Interface (`InterfaceType.PF`)

- `_raise_error_if_not_pf()` - Raise error in case current interface is not PF.
- `_get_vfs_details()` - Get VF details of PF interface.

- `set_max_tx_rate(self, vf_id: int, value: int) -> None` - Set max_tx_rate VF-d parameter status.
- `set_min_tx_rate(self, vf_id: int, value: int) -> None` - Set min_tx_rate VF-d parameter status.
- `set_trust(vf_id: int, state: State) -> None` - Set trust parameter to On/Off
- `set_spoofchk(vf_id: int, state: State) -> None` - Set spoofchk parameter to On/Off
- `get_trust(vf_id: int) -> State` - Get trust setting value for VF
- `get_spoofck(vf_id: int) -> State` - Get spoofchk setting value for VF
- `set_link_for_vf(vf_id: int, link_state: LinkState) -> None` - Set link for a VF interface.
- `set_vlan_for_vf(vf_id: int, vlan_id: int, proto: VlanProto) -> None` - Set port VLAN for a VF interface
- `set_mac_for_vf(vf_id: int, mac: MACAddress) -> None` - Set MAC address for VF interface.
- `get_max_vfs() -> int` - Get maximal number of VFs per interface based on either name or PCI Address (if name not set on the interface).
- `get_current_vfs() -> int` - Get current number of VFs per interface based on either name or PCI Address (if name not set on the interface).
- `get_designed_number_vfs() -> tuple[int, int]` - Get designed max number of VFs, total and per PF.
- `get_link_state(self, vf_id: int) -> LinkState` - Get link-state setting value for VF
- `get_mac_address(self, vf_id: int) -> MACAddress` Get mac address value for VF
#### Virtualization Data Structures:
```python
@dataclass
class VFInfo:
    """Structure for VF information."""

    vf_id: str
    pci_address: PCIAddress
    owner_world_id: str
```

#### StatChecker

StatChecker is operating on unified between different card families statistic names.
Under the hood, there are methods that convert real statistics from a system. 

e.g. `rx-0.packets` from ethtool after parsing is represented in the structure as `rx_0.packets`,
so code translates it to the common `rx_queues_0_packets`.

That's the purpose of `get_per_queue_stat_string` API in Stats feature and `_search_statistics_name` API from `StatChecker` class.

[Linux, Windows, FreeBSD]
- `add(stat_name: str, stat_trend: Trend | Value, threshold: int = 0) -> None` - Add new statistic to be handled.
- `modify(stat_name: str, stat_trend: Trend | Value, threshold: int) -> None` - Modify expected trend of value and threshold for the trend for already added statistic.
- `get_values() -> Dict[str, List[Union[int, str]]]` - Get current values for statistic defined by add() method.
- `invalid_stats_found() -> None` - Check if the target statistics are supported by the driver. Raises NotSupportedStatistic if unsupported statistic found in added statistics.
- `validate_trend() -> Optional[Dict]` - Validate gathered data.
- `get_number_of_valid_statistics() -> int` - Get difference of all parameters and parameters that were recognized as valid.
- `get_single_diff(stat_name: str, series: int) -> None` - Get difference for stat_name in desired series.
- `reset() -> None` - Reset all gathered statistics values.
- `clear_values() -> None` - Reset all gathered values. Configs are preserved.
- `get_packet_errors(error_names: Union[Tuple, List]) -> Dict` - Gather error statistics on adapter.

#### Buffers

[Linux]
- `get_rx_checksumming(self) -> Optional[State]` : Get RX Checksumming
- `set_rx_checksumming(self, value: Union[str, bool]) -> str`: Set RX Checksumming
- `get_tx_checksumming(self) -> Optional[State]`: Get TX Checksumming
- `set_tx_checksumming(self, value: Union[str, bool]) -> str`: Set TX Checksumming
- `find_buffer_sizes(self, direction: str) -> Dict[str, str]`: Find Buffer sizes
- `get_rx_buffers(self, attr: BuffersAttribute = BuffersAttribute.NONE) -> int`: Get RX Buffers
- `get_tx_buffers(self, attr: BuffersAttribute = BuffersAttribute.NONE) -> int`: Get TX Buffers
- `get_min_buffers(self) -> Optional[int]`: Get Minimum buffer size

[Windows]
- `get_rx_buffers(self, attr: BuffersAttribute = BuffersAttribute.NONE) -> int`: Get RX Buffers
- `get_tx_buffers(self, attr: BuffersAttribute = BuffersAttribute.NONE) -> int`: Get TX Buffers

[ESXi]
- `set_ring_size(self, rx_ring_size : int | None = None, tx_ring_size: int | None = None) -> None:` - Set RX/TX ring size.
- `get_ring_size(self, preset : bool = False) ->  RingSize:` - Get RX/TX ring size.

#### DMA
[Windows]
- `set_dma_coalescing(self, value: int = 0, method_registry: bool = True) -> None`: Set dma coalescing value
- `get_dma_coalescing(self, cached: bool = True) -> Optional[int]`: Get dma coalescing value from setting

#### Capture
[Tshark: Windows, Linux, FreeBSD] 
</br>
[Tcpdump: Linux, ESXI]
</br>
[PktCap: ESXI]

Tcpdump, Tshark and PktCap are [mfd_packet_capture](https://github.com/intel/mfd-packet-capture) objects.
All functions are accesible by
- `interface.capture.tshark`
- `interface.capture.tcpdump`
- `interface.capture.pktcap`

#### VLAN
[ESXi]
- `set_vlan_tpid(self, tpid: str) -> None:`: Set TPID used for VLAN tagging by VFs.

#### Offload
[Windows]
`get_offload(self, protocol: Protocol, ip_ver: IPVersion) -> str` - Get checksum offloading status.
`set_offload(self, protocol: Protocol, ip_ver: IPVersion, value: str) -> None` - Set offload value.
`get_checksum_offload_settings(self, protocol: Protocol, ip_ver: IPVersion) -> RxTxOffloadSetting` - Fetch checksum offload settings for RX and TX.
`set_checksum_offload_settings(self, rx_tx_settings: RxTxOffloadSetting, protocol: Protocol, ip_ver: IPVersion) -> None` - Set checksum offload settings.

[Linux]
`get_lso() -> OffloadSetting` - Get LSO offload settings.
`set_lso(value: OffloadSetting) -> None` - Set LSO offload settings.
`get_lro() -> OffloadSetting` - Get LRO offload settings.
`set_lro(value: OffloadSetting) -> None` - Set LRO offload settings.
`get_rx_checksumming() -> OffloadSetting` - Get RX checksum offload settings.
`set_rx_checksumming(value: OffloadSetting) -> None` - Set RX checksum offload settings.
`get_tx_checksumming() -> OffloadSetting` - Get TX checksum offload settings.
`set_tx_checksumming(value: OffloadSetting) -> None` - Set TX checksum offload settings.
`get_rx_vlan_offload() -> OffloadSetting` - Get RX VLAN offload settings.
`set_rx_vlan_offload(value: OffloadSetting) -> None` - Set RX VLAN offload settings.
`get_tx_vlan_offload() -> OffloadSetting` - Get TX VLAN offload settings.
`set_tx_vlan_offload(value: OffloadSetting) -> None` - Set TX VLAN offload settings.
`get_checksum_offload_settings() -> RxTxOffloadSetting` - Get checksum offload settings.
`set_checksum_offload_settings(rx_tx_settings: RxTxOffloadSetting) -> None` - Set checksum offload settings.

[ESXi]
`change_offload_setting(offload: str, enable: State = State.ENABLED) -> None` - Change HW offload setting on ESXi host.
`check_offload_setting(offload: str) -> bool` - Check if HW offload setting is enabled on ESXi host.
`set_hw_capabilities(offload: str, enable: State) -> None` - Set hardware capabilities for offload on PF.
`get_hw_capabilities(offload: str) -> str` - Get hardware capabilities for offload on PF. 

#### ENS
[ESXi]
`is_ens_capable(self) -> bool` - Check if ENS is capable on the interface.
`is_ens_enabled(self) -> bool` - Check if ENS is enabled on the interface.
`is_ens_unified_driver(self) -> bool` - Check if ENS is a unified driver on the interface.
`is_ens_interrupt_capable(self) -> bool` - Check if ENS is interrupt capable on the interface.
`is_ens_interrupt_enabled(self) -> bool` - Check if ENS is interrupt enabled on the interface.

#### NIC Team
[Windows]

**Methods**

  `add_interface_to_nic_team(self, team_name: str) -> str`
  
  This method adds an interface as a new member to an existing NIC team.  
  
  **Parameters:**  
  
  * team_name (str): The name of the NIC team.
  
  **Returns:**  
  
  * str: The output of the command execution.

  `add_vlan_to_nic_team(self, team_name: str, vlan_name: str, vlan_id: int)`

  This method creates and adds a team interface with a given VLAN ID to the specified NIC team.  
  
  **Parameters:**  
  
  * team_name (str): The name of the NIC team.
  * vlan_name (str): The name for the VLAN interface.
  * vlan_id (int): The VLAN ID.
    
  `set_vlan_id_on_nic_team_interface(self, vlan_id: int, team_name: str) -> str`

  This method sets a new VLAN ID on a default NIC team interface.  
  
  **Parameters:**  
  
  * vlan_id (int): The desired VLAN ID.
    * team_name (str): The name of the NIC team.
  
  **Returns:**
  
  * str: The output of the command execution.

  `remove_interface_from_nic_team(self, team_name: str) -> None`

  This method removes network interface from specified NIC team.  
  
  **Parameters:**  

  * team_name (str): The name of the NIC team.
  
  **Returns:**
  
  * Nothing

#### MAC
[Linux]
- `get_multicast_mac_number() -> int` : Get number of multicast MAC addresses.

##### Data structures

Structure used for getting and setting checksum settings
```python
@dataclass(unsafe_hash=True)
class RxTxOffloadSetting:
    """Class for string offload setting for rx and tx."""

    rx_enabled: bool
    tx_enabled: bool
```

## NetworkInterface Data structures:
```python
@dataclass
class RingBuffer:
    rx: Optional[int] = None
    rx_mini: Optional[int] = None
    rx_jumbo: Optional[int] = None
    tx: Optional[int] = None
```

```python
@dataclass
class RingBufferSettings:
    max: RingBuffer = field(default_factory=RingBuffer)
    current: RingBuffer = field(default_factory=RingBuffer)
```

```python
class VlanProto(Enum):
    Dot1q = "802.1Q"
    Dot1ad = "802.1ad"
```

```python
class LinkState(Enum):
    AUTO = "auto"
    ENABLE = "enable"
    DISABLE = "disable"
```

```python
@dataclass
class VFDetail:
    """VF Details."""

    id: int  # noqa: A003
    mac_address: "MACAddress"
    spoofchk: State
    trust: State
```
## Common Data Structures 
### (mfd_network_adapter.data_structures)

```python
class State(Enum):
    """States."""

    ENABLED = auto()
    DISABLED = auto()
```

## OS supported:
* LINUX
* WINDOWS
* ESXi
* FREEBSD

#### Flow control
[Linux]
- `get_flow_control() -> FlowControlParams` - Get flow control parameters for network interface.
- `set_flow_control(flowcontrol_params: FlowControlParams) -> None` - Set flow control parameters on network interface.
- `set_receive_flow_hash(flow_hash_params: FlowHashParams) -> str` - Configures recieve flow hash on the interface.
- `set_flow_director_atr(enabled: State) -> str` - Set flow director atr on the interface.
- `get_flow_director_atr() -> State` - Get flow director atr on the interface.

[FreeBSD]
- `set_flow_control(flowcontrol_params: FlowControlParams) -> None` - Disable/Enable flow control option.
- `get_flow_control() -> FlowControlParams` - Get flow control parameters for network interface.
- `get_flow_control_counter(flow_control_counter: FlowCtrlCounter, mac_stats_sysctl_path: str) -> int` - Get flow control counter value.

[Windows]
- `set_flow_control(self, flowcontrol_params: FlowControlParams) -> None` - Set flow control on interface.
- `get_flow_control(self) -> FlowControlParams` - Get Flow Control params.
- `get_flow_control_registry(self) -> str` - Get flow control setting from registry.
- `set_flow_control_registry(self, setting: FlowControlType) -> None` - Set flow control setting from registry.
- `set_flow_ctrl_watermark(self, watermark: Watermark, value: str) -> None` - Set flow control watermark registry entry
- `get_flow_ctrl_watermark(self, watermark: Watermark) -> str` - Get flow control watermark value
- `remove_flow_ctrl_watermark(self, watermark: Watermark) -> None` - Remove flow control watermark value
- `get_flow_ctrl_values(self) -> List` - Get all supported flow control values

[ESXi]
- `set_flow_control_settings(self, *, autoneg: bool | None = None, rx_pause: bool | None = None, tx_pause: bool | None = None, setting_timeout: int = 5) -> None:` - Set flow control settings.
- `get_flow_control_settings(self) -> PauseParams:` - Get flow control settings.

## Data structures:

```
@dataclass
class FlowControlParams:
    """Dataclass for the interface flow control parameters."""

    autonegotiate: Optional[str] = field(default="off")
    tx: Optional[str] = field(default="off")
    rx: Optional[str] = field(default="off")
    tx_negotiated: str = field(
        init=False,
        default=None,
        metadata={
            "description": "This field indicates the operational status of tx negotiated with the peer."
            + "It will be populated by the method get_flow_control"
        },
    )
    rx_negotiated: str = field(
        init=False,
        default=None,
        metadata={
            "description": "This field indicates the operational status of rx negotiated with the peer."
            + "It will be populated by the method get_flow_control"
        },
    )

@dataclass
class FlowHashParams:
    """Data class for the receive side flow hashing."""

    flow_type: str
    hash_value: Optional[str] = None

    def __post_init__(self):
        if self.hash_value is None:
            self.hash_value = self._default_hashes()

    def _default_hashes(self) -> str:
        """
        Return default hash combinations for the given traffic flow type.

        :raises FlowControlException: When the flow type is not defined in the predefined_hash_maps
        :return: str hash combination for a given flow
        """
        predefined_hash_maps = {
            "tcp4": "sdfn",
            "udp4": "sdfn",
            "ipv4": "sd",
            "tcp6": "sdfn",
            "udp6": "sdfn",
            "ipv6": "sd",
            "sctp4": "sdfn",
            "sctp6": "sdfn",
        }
        hash_value = predefined_hash_maps.get(self.flow_type)
        if hash_value is None:
            raise FlowControlException(
                f"The hash_value for the flow '{self.flow_type}' needs to be defined by the user"
            )
        return hash_value
```
### - BUFFERS

[Linux] Get RX Checksumming

```python
get_rx_checksumming(self) -> Optional[State]
```

[Linux] Set RX Checksumming

```python
set_rx_checksumming(self, value: Union[str, bool]) -> str
```

[Linux] Get TX Checksumming

```python
get_tx_checksumming(self) -> Optional[State]
```

[Linux] Set TX Checksumming

```python
set_tx_checksumming(self, value: Union[str, bool]) -> str
```

[Linux] Find Buffer sizes

```python
find_buffer_sizes(self, direction: str) -> Dict[str, str]
```

[Linux, Windows] Get RX Buffers

```python
get_rx_buffers(self, attr: BuffersAttribute = BuffersAttribute.NONE) -> int
```

[Linux, Windows] Get TX Buffers

```python
get_tx_buffers(self, attr: BuffersAttribute = BuffersAttribute.NONE) -> int
```

[Linux] Get Minimum buffer size

```python
get_min_buffers(self) -> Optional[int]
```

### - DMA

[Windows] Set dma coalescing value

```python
set_dma_coalescing(self, value: int = 0, method_registry: bool = True) -> None
```

[Windows] Get dma coalescing value from setting

```python
get_dma_coalescing(self, cached: bool = True) -> Optional[int]
```

## Static API

Package with API, which don't require object creation. 

### Basic

#### Linux
- `get_mac_address(connection: "Connection", interface_name: str, namespace: Optional[str]) -> MACAddress:` - Get MAC Address of interface

e.g. usage
```python
from mfd_network_adapter.api.basic.linux import get_mac_address
from mfd_connect import RPyCConnection


connection = RPyCConnection(ip="...")
get_mac_address(connection, interface_name="eth1", namespace="ns1")
```

#### Windows
-`get_logical_processors_count(connection: "RPyCConnection") -> int` - Get logical processors count.

### VLAN

#### ESXi
`set_vlan_tpid(connection: "Connection", tpid: str, interface_name: str) -> None:`: Set TPID used for VLAN tagging by VFs on given network interface.

### LINK

#### ESXi
`set_administrative_privileges(connection: "Connection", state: State, interface_name: str) -> None`: Set administrative link privileges.
`get_administrative_privileges(connection: "Connection", interface_name: str) -> State`: Get administrative link privileges.

### Utils

#### ESXi
`is_vib_installed(connection: "Connection", vib_name: str) -> bool`: Check if vib is installed.

## Poolmon
The `Poolmon` class provides an interface to interact with the Poolmon tool. Here is a list of its methods and their descriptions:

-`get_version(self) -> str` - Get version
-`pool_snapshot(self, log_file: str | None = None) -> "Path":` - Take a snapshot of the pool.
-`get_tag_for_interface(self, service_name: str) -> str:` - Get the poolmon tag for the interface.
-`get_values_from_snapshot(self, tag: str, output: str) -> PoolmonSnapshot` - Parse the snapshot output
### Usage

Here is a basic usage example of the `Poolmon` class:

```python
from mfd_network_adapter.network_interface.feature.memory.windows import Poolmon

# Create a connection object
connection = ...

# Create a Poolmon object
poolmon = Poolmon(connection=connection)

# Check if the Poolmon tool is available
poolmon.check_if_available()

# Get the version of the Poolmon tool
print(poolmon.get_version())

# Take a snapshot of the pool
snapshot_path = poolmon.pool_snapshot()

# Get the tag for an interface
tag = poolmon.get_tag_for_interface("service_name")

# Get values from the snapshot
values = poolmon.get_values_from_snapshot(tag, snapshot_path.read_text())
print(values)
```

Please replace `...` with the actual connection object.

## Issue reporting

If you encounter any bugs or have suggestions for improvements, you're welcome to contribute directly or open an issue [here](https://github.com/intel/mfd-network-adapter/issues).
