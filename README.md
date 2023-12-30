
# Setting up Virtual Network between Namespaces

This guide outlines the steps to create two namespaces named ***blue-namespace*** and ***lemon-namespace***, and establish a virtual Ethernet network between them using ***veth*** interfaces. The goal is to enable communication between the namespaces and allow them to ping each other.

## Prerequisites

- Linux operating system
- Root or sudo access
- Packages

    ```bash
    sudo apt update
    sudo apt upgrade -y
    sudo apt install iproute2
    sudo apt install net-tools
    ```

# Steps

## 1. Enable IP forwarding in the Linux kernel:

   ```shell
   sudo sysctl -w net.ipv4.ip_forward=1
   ```
   This step enables IP forwarding in the Linux kernel, allowing the namespaces to communicate with each other.

## 2. Create namespaces:

![Namespaces](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/19b3e3df-0f1b-4a9b-a0d9-80e61b766a6e.png)

   ```shell
   sudo ip netns add blue-namespace
   sudo ip netns add lemon-namespace
   ```
   This step creates two namespaces named ***blue-namespace*** and ***lemon-namespace***.

## 3. Create the virtual Ethernet link pair:

![Veth Cable](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/f3053bd2-ed2a-4ccb-b5ee-9d730d23e409.png)

   ```shell
   sudo ip link add veth-blue type veth peer name veth-lemon
   ```
   This command creates a virtual Ethernet link pair consisting of veth-blue and veth-lemon at ***root namespace***.

In order to verify, run `sudo ip link list`

**Expected Output:**

```bash
6: veth-lemon@veth-blue: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 22:21:fc:9e:d0:2b brd ff:ff:ff:ff:ff:ff
7: veth-blue@veth-lemon: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 2e:34:8e:0c:1c:6e brd ff:ff:ff:ff:ff:ff
```

## 4. Set the cable as NIC

![Nic](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/2896cbaa-3d58-4079-9e8f-7a80451713c7.png)

   ```shell
   sudo ip link set veth-blue netns blue-namespace
   sudo ip link set veth-lemon netns lemon-namespace
   ```
   This command acts as  ***NIC*** link pair consisting of veth-blue and veth-lemon. 

To verify run `sudo ip netns exec blue-namespace ip link` and `sudo ip netns exec lemon-namespace ip link`

**Expected Output**

```bash
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
7: veth-blue@if6: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 2e:34:8e:0c:1c:6e brd ff:ff:ff:ff:ff:ff link-netns lemon-namespace
```
and 
```bash
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
6: veth-lemon@if7: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 22:21:fc:9e:d0:2b brd ff:ff:ff:ff:ff:ff link-netns blue-namespace
```

But as we see, interface has been created but it's **DOWN** and has no ip. Now assign a ip address and turn it **UP**.

## 5. Assign IP Addresses to the Interfaces

![Ip](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/dd04edc1-9c80-46ac-a2cf-ba60086944aa.png)

   ```shell
   sudo ip netns exec blue-namespace ip addr add 192.168.0.1/24 dev veth-blue
   sudo ip netns exec lemon-namespace ip addr add 192.168.0.2/24 dev veth-lemon
   ```
   In this step, IP addresses are assigned to the veth-blue interface in the blue-namespace and to the veth-lemon interface in the lemon-namespace.

To verify run `sudo ip netns exec blue-namespace ip addr` and `sudo ip netns exec lemon-namespace ip addr`

**Expected Output:**
```bash
7: veth-blue@if6: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 2e:34:8e:0c:1c:6e brd ff:ff:ff:ff:ff:ff link-netns lemon-namespace
    inet 192.168.0.1/24 scope global veth-blue
       valid_lft forever preferred_lft forever
```
and
```bash
6: veth-lemon@if7: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 22:21:fc:9e:d0:2b brd ff:ff:ff:ff:ff:ff link-netns blue-namespace
    inet 192.168.0.2/24 scope global veth-lemon
       valid_lft forever preferred_lft forever
```

## 6. Set the Interfaces Up

![Interface](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/dea5d936-bc84-491b-9268-323be3a8ba32.png)

   ```shell
   sudo ip netns exec blue-namespace ip link set veth-blue up
   sudo ip netns exec lemon-namespace ip link set veth-lemon up
   ```
   These commands set the veth-blue and veth-lemon interfaces ***up***, enabling them to transmit and receive data.

Now run again `sudo ip netns exec blue-namespace ip link` and `sudo ip netns exec lemon-namespace ip link` to verify

**Expected Output:**
```bash
7: veth-blue@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 2e:34:8e:0c:1c:6e brd ff:ff:ff:ff:ff:ff link-netns lemon-namespace
```
and
```bash
6: veth-lemon@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 22:21:fc:9e:d0:2b brd ff:ff:ff:ff:ff:ff link-netns blue-namespace
```

