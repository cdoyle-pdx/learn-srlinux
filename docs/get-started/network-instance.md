---
comments: true
---

# Network Instances (VRF's)

## Overview
!!! info inline end "More Info"
    This overview is an excerpt from SR Linux operational documentation found [here](https://documentation.nokia.com/srlinux/24-7/books/config-basics/network-instances.html#basic-network-instance-configuration).

On the SR Linux device, you can configure one or more virtual routing instances, known as ‟network-instances”. Each network-instance has its own interfaces, its own protocol instances, its own route table, and its own FIB.

When a packet arrives on a subinterface associated with a network-instance, it is forwarded according to the FIB of that network-instance. Transit packets are typically forwarded out another subinterface of the network-instance.

SR Linux supports three types of network-instances: `default`, `ip-vrf`, and `mac-vrf`. Type `default` is the default network-instance (sometimes referred to as the "global" instance) and only one of this type is supported. Type `ip-vrf` is the regular (Layer-3) network-instance; you can create multiple network-instances of this type.

Type `mac-vrf` functions as a broadcast domain and is associated with an `ip-vrf` network-instance via an Integrated Routing and Bridging (IRB) to support tunneling of Layer-2 traffic across an IP network. See the [mac-vrf network-instance](https://documentation.nokia.com/srlinux/24-7/books/config-basics/network-instances.html#mac-vrf-network-instance) documentation for more information.

The SR Linux factory configuration has a management network-instance (type `ip-vrf`) configured by default. The out-of-band mgmt0 interface is the only member of this management network-instance.

**NOTE** Starting from the factory configuration, no in-band interfaces will pass traffic until the following conditions are met:
* the interfaces are configured under the `interfaces` hierarchy
* the interfaces are set to `admin-state enabled`
* a `network-instance` is created and the configured interfaces are added as members

## Configuring Lab Nodes

Our first task is to define a "default" VRF for our revenue interfaces. We can name this VRF anything, but "default" feels intuitive, doesn't it?

/// tab | `create 'default' VRF`

```srl
--{ running }--[  ]--
A:leaf1# enter candidate

--{ candidate shared default }--[  ]--
A:leaf1# network-instance default

--{ * candidate shared default }--[ network-instance default ]--
A:leaf1#
```

///
/// tab | `add interfaces (leaf1 & 2)`

```srl
--{ +* candidate shared default }--[ network-instance default ]--
A:leaf1# set interface ethernet-1/49.0

--{ +* candidate shared default }--[ network-instance default ]--
A:leaf1# set interface lo0.0

--{ +* candidate shared default }--[ network-instance default ]--
A:leaf1#

--{ +* candidate shared default }--[ network-instance default ]--
A:leaf1# /

--{ +* candidate shared default }--[  ]--
A:leaf1# commit now
All changes have been committed. Leaving candidate mode.

--{ + running }--[  ]--
A:leaf1#
```

///
/// tab | `add interfaces (spine1)`

```srl
--{ +* candidate shared default }--[ network-instance default ]--
A:spine1# set interface ethernet-1/{1,2}.0

--{ +* candidate shared default }--[ network-instance default ]--
A:spine1# /

--{ +* candidate shared default }--[  ]--
A:spine1# commit now
All changes have been committed. Leaving candidate mode.

--{ + running }--[  ]--
A:spine1#
```
///

<details>
<summary>Ranges and Wildcards (Optional, but very handy...)</summary>

<p>You might have noticed a bit of non-standard syntax in the spine1 configuration above. Rather than set ethernet-1/1.0 and ethernet-1/2.0 discretely, a range was defined using a combination of curly braces and a comma. SR Linux supports defining a range and/or using wildcards to optimize configuration and show/info output. These tools can even be combined and stacked within the same command!</p>
<b>Curly braces and a pair of periods for a range</b>
```
--{ +* candidate shared default }--[ network-instance default ]--
A:spine1# set interface ethernet-1/{1..7}.0
```
<i>This range command will include ethernet-1/1.0 through ethernet-1/7.0</i>
<br>
<br>
<b>Curly braces and commas for specific values</b>
```
--{ +* candidate shared default }--[ network-instance default ]--
A:spine1# set interface ethernet-1/{1,2,3,5,7}.0
```
<i>This range command will include ethernet-1/1.0, ethernet-1/2.0, ethernet-1/3.0, ethernet-1/5.0, and ethernet-1/7.0</i>
<br>
<br>
<b>Combine elements to optimize your command</b>
```
--{ +* candidate shared default }--[ network-instance default ]--
A:spine1# set interface ethernet-1/{1..3,5,7}.0
```
<i>This range command will include ethernet-1/1.0, ethernet-1/2.0, ethernet-1/3.0, ethernet-1/5.0, and ethernet-1/7.0</i>
<br>
<br>
<b>Use wildcards just about anywhere!</b>
```
--{ + running }--[  ]--
A:spine1# info interface *
```
<i>This will show the all configured interfaces</i>

```
--{ + running }--[  ]--
A:leaf1# info from state network-instance default route-table ipv4-unicast route * id * route-type bgp route-owner * origin-network-instance * active
```
<i>This will display the active status for all IPv4 unicast BGP routes in the default network instance route table</i>
</details>
