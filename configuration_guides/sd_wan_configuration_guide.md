# FortiGate SD-WAN Configuration Guide for VPN Optimization

## Introduction

This guide details the configuration of FortiGate SD-WAN to optimize IPsec VPN traffic, providing enhanced performance, reliability, and intelligent path selection for inter-site connectivity. By integrating VPN tunnels into the SD-WAN fabric, organizations can leverage multiple WAN links to dynamically route VPN traffic based on real-time link quality and application requirements.

## Prerequisites

Before configuring SD-WAN for VPN optimization, ensure the following:

*   FortiGate devices are deployed at all sites requiring SD-WAN and VPN connectivity.
*   Multiple WAN interfaces are configured and connected to different ISPs or network links.
*   IPsec VPN tunnels are pre-configured between the FortiGate devices, with each tunnel utilizing a distinct WAN interface. (Refer to the IPsec VPN Configuration Guide for details).
*   Administrative access to the FortiGate devices.
*   FortiOS version 7.0 or later is recommended for full SD-WAN zone support for static routes.

## Network Topology (Example)

Consider a scenario with a Head Office (HO) and a Branch Office (BO), each with two WAN links and two IPsec VPN tunnels established between them.

*   **Head Office FortiGate**
    *   WAN1 Interface: `port1` (ISP1, Public IP: `203.0.113.1`)
    *   WAN2 Interface: `port3` (ISP2, Public IP: `203.0.113.2`)
    *   Internal Network: `192.168.1.0/24`
    *   IPsec VPN Tunnel 1 (over WAN1): `HO-to-BO-VPN1`
    *   IPsec VPN Tunnel 2 (over WAN2): `HO-to-BO-VPN2`

*   **Branch Office FortiGate**
    *   WAN1 Interface: `port1` (ISP1, Public IP: `198.51.100.1`)
    *   WAN2 Interface: `port3` (ISP2, Public IP: `198.51.100.2`)
    *   Internal Network: `192.168.2.0/24`
    *   IPsec VPN Tunnel 1 (over WAN1): `BO-to-HO-VPN1`
    *   IPsec VPN Tunnel 2 (over WAN2): `BO-to-HO-VPN2`

## Configuration Steps

### Step 1: Create SD-WAN Zone

First, create an SD-WAN zone to group the VPN interfaces. This zone will be used for intelligent traffic steering.

1.  Navigate to **Network > SD-WAN**.
2.  Under **SD-WAN Zones**, click **Create New**.
3.  **Name**: `VPN_SDWAN_Zone` (or a descriptive name).
4.  **Type**: `Software Switch`.
5.  Leave **Members** empty for now.
6.  Click **OK**.

### Step 2: Add IPsec VPN Tunnels as SD-WAN Members

Add the pre-configured IPsec VPN tunnel interfaces to the newly created SD-WAN zone. It is crucial that these VPN tunnels were created manually (not via the IPsec VPN wizard) to be eligible as SD-WAN members.

1.  Navigate to **Network > SD-WAN**.
2.  Under **SD-WAN Interface Members**, click **Create New**.
3.  **Interface**: Select `HO-to-BO-VPN1` (the first IPsec tunnel interface).
4.  **Zone**: Select `VPN_SDWAN_Zone`.
5.  **Gateway**: Leave as `0.0.0.0` if the VPN tunnel itself handles routing, or specify if needed.
6.  Click **OK**.
7.  Repeat steps 2-6 for `HO-to-BO-VPN2` (the second IPsec tunnel interface).

### Step 3: Configure Performance SLA (Service Level Agreement)

Performance SLAs monitor the quality of each SD-WAN member (VPN tunnel) by sending probes and measuring metrics like latency, jitter, and packet loss. This data is used by SD-WAN rules for intelligent path selection.

1.  Navigate to **Network > SD-WAN**.
2.  Under **Performance SLA**, click **Create New**.
3.  **Name**: `VPN_SLA_Monitor`
4.  **Protocol**: `Ping` (ICMP) is commonly used for basic reachability and latency. You can also use `HTTP`, `HTTPS`, `DNS`, or `TCP` for application-specific monitoring.
5.  **Server**: Enter the IP address of a reliable server in the remote network (e.g., `192.168.2.10` in the Branch Office LAN). This server should be reachable through the VPN tunnels.
6.  **Interval**: `500` ms (how often probes are sent).
7.  **Failures before inactive**: `5` (number of consecutive failures before a link is marked inactive).
8.  **Restore after**: `5` (number of consecutive successes before an inactive link is restored).
9.  **Apply to Members**: Select both `HO-to-BO-VPN1` and `HO-to-BO-VPN2`.
10. **Thresholds**: Define acceptable thresholds for latency, jitter, and packet loss. For example:
    *   **Latency Threshold**: `100` ms
    *   **Jitter Threshold**: `50` ms
    *   **Packet Loss Threshold**: `2` %
