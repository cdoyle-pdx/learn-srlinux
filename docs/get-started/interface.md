---
comments: true
---

# SR Linux Interfaces

<script type="text/javascript" src="https://viewer.diagrams.net/js/viewer-static.min.js" async></script>

Once you feel reasonably comfortable navigating the [CLI](https://learn.srlinux.dev/get-started/cli/), you'll likely want to start connecting you lab nodes together. You know... Networking! Let's start by looking at how our lab nodes are connected to each other. There are a few ways to do this...

## Viewing/Verifying ContainerLab Links

We need to start by identifying how each node in our lab is interconnected.

### Logical Diagram

This is the drawio diagram included with the srlinux-getting-started ContainerLab files. It shows a high-level relationship between deployed nodes, but not the interfaces...

-{{ diagram(url='srl-labs/srlinux-getting-started/main/diagrams/topology.drawio', title='Lab Topology', page=0) }}-

### Phyiscal Links

Links between virtual nodes in ContainerLabs are defined within the lab's YAML configuration file. For the srlinux-getting-started lab, the file is named lab.clab.yml.

```srl
➜  srlinux-getting-started git:(main) cat lab.clab.yml
name: learn-srlinux
prefix: ""

topology:
# clipped
  links:
    # inter-switch links
    - endpoints: ["leaf1:e1-49", "spine1:e1-1"]
    - endpoints: ["leaf2:e1-49", "spine1:e1-2"]
    # client links
    - endpoints: ["client1:eth1", "leaf1:e1-1"]
    - endpoints: ["client2:eth1", "leaf2:e1-1"]
```

### Generate a Graph

ContainerLabs provides a built-in tool for generating a topology diagram, complete with links, that can be viewed with a web browser. Simply run the "clab graph" command to spawn a local web server showing the lab graph.

*NOTE: You may need to specify the YAML file in the command (as shown below) if you have more than one ContainerLab environment downloaded locally.*

*NOTE: The web server that spawns with this command runs in user-space and is deactivated when a break command is issued (usually ctrl-c).*

```srl
➜  srlinux-getting-started git:(main) sudo clab graph lab.clab.yml
INFO[0000] Parsing & checking topology file: lab.clab.yml
INFO[0000] Serving topology graph on http://0.0.0.0:50080
```

Once the topology graph is *Serving*, you can view it using by opening a browser and replacing the "0.0.0.0" in the provided URL with the IP address (or loopback if viewing locally) and specified port number (50080 in our example).

![image](https://github.com/cdoyle-pdx/learn-srlinux/blob/main/docs/images/learn-srlinux_graph.png)

*NOTE: This process assumes that your browser supports navigation to an insecure (HTTP) site and/or appropriate permissions and/or filters. Use of a VPN, firewall, or other external considerations may impact your ability to view this graph.*

/// admonition | A word on VRF's
    type: subtle-note
If you have experience on other platforms, you are likely used some form of _default_ VRF. This _default_ VRF typically possesses two common traits:

* The _default_ VRF contains all front-panel "revenue" interfaces
* The _default_ VRF is abstracted and does not require explicit configuration

The concept of an abstracted, _default_ VRF does not exist in SR Linux. The factory configuration includes a VRF ("network-instance") for the management interface, but nothing more. Connecting revenue interfaces together between nodes requires the creation of a new VRF that your connected interfaces can be added to. Network instances are covered in the next section of this guide.
///

## Routed Interface Configuration

_Adding interface configurations first allows you to take advantage of context-aware autocomplete when configuring the VRF we will add them to. Not strictly required, but useful._

Before we configure any layer-3, we need to decide on the address space we want to use. Our ultimate goal will be to use interface peering for loopback reachability, configure OSPF for loopback reachability, then configure eBGP between connected loopbacks. You can define your own nets for this step, or you can congfigure based on the allocations below:

```srl
leaf1:e1-49 (10.0.0.0/31) <-> (10.0.0.1/31) e1-1:spine1
leaf2:e1-49 (10.0.0.2/31) <-> (10.0.0.3/31) e1-2:spine1

leaf1:lo0 (172.31.0.1/32)
leaf2:lo0 (172.31.0.2/32)
spine1:lo0 (172.31.0.11/32)
```

### Verify interface layer-1/2 state

Use the `show interface` command to view active interfaces. Note the difference in output between the configured _mgmt0_ interface, and the unconfigured _ethernet-1/1 and -1/49_ interfaces. We know from our diagram that _ethernet-1/49_ connects to spine1 and we can see that it is in an _up_ state. We'll be disregarding _ethernet-1/1_ for now as it connects to a client that is out-of-scope of this guide.

_Example on leaf1 - repeat as-desired on other lab nodes_

```srl
--{ running }--[  ]--
A:leaf1# show interface
=====================================================================================================================================
ethernet-1/1 is up, speed 25G, type None
-------------------------------------------------------------------------------------------------------------------------------------
ethernet-1/49 is up, speed 100G, type None
-------------------------------------------------------------------------------------------------------------------------------------
mgmt0 is up, speed 1G, type None
  mgmt0.0 is up
    Network-instances:
      * Name: mgmt (ip-vrf)
    Encapsulation   : null
    Type            : None
    IPv4 addr    : 172.20.20.4/24 (dhcp, preferred)
    IPv6 addr    : 2001:172:20:20::4/64 (dhcp, preferred)
    IPv6 addr    : fe80::42:acff:fe14:1404/64 (link-layer, preferred)
-------------------------------------------------------------------------------------------------------------------------------------
=====================================================================================================================================
Summary
  0 loopback interfaces configured
  2 ethernet interfaces are up
  1 management interfaces are up
  1 subinterfaces are up
=====================================================================================================================================
```

### Configure IPv4 on the interfaces interconnecting leaf1, leaf2, and spine1

When configuring SR Linux, you can either "set from root" or "move and set", two terms I made up just now. 

**"Set from root"** involves using a set command from the root of the hierarchy and iterating your configuration in a single line. This is useful if set elements are linear within the hierarchy.

For example, to configure an IP address on leaf1 for ethernet-1/49, the set elements in the config hierarchy look like this:

_leaf1_
```
root
  |
  -- interface
          |
          -- ethernet-1/49
                    |
                    -- subinterface 0
                                |
                                -- ipv4
                                    |
                                    -- admin-state enable
                                    |
                                    -- address 10.0.0.0/31
```
Configure leaf1 using the "set from root" method. Don't forget to configure ethernet-1/49 **_and_** lo0!

_leaf1_
```srl
--{ candidate shared default }--[  ]--
A:leaf1# set interface ethernet-1/49 subinterface 0 ipv4 admin-state enable address 10.0.0.0/31

--{ * candidate shared default }--[  ]--
A:leaf1# set interface lo0 subinterface 0 ipv4 admin-state enable address 172.31.0.1/32

--{ * candidate shared default }--[  ]--
A:leaf1#

--{ * candidate shared default }--[  ]--
A:leaf1# commit now
All changes have been committed. Leaving candidate mode.

--{ + running }--[  ]--
A:leaf1#
```

/// admonition | Remaining in Candidate Mode
    type: note
If you wish to remain in Candidate mode, use `commit stay`.
///

!!! info inline end "Remaining in Canddidate Mode"
    If you wish to remain in Candidate mode after the commit, use the `commit stay` command.

**"Move and set"** is useful if you have set elements that are within a section of the hierarchy and are non-linear. Instead of typing out a long set command from the root of the configuration multiple times, you navigate to the deepest branch of the hierarchy that is common to the configuration you are setting.

Let's say I want to configure an IP address on leaf2 for ethernet-1/49 **_and_** set the admin-state for the ipv6 address family to `enable`:

_leaf2_
```
root
  |
  -- interface
          |
          -- ethernet-1/49
                    |
                    -- subinterface 0
                                |
                                -- ipv4
                                    |
                                    -- admin-state enable
                                    |
                                    -- address 10.0.0.2/31
                                |
                                -- ipv6
                                    |
                                    -- admin-state enable
```
We can see that the _subinterface `0`_ branch is common to the ipv4 and ipv6 elements we need to configure. 

Configuring leaf2 using the "move and set" method looks like this:

```srl
--{ * candidate shared default }--[  ]--
A:leaf2# interface ethernet-1/49 subinterface 0

--{ * candidate shared default }--[ interface ethernet-1/49 subinterface 0 ]--
A:leaf2# set ipv4 admin-state enable address 10.0.0.2/31

--{ * candidate shared default }--[ interface ethernet-1/49 subinterface 0 ]--
A:leaf2# set ipv6 admin-state enable
```
We still need to configure the loopback interface. We can jump directly there by using the `/interface lo0 subinterface 0` command:

```srl
--{ * candidate shared default }--[ interface ethernet-1/49 subinterface 0 ]--
A:leaf2# /interface lo0 subinterface 0

--{ * candidate shared default }--[ interface lo0 subinterface 0 ]--
A:leaf2# set ipv4 address 172.31.0.2/32

--{ * candidate shared default }--[ interface lo0 subinterface 0 ]--
A:leaf2#/

--{ * candidate shared default }--[  ]--
A:leaf2# commit now
All changes have been committed. Leaving candidate mode.

--{ + running }--[  ]--
A:leaf2#
```

**Now configure spine1, _ethernet-1/1_ and _ethernet-1/2_ using either method!**
