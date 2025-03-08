# GRO and LRO: Tuning Kernel Parameters for Efficient Overlay Traffic Handling

When running overlay networks (such as those used in Kubernetes with Cilium or Docker Swarm), **GRO (Generic Receive Offload)** and **LRO (Large Receive Offload)** are kernel features 
that can significantly impact the performance of the network stack by reducing CPU load and increasing throughput. 
Properly tuning these kernel parameters can help handle overlay traffic more efficiently.

## 1. GRO (Generic Receive Offload)

**GRO** is a kernel feature that allows the network interface card (NIC) to accumulate multiple packets into a single larger packet before passing it up to the kernel network stack. 
This reduces the overhead of processing each packet individually, especially useful in high-throughput environments.

### How it works:
- **GRO** works by coalescing multiple small packets (typically part of the same flow) into a larger more efficient packet.
- This reduces the number of interrupts and CPU cycles required for packet processing.
- GRO works on lower layers L2/L3 and hence protocol agnostic
- GRO is a NIC card feature.

### Overlay networks:
- Since overlay traffic involves tunneling packets (e.g., VXLAN, GRE), the original packets are encapsulated and sent across the network to the destination.
- At the destination, they are decapsulated. Optimizing **GRO** helps by allowing the kernel to efficiently handle these decapsulated packets and reduce processing overhead.

### Tuning GRO for Overlay Traffic:
- **Command to check if GRO is enabled**: This is usually enabled by default

   ```
   # Check GRO status on an interface (e.g. eth0)
   ethtool -K <interface-name> | grep gro
   ```

- **Enable GRO on the network interfaces**: If GRO is not enabled by default, it can be enabled using the following command:
    ```
    # Enable GRO on an interface (e.g., eth0)
    sudo ethtool -K eth0 gro on
    ```

- **Tune maximum packet size for GRO**: GRO can handle packets up to a certain size. We can tune the GRO maximum segment size by adjusting the `rx_buffer_len`:
    ```
    # Example of adjusting GRO parameters for performance
    sysctl -w net.core.rmem_max=16777216
    ```

- **Monitor GRO performance**: Check if GRO is active and examine its performance:
    ```bash
    # Check GRO status on the network interface
    ethtool -S eth0 | grep gro
    ```

## 2. LRO (Large Receive Offload)

**LRO** is similar to GRO but works on a used on L4/L5 layer. It is typically used in scenarios where TCP traffic is processed.
It combines multiple incoming TCP packets into a single, larger one before handing it off to the kernel. 
This can help improve throughput and reduce CPU overhead for TCP traffic.

### How it works:
- **LRO** is more specific to **TCP** traffic and it reduces the **number of TCP segments** that need to be processed
- Improves effective throughput by reducing the interrupt rate.
- LRO is implemented in kernel level, some interface may support in NIC (don't know the exact details)
- LRO typically used for TCP/IP stack optimizations for improving throughput when handling high numbers of TCP packets

### Overlay networks:
- In overlay networks, if thera are applications that generate a lot of TCP traffic (e.g., microservices within a Kubernetes environment)
- Enabling **LRO** can help reduce the processing cost for each individual packet.

### Tuning LRO for Overlay Traffic:
- **Enable LRO on the network interfaces**:
    ```
    # Enable LRO on an interface (e.g., eth0)
    sudo ethtool -K eth0 lro on
    ```

- **Monitor LRO performance**: We can monitor whether **LRO** is enabled and check performance by using:
    ```
    # Check LRO status on the network interface
    ethtool -S eth0 | grep lro
    ```

## 3. Other Kernel Parameters for Overlay Traffic

In addition to tuning **GRO** and **LRO**, there are other kernel parameters that can further optimize overlay traffic performance, such as:

### a. Tuning the MTU (Maximum Transmission Unit):
For overlay networks using protocols like **VXLAN**, **GRE**, or **IPsec**, it is crucial to adjust the MTU on the network interfaces to 
account for the additional overhead introduced by encapsulation.

- **Set MTU to account for tunneling overhead**: For instance, VXLAN adds an overhead of 50 bytes, so the MTU needs to be adjusted to avoid fragmentation:
    ```
    sudo ifconfig eth0 mtu 1450
    ```
- Adjust the MTU for all interfaces involved in the overlay network setup (host and virtual interfaces).

### b. Tune the TCP/UDP Receive Buffers:
Large overlay traffic can involve large numbers of small packets, which can benefit from larger buffer sizes.
    ```
    # Adjust buffer sizes for better throughput
    sysctl -w net.core.rmem_default=16777216
    sysctl -w net.core.rmem_max=16777216
    ```

### c. Enable TCP Segmentation Offload (TSO):
**TSO** can offload the segmentation of large data streams into smaller TCP segments to the NIC, reducing CPU usage.
    ```
    # Enable TSO on a network interface (e.g., eth0)
    sudo ethtool -K eth0 tso on
    ```

## 5. Practical Considerations

- **Hardware Offloading**: If  network interface supports it, enabling hardware offload features such as **GRO**, **LRO**, and **TSO** directly on the NIC
  can provide significant performance improvements, reducing the load on the CPU.
- **Testing and Monitoring**: After tuning these parameters, it’s essential to monitor the system’s performance.
- Tools like **netstat**, **iftop**, and **ethtool** can be useful for analyzing network traffic and diagnosing issues.