11. Click **OK**.

#### CLI Configuration for Performance SLA Source (Important!)

For Performance SLA to function correctly, you might need to explicitly define the source interface for the probes via CLI, especially if the FortiGate has multiple internal interfaces or complex routing.

```cli
config system sdwan
    config members
        edit <member_id_for_VPN1>  # Find member ID using 'show system sdwan members'
            set source <local_LAN_interface_IP> # e.g., 192.168.1.1
        next
        edit <member_id_for_VPN2>  # Find member ID for VPN2
            set source <local_LAN_interface_IP> # e.g., 192.168.1.1
        next
    end
end
```

Replace `<member_id_for_VPN1>`, `<member_id_for_VPN2>`, and `<local_LAN_interface_IP>` with your specific values. The source IP must be part of the local network that is allowed through the VPN tunnel.

### Step 4: Configure SD-WAN Rules

SD-WAN rules dictate how traffic is steered across the available VPN tunnels based on various criteria and the Performance SLA metrics.

1.  Navigate to **Network > SD-WAN**.
2.  Under **SD-WAN Rules**, click **Create New**.
3.  **Name**: `VPN_Traffic_Optimization`
4.  **Source Interface**: `port2` (Head Office LAN interface)
5.  **Source Address**: `all` (or specific internal subnets/hosts)
6.  **Destination Address**: `Branch_Office_LAN` (remote network subnet)
7.  **Service**: `ALL` (or specific services like `HTTP`, `SMB`, `RDP`)
8.  **Outgoing Interfaces**: Select `VPN_SDWAN_Zone`.
9.  **Strategy**: Choose a strategy based on your optimization goals:
    *   **Best Quality**: Prioritizes the link with the best quality based on SLA metrics (e.g., lowest latency, lowest jitter, lowest packet loss).
    *   **Lowest Cost (SLA)**: Prioritizes the link that meets SLA requirements with the lowest cost (if costs are assigned to interfaces).
    *   **Maximize Bandwidth (SLA)**: Distributes traffic across multiple links to maximize aggregate bandwidth.
    *   **Redundant Interface**: Uses a primary link and fails over to a secondary if the primary fails.
    *   For VPN optimization, **Best Quality** or **Lowest Cost (SLA)** are often preferred.
10. **Measured SLA**: Select `VPN_SLA_Monitor`.
11. **Load Balancing Algorithm**: If using Maximize Bandwidth, choose an algorithm (e.g., `Source IP Based`, `Weight Based`).
12. Click **OK**.

**Important**: Ensure this SD-WAN rule is placed *above* any general 


internet access rules to ensure VPN traffic is correctly steered.

### Step 5: Configure Firewall Policies

Firewall policies are required to allow traffic to flow from your internal network through the SD-WAN zone to the remote VPN networks.

1.  Navigate to **Policy & Objects > Firewall Policy**.
2.  Click **Create New**.
3.  **Name**: `HO-LAN-to-VPN_SDWAN`
4.  **Incoming Interface**: `port2` (Head Office LAN interface)
5.  **Outgoing Interface**: `VPN_SDWAN_Zone`
6.  **Source**: `all` (or specific source addresses/groups from HO LAN)
7.  **Destination**: `Branch_Office_LAN` (remote network subnet)
8.  **Schedule**: `always`
9.  **Service**: `ALL` (or specific services)
10. **Action**: `ACCEPT`
11. **NAT**: `Disable` (as traffic is routed between private networks)
12. **Log Allowed Traffic**: `All Sessions`
13. Click **OK**.

### Step 6: Configure Static Routes for SD-WAN Zone

Ensure that traffic destined for the remote network is routed through the `VPN_SDWAN_Zone`.

1.  Navigate to **Network > Static Routes**.
2.  Click **Create New**.
3.  **Destination**: `Subnet` `192.168.2.0/24` (Branch Office network)
4.  **Device**: `VPN_SDWAN_Zone`
5.  **Administrative Distance**: `10` (default, can be adjusted)
6.  Click **OK**.

### Step 7: Configure Blackhole Route (Optional but Recommended)

