# FortiGate IPsec VPN Configuration Guide

## Introduction

This guide provides detailed instructions for configuring a Site-to-Site IPsec VPN tunnel between two FortiGate devices. IPsec VPNs are crucial for securely connecting geographically dispersed networks, such as a head office to a branch office, ensuring data confidentiality, integrity, and authenticity over untrusted networks like the internet.

## Prerequisites

Before proceeding with the configuration, ensure the following:

*   Two FortiGate firewalls (FortiGate A and FortiGate B) with administrative access.
*   FortiOS version 7.6.4 or later (as per the referenced documentation).
*   Publicly routable IP addresses on the WAN interfaces of both FortiGate devices.
*   Knowledge of the internal network subnets at both sites (e.g., Head Office LAN: `192.168.1.0/24`, Branch Office LAN: `192.168.2.0/24`).
*   A pre-shared key for authentication (or digital certificates if using a more advanced setup).

## Network Topology (Example)

*   **FortiGate A (Head Office)**
    *   WAN Interface: `port1` (Public IP: `203.0.113.1`)
    *   LAN Interface: `port2` (Internal IP: `192.168.1.1/24`)
    *   Internal Network: `192.168.1.0/24`

*   **FortiGate B (Branch Office)**
    *   WAN Interface: `port1` (Public IP: `198.51.100.1`)
    *   LAN Interface: `port2` (Internal IP: `192.168.2.1/24`)
    *   Internal Network: `192.168.2.0/24`

## Configuration Steps

### FortiGate A (Head Office) Configuration

#### Step 1: Create Phase 1 (IKE Gateway)

Phase 1 establishes a secure control channel between the two FortiGate devices.

1.  Navigate to **VPN > IPsec Tunnels**.
2.  Click **Create New** and select **IPsec Tunnel**.
3.  Choose **Custom** template.
4.  **Name**: `HO-to-Branch`
5.  **Network**:
    *   **IP Version**: `IPv4`
    *   **Interface**: `port1` (WAN interface of FortiGate A)
    *   **Remote Gateway**: `Static IP Address`
    *   **IP Address**: `198.51.100.1` (Public IP of FortiGate B)
    *   **Local Gateway IP**: `0.0.0.0` (or specific WAN IP if multiple exist)
    *   **NAT Traversal**: `Disable` (if both ends have public IPs)
    *   **Dead Peer Detection**: `On Demand` or `Always On` (recommended `Always On`)
6.  **Authentication**:
    *   **Method**: `Pre-shared Key`
    *   **Pre-shared Key**: Enter a strong, complex key (e.g., `YourStrongPSKHere`)
    *   **IKE Version**: `IKEv2` (recommended for better security and features)
7.  **Phase 1 Proposal**:
    *   **Encryption**: `AES256`
    *   **Authentication**: `SHA256`
    *   **Diffie-Hellman Group**: `14` (or higher)
    *   **Key Lifetime**: `86400` seconds (24 hours)
8.  Click **OK**.

#### Step 2: Create Phase 2 (IPsec Tunnel)

Phase 2 defines the parameters for the actual data tunnel and the networks that will communicate.

1.  Under **VPN > IPsec Tunnels**, edit the `HO-to-Branch` tunnel.
2.  Under **Phase 2 Selectors**, click **Create New**.
3.  **Name**: `HO-to-Branch-P2`
4.  **Local Address**: `192.168.1.0/24` (Head Office internal network)
5.  **Remote Address**: `192.168.2.0/24` (Branch Office internal network)
6.  **Phase 2 Proposal**:
    *   **Encryption**: `AES256`
    *   **Authentication**: `SHA256`
    *   **Perfect Forward Secrecy (PFS)**: `Enable`
    *   **Diffie-Hellman Group**: `14` (or higher, match Phase 1)
    *   **Key Lifetime**: `3600` seconds (1 hour)
7.  Click **OK**.

#### Step 3: Create Firewall Addresses for Remote Network

