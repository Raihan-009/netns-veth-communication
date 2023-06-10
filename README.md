
# Setting up Virtual Network between Namespaces

This guide outlines the steps to create two namespaces named `blue-namespace` and `lemon-namespace`, and establish a virtual Ethernet network between them using `veth` interfaces. The goal is to enable communication between the namespaces and allow them to ping each other.

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

![Namespaces](https://github.com/Raihan-009/Linux-Network-Namespaces/blob/dev/block-diagrams/namespaces.jpg)

   ```shell
   sudo ip netns add blue-namespace
   sudo ip netns add lemon-namespace
   ```
   This step creates two namespaces named `blue-namespace` and `lemon-namespace`.

## 3. Create the virtual Ethernet link pair:

![Namespaces](https://github.com/Raihan-009/Linux-Network-Namespaces/blob/dev/block-diagrams/veth-cable.jpg)

   ```shell
   sudo ip link add veth-blue type veth peer name veth-lemon
   ```
   This command creates a virtual Ethernet link pair consisting of `veth-blue` and `veth-lemon` at `root namespace`.

## 4. Set the cable as NIC

![Namespaces](https://github.com/Raihan-009/Linux-Network-Namespaces/blob/dev/block-diagrams/nic.jpg)

   ```shell
   sudo ip link set veth-blue netns blue-namespace
   sudo ip link set veth-lemon netns lemon-namespace
   ```
   This command acts as  `NIC` link pair consisting of `veth-blue` and `veth-lemon`. 

## 5. Assign IP Addresses to the Interfaces

![Namespaces](https://github.com/Raihan-009/Linux-Network-Namespaces/blob/dev/block-diagrams/assigned-ip.jpg)

   ```shell
   sudo ip netns exec blue-namespace ip addr add 192.168.0.1/24 dev veth-blue
   sudo ip netns exec lemon-namespace ip addr add 192.168.0.2/24 dev veth-lemon
   ```
   In this step, IP addresses are assigned to the veth-blue interface in the blue-namespace and to the veth-lemon interface in the lemon-namespace.

## 6. Set the Interfaces Up

![Namespaces](https://github.com/Raihan-009/Linux-Network-Namespaces/blob/dev/block-diagrams/up-state.jpg)

   ```shell
   sudo ip netns exec blue-namespace ip link set veth-blue up
   sudo ip netns exec lemon-namespace ip link set veth-lemon up
   ```
   These commands set the veth-blue and veth-lemon interfaces `up`, enabling them to transmit and receive data.

## 7. Set Default Routes

   ```shell
   sudo ip netns exec blue-namespace ip route add default via 192.168.0.1 dev veth-blue
   sudo ip netns exec lemon-namespace ip route add default via 192.168.0.2 dev veth-lemon
   ```
   These commands set the `default routes` within each namespace, allowing them to route network traffic.

## 8. Test Connectivity

   ```shell
   sudo ip netns exec blue-namespace ping 192.168.0.2
   sudo ip netns exec lemon-namespace ping 192.168.0.1
   ```
   Use these commands to test the connectivity between the namespaces by pinging each other's IP address.

## 9. Clean Up (optional)

   ```shell
   sudo ip netns del blue-namespace
   sudo ip netns del lemon-namespace
   ```
   If you want to remove the namespaces run these commands to clean up the setup.