A blackhole route prevents traffic from being routed over the internet if all VPN tunnels to a specific destination are down. This ensures traffic is dropped rather than sent unencrypted or to an incorrect destination.

1.  Access the FortiGate CLI.
2.  Execute the following commands:
    ```cli
    config router static
        edit <next_available_entry_id>  # e.g., 4
            set dst 192.168.2.0 255.255.255.0  # Remote network subnet
            set distance 250  # Higher than normal static routes
            set blackhole enable
        end
    ```
    Replace `<next_available_entry_id>` with an unused static route ID and `192.168.2.0 255.255.255.0` with your remote network details.

## Performance Report

To evaluate the effectiveness of the SD-WAN integration with VPN, regular monitoring and performance reporting are essential. The goal is to ensure that VPN traffic is consistently using the optimal path and that failover mechanisms work as expected.

### Key Performance Indicators (KPIs)

*   **Latency**: Time delay for packets to travel from source to destination. Lower is better.
*   **Jitter**: Variation in packet delay. Lower is better, especially for real-time applications.
*   **Packet Loss**: Percentage of packets that fail to reach their destination. Zero is ideal.
*   **Throughput**: Actual data transfer rate. Higher is better.
*   **Link Utilization**: Bandwidth usage of each WAN link.

### Monitoring Tools

FortiGate provides several tools for monitoring SD-WAN and VPN performance:

*   **SD-WAN Monitor**: Navigate to **Network > SD-WAN Monitor**. This dashboard provides real-time visibility into the status of SD-WAN members, their performance metrics (latency, jitter, packet loss), and the active link chosen by SD-WAN rules.
*   **Log & Report**: Review **Traffic Logs** and **System Events** for information on traffic flow, policy hits, and any SD-WAN or VPN related events (e.g., link failures, failovers).
*   **CLI Commands**: Advanced monitoring can be done via CLI:
    *   `diagnose sys sdwan health-check`: Shows the status of health checks.
    *   `diagnose sys sdwan service`: Displays SD-WAN rule information and active members.
    *   `diagnose vpn tunnel list`: Shows IPsec tunnel status.

### Reporting Methodology

1.  **Baseline Measurement**: Before implementing SD-WAN, establish baseline performance metrics for VPN traffic over individual WAN links.
2.  **Continuous Monitoring**: Use FortiGate's built-in monitoring tools to continuously track KPIs for each VPN tunnel within the SD-WAN zone.
3.  **Data Collection**: Collect data over a period (e.g., daily, weekly) to identify trends and anomalies.
4.  **Analysis**: Compare current performance against baselines and SLA thresholds. Analyze events like link degradation or failovers.
5.  **Optimization**: Based on the analysis, fine-tune SD-WAN rules, SLA thresholds, or even network configurations to further optimize performance.

### Sample Performance Report Structure

#### Executive Summary

Brief overview of SD-WAN performance, key findings, and recommendations.

#### Current SD-WAN Status

*   Overall health of the `VPN_SDWAN_Zone`.
*   Status of individual VPN tunnels (up/down, active/inactive).

#### Performance Metrics

| Metric       | VPN Tunnel 1 (ISP1) | VPN Tunnel 2 (ISP2) | Target (SLA) | Observations                                  |
| :----------- | :------------------ | :------------------ | :----------- | :-------------------------------------------- |
| **Latency**  | 25 ms               | 40 ms               | < 50 ms      | VPN Tunnel 1 consistently lower latency.      |
| **Jitter**   | 5 ms                | 12 ms               | < 20 ms      | Both links within acceptable jitter.          |
| **Packet Loss**| 0%                  | 1%                  | < 2%         | VPN Tunnel 2 experienced minor packet loss.   |
| **Throughput** | 80 Mbps             | 60 Mbps             | > 50 Mbps    | Both links meeting minimum throughput.        |

#### SD-WAN Rule Effectiveness

*   Confirmation that `VPN_Traffic_Optimization` rule is correctly steering traffic.
*   Examples of traffic flows and the chosen outgoing interface.

#### Failover Testing Results

*   Details of any simulated or actual link failures.
*   Time taken for failover to occur.
*   Impact on user experience during failover.

#### Recommendations

*   Adjust SLA thresholds if needed.
*   Consider bandwidth upgrades for underperforming links.
*   Review firewall policies for further optimization.

This comprehensive approach to SD-WAN configuration and performance reporting ensures that your FortiGate-powered VPN infrastructure is robust, efficient, and aligned with business needs.