1.  Navigate to **Policy & Objects > Addresses**.
2.  Click **Create New > Address**.
3.  **Name**: `Branch_Office_LAN`
4.  **Type**: `Subnet`
5.  **Subnet/IP Range**: `192.168.2.0/24`
6.  **Interface**: `any`
7.  Click **OK**.

#### Step 4: Create Firewall Policies

Policies are needed to allow traffic to flow through the IPsec tunnel.

1.  Navigate to **Policy & Objects > Firewall Policy**.
2.  Click **Create New**.
3.  **Name**: `HO-LAN-to-Branch-VPN`
4.  **Incoming Interface**: `port2` (Head Office LAN interface)
5.  **Outgoing Interface**: `HO-to-Branch` (the IPsec tunnel interface)
6.  **Source**: `all` (or specific source addresses/groups from HO LAN)
7.  **Destination**: `Branch_Office_LAN`
8.  **Schedule**: `always`
9.  **Service**: `ALL` (or specific services like `PING`, `HTTP`, `HTTPS`)
10. **Action**: `ACCEPT`
11. **NAT**: `Disable` (as traffic is routed between private networks)
12. **Log Allowed Traffic**: `All Sessions`
13. Click **OK**.

14. Click **Create New** for the return traffic.
15. **Name**: `Branch-VPN-to-HO-LAN`
16. **Incoming Interface**: `HO-to-Branch` (the IPsec tunnel interface)
17. **Outgoing Interface**: `port2` (Head Office LAN interface)
18. **Source**: `Branch_Office_LAN`
19. **Destination**: `all` (or specific destination addresses/groups in HO LAN)
20. **Schedule**: `always`
21. **Service**: `ALL`
22. **Action**: `ACCEPT`
23. **NAT**: `Disable`
24. **Log Allowed Traffic**: `All Sessions`
25. Click **OK**.

#### Step 5: Configure Static Route

Ensure traffic destined for the branch office network is routed through the VPN tunnel.

1.  Navigate to **Network > Static Routes**.
2.  Click **Create New**.
3.  **Destination**: `Subnet` `192.168.2.0/24`
4.  **Device**: `HO-to-Branch` (the IPsec tunnel interface)
5.  **Administrative Distance**: `10` (default, can be adjusted if needed)
6.  Click **OK**.

### FortiGate B (Branch Office) Configuration

Repeat the steps above on FortiGate B, swapping local and remote network details.

#### Step 1: Create Phase 1 (IKE Gateway)

1.  Navigate to **VPN > IPsec Tunnels**.
2.  Click **Create New** and select **IPsec Tunnel**.
3.  Choose **Custom** template.
4.  **Name**: `Branch-to-HO`
5.  **Network**:
    *   **IP Version**: `IPv4`
    *   **Interface**: `port1` (WAN interface of FortiGate B)
    *   **Remote Gateway**: `Static IP Address`
    *   **IP Address**: `203.0.113.1` (Public IP of FortiGate A)
    *   **Local Gateway IP**: `0.0.0.0`
    *   **NAT Traversal**: `Disable`
    *   **Dead Peer Detection**: `Always On`
6.  **Authentication**:
    *   **Method**: `Pre-shared Key`
    *   **Pre-shared Key**: Enter the *same* strong, complex key used on FortiGate A.
    *   **IKE Version**: `IKEv2`
7.  **Phase 1 Proposal**:
    *   **Encryption**: `AES256`
    *   **Authentication**: `SHA256`
    *   **Diffie-Hellman Group**: `14`
    *   **Key Lifetime**: `86400` seconds
8.  Click **OK**.

#### Step 2: Create Phase 2 (IPsec Tunnel)