## 7. Set Default Routes

   ```shell
   sudo ip netns exec blue-namespace ip route add default via 192.168.0.1 dev veth-blue
   sudo ip netns exec lemon-namespace ip route add default via 192.168.0.2 dev veth-lemon
   ```
   These commands set the **default routes** within each namespace, allowing them to route network traffic.

In order to verify run `sudo ip netns exec blue-namespace ip route` and `sudo ip netns exec lemon-namespace ip route`

**Expected Output:**
```bash
default via 192.168.0.1 dev veth-blue 
192.168.0.0/24 dev veth-blue proto kernel scope link src 192.168.0.1
```
and
```bash
default via 192.168.0.2 dev veth-lemon 
192.168.0.0/24 dev veth-lemon proto kernel scope link src 192.168.0.2
```
### In addition, the `route` command in the context of the `ip netns exec` allows you to view the routing table of a specific network namespace. The routing table contains information about how network traffic should be forwarded or delivered.

To view the routing table of the lemon-namespace, you can execute the following command:

```shell
sudo ip netns exec lemon-namespace route
```
***Output***
```bash
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         192.168.0.2     0.0.0.0         UG    0      0        0 veth-lemon
192.168.0.0     0.0.0.0         255.255.255.0   U     0      0        0 veth-lemon
```
To view the routing table of the blue-namespace, you can execute the following command:

```shell
sudo ip netns exec blue-namespace route
```
**Output**
```bash
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         192.168.0.1     0.0.0.0         UG    0      0        0 veth-blue
192.168.0.0     0.0.0.0         255.255.255.0   U     0      0        0 veth-blue
```

## 8. Test Connectivity

   ```shell
   sudo ip netns exec blue-namespace ping 192.168.0.2
   sudo ip netns exec lemon-namespace ping 192.168.0.1
   ```
   Use these commands to test the connectivity between the namespaces by pinging each other's IP address.

**Expected Output:**
```bash
PING 192.168.0.2 (192.168.0.2) 56(84) bytes of data.
64 bytes from 192.168.0.2: icmp_seq=1 ttl=64 time=0.024 ms
64 bytes from 192.168.0.2: icmp_seq=2 ttl=64 time=0.069 ms
64 bytes from 192.168.0.2: icmp_seq=3 ttl=64 time=0.063 ms
64 bytes from 192.168.0.2: icmp_seq=4 ttl=64 time=0.064 ms
64 bytes from 192.168.0.2: icmp_seq=5 ttl=64 time=0.063 ms
^C
--- 192.168.0.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4099ms
rtt min/avg/max/mdev = 0.024/0.056/0.069/0.016 ms
```

and

```bash
PING 192.168.0.1 (192.168.0.1) 56(84) bytes of data.
64 bytes from 192.168.0.1: icmp_seq=1 ttl=64 time=0.033 ms
64 bytes from 192.168.0.1: icmp_seq=2 ttl=64 time=0.072 ms
64 bytes from 192.168.0.1: icmp_seq=3 ttl=64 time=0.071 ms
64 bytes from 192.168.0.1: icmp_seq=4 ttl=64 time=0.074 ms
64 bytes from 192.168.0.1: icmp_seq=5 ttl=64 time=0.070 ms
^C
--- 192.168.0.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4099ms
rtt min/avg/max/mdev = 0.033/0.064/0.074/0.015 ms
```

### Furthermore, the `arp` command in the context of the `ip netns exec` allows you to view the ARP cache of a specific network namespace. The ARP cache contains mappings of IP addresses to MAC addresses.

To view the ARP cache of the blue-namespace, you can execute the following command:

```shell
sudo ip netns exec blue-namespace arp
```
***Output***
```bash
Address                  HWtype  HWaddress           Flags Mask            Iface
127.0.0.53                       (incomplete)                              veth-blue
192.168.0.2              ether   22:21:fc:9e:d0:2b   C                     veth-blue
```
To view the ARP cache of the lemon-namespace, you can execute the following command:

```shell
sudo ip netns exec lemon-namespace arp
```

***Output***
```bash
Address                  HWtype  HWaddress           Flags Mask            Iface
192.168.0.1              ether   2e:34:8e:0c:1c:6e   C                     veth-lemon
127.0.0.53                       (incomplete)                              veth-lemon
```
## 9. Clean Up (optional)

   ```shell
   sudo ip netns del blue-namespace
   sudo ip netns del lemon-namespace
   ```
   If you want to remove the namespaces run these commands to clean up the setup.

   # Cheers! 🍻 Have a good day!