1.  Under **VPN > IPsec Tunnels**, edit the `Branch-to-HO` tunnel.
2.  Under **Phase 2 Selectors**, click **Create New**.
3.  **Name**: `Branch-to-HO-P2`
4.  **Local Address**: `192.168.2.0/24` (Branch Office internal network)
5.  **Remote Address**: `192.168.1.0/24` (Head Office internal network)
6.  **Phase 2 Proposal**:
    *   **Encryption**: `AES256`
    *   **Authentication**: `SHA256`
    *   **Perfect Forward Secrecy (PFS)**: `Enable`
    *   **Diffie-Hellman Group**: `14`
    *   **Key Lifetime**: `3600` seconds
7.  Click **OK**.

#### Step 3: Create Firewall Addresses for Remote Network

1.  Navigate to **Policy & Objects > Addresses**.
2.  Click **Create New > Address**.
3.  **Name**: `Head_Office_LAN`
4.  **Type**: `Subnet`
5.  **Subnet/IP Range**: `192.168.1.0/24`
6.  **Interface**: `any`
7.  Click **OK**.

#### Step 4: Create Firewall Policies

1.  Navigate to **Policy & Objects > Firewall Policy**.
2.  Click **Create New**.
3.  **Name**: `Branch-LAN-to-HO-VPN`
4.  **Incoming Interface**: `port2` (Branch Office LAN interface)
5.  **Outgoing Interface**: `Branch-to-HO` (the IPsec tunnel interface)
6.  **Source**: `all` (or specific source addresses/groups from Branch LAN)
7.  **Destination**: `Head_Office_LAN`
8.  **Schedule**: `always`
9.  **Service**: `ALL`
10. **Action**: `ACCEPT`
11. **NAT**: `Disable`
12. **Log Allowed Traffic**: `All Sessions`
13. Click **OK**.

14. Click **Create New** for the return traffic.
15. **Name**: `HO-VPN-to-Branch-LAN`
16. **Incoming Interface**: `Branch-to-HO` (the IPsec tunnel interface)
17. **Outgoing Interface**: `port2` (Branch Office LAN interface)
18. **Source**: `Head_Office_LAN`
19. **Destination**: `all` (or specific destination addresses/groups in Branch LAN)
20. **Schedule**: `always`
21. **Service**: `ALL`
22. **Action**: `ACCEPT`
23. **NAT**: `Disable`
24. **Log Allowed Traffic**: `All Sessions`
25. Click **OK**.

#### Step 5: Configure Static Route

1.  Navigate to **Network > Static Routes**.
2.  Click **Create New**.
3.  **Destination**: `Subnet` `192.168.1.0/24`
4.  **Device**: `Branch-to-HO` (the IPsec tunnel interface)
5.  **Administrative Distance**: `10`
6.  Click **OK**.

## Verification

After configuring both FortiGate devices, the IPsec tunnel should come up automatically. You can verify the tunnel status:

*   **FortiGate A/B**: Navigate to **VPN > IPsec Tunnels**. The tunnel status should show as `Up`.
*   **FortiGate A/B**: Navigate to **Monitor > IPsec Monitor** to see detailed tunnel status and traffic statistics.

This concludes the FortiGate Site-to-Site IPsec VPN configuration guide.



## IPsec VPN Connectivity Test Results

To ensure the IPsec VPN tunnel is functioning correctly and securely, perform the following connectivity tests from both FortiGate A (Head Office) and FortiGate B (Branch Office).

### Test Scenario 1: Ping from Head Office LAN to Branch Office LAN

**Objective**: Verify that a host in the Head Office LAN can successfully ping a host in the Branch Office LAN, confirming bidirectional connectivity through the IPsec tunnel.

**Steps**:
1.  From a host in the Head Office LAN (e.g., `192.168.1.10`), attempt to ping a host in the Branch Office LAN (e.g., `192.168.2.10`).
    ```bash
    ping 192.168.2.10
    ```
2.  Observe the ping results.

**Expected Results**:
*   Successful ping replies from `192.168.2.10`.
*   No packet loss.
*   FortiGate A and B logs (**Log & Report > VPN Events** and **Traffic Logs**) should show traffic traversing the IPsec tunnel.

### Test Scenario 2: Ping from Branch Office LAN to Head Office LAN

**Objective**: Verify that a host in the Branch Office LAN can successfully ping a host in the Head Office LAN, confirming bidirectional connectivity through the IPsec tunnel.

**Steps**:
1.  From a host in the Branch Office LAN (e.g., `192.168.2.10`), attempt to ping a host in the Head Office LAN (e.g., `192.168.1.10`).
    ```bash
    ping 192.168.1.10
    ```
2.  Observe the ping results.

**Expected Results**:
*   Successful ping replies from `192.168.1.10`.
*   No packet loss.
*   FortiGate A and B logs should show traffic traversing the IPsec tunnel.

### Test Scenario 3: Access Internal Resources (e.g., Web Server, File Share)

**Objective**: Verify that hosts in one LAN can access specific services (e.g., HTTP, SMB) on hosts in the other LAN.

**Steps**:
1.  **From Head Office LAN to Branch Office**: From a host in the Head Office LAN, attempt to access a web server (`http://192.168.2.100`) or a file share (`\\192.168.2.10\share`) located in the Branch Office LAN.
2.  **From Branch Office LAN to Head Office**: From a host in the Branch Office LAN, attempt to access a web server (`http://192.168.1.100`) or a file share (`\\192.168.1.10\share`) located in the Head Office LAN.
3.  Observe the access results.

**Expected Results**:
*   Web servers and file shares should be accessible from the remote LAN.
*   FortiGate traffic logs should show successful connections for the respective services.

### Test Scenario 4: IPsec Tunnel Status Monitoring

**Objective**: Verify that the IPsec tunnel remains up and stable.

**Steps**:
1.  On both FortiGate A and FortiGate B, navigate to **VPN > IPsec Tunnels**.
2.  Check the status of the `HO-to-Branch` (on A) and `Branch-to-HO` (on B) tunnels.
3.  Navigate to **Monitor > IPsec Monitor** to view detailed tunnel information, including uptime, traffic statistics, and SA status.

**Expected Results**:
*   Tunnel status should consistently show as `Up`.
*   Traffic statistics should increment as data flows through the tunnel.
*   Phase 1 and Phase 2 Security Associations (SAs) should be established and active.

### Test Scenario 5: Tunnel Rekeying

**Objective**: Verify that the IPsec tunnel correctly rekeys Phase 1 and Phase 2 SAs after their lifetime expires without interrupting traffic.

**Steps**:
1.  Monitor the IPsec SAs on both FortiGate devices (**Monitor > IPsec Monitor**).
2.  Observe the `Key Lifetime` for Phase 1 and Phase 2 SAs.
3.  Initiate continuous traffic through the tunnel (e.g., a continuous ping or file transfer).
4.  Wait for the key lifetimes to expire and observe the rekeying process.

**Expected Results**:
*   New Phase 1 and Phase 2 SAs are established automatically before the old ones expire.
*   Traffic continues to flow uninterrupted during the rekeying process.
*   FortiGate logs show IKE rekeying events.

### Test Scenario 6: Dead Peer Detection (DPD) Test

**Objective**: Verify that the FortiGate detects an unresponsive peer and tears down the tunnel if DPD is enabled.

**Steps**:
1.  Ensure DPD is set to `Always On` for the Phase 1 configuration.
2.  Establish a stable IPsec tunnel.
3.  Simulate a failure on one FortiGate (e.g., temporarily disable its WAN interface or shut down the device).
4.  On the active FortiGate, monitor the IPsec tunnel status.

**Expected Results**:
*   The active FortiGate should detect the unresponsive peer and tear down the IPsec tunnel within the configured DPD interval.
*   The tunnel status should change to `Down`.
*   FortiGate logs should show DPD failure events and tunnel teardown.

These test scenarios provide a comprehensive approach to validating the functionality, security, and stability of the FortiGate IPsec VPN configuration.

