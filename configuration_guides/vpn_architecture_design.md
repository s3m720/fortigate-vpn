# VPN Architecture and Configuration Plans

## 1. SSL VPN Architecture Design

### Overview

SSL VPN provides secure remote access to internal network resources for individual users. It is particularly useful for employees working remotely, allowing them to access corporate applications and data securely over the internet using a standard web browser or a dedicated client. FortiGate's SSL VPN implementation supports both web-mode and tunnel-mode connections, offering flexibility based on user requirements and security policies.

### Key Components

*   **FortiGate Firewall**: Acts as the SSL VPN gateway, terminating encrypted connections from remote users.
*   **Remote Users**: Individuals connecting from external networks (e.g., home, public Wi-Fi) using a FortiClient VPN client or a web browser.
*   **Internal Network Resources**: Servers, applications, and data within the corporate network that remote users need to access.
*   **Authentication Server**: (Optional but recommended) An external server like LDAP, RADIUS, or FortiAuthenticator for centralized user authentication. Local FortiGate user database can also be used.

### Connection Flow

1.  **Initiation**: A remote user initiates an SSL VPN connection to the FortiGate's public IP address on a specified port (default 10443).
2.  **Authentication**: The FortiGate authenticates the user against its local database or an external authentication server. Multi-factor authentication (MFA) is highly recommended for enhanced security.
3.  **Portal/Tunnel Assignment**: Upon successful authentication, the user is assigned to an SSL VPN portal (for web-mode access) or a tunnel (for full network access via FortiClient).
    *   **Web-mode**: Provides access to specific internal web applications, file shares, or network services through a web browser interface.
    *   **Tunnel-mode**: Establishes a full network layer tunnel, allowing the remote user's device to become a virtual member of the internal network, accessing resources as if directly connected.
4.  **Policy Enforcement**: Firewall policies on the FortiGate control what resources the authenticated SSL VPN users can access within the internal network.
5.  **Traffic Encryption**: All traffic between the remote user and the FortiGate is encrypted using SSL/TLS protocols, ensuring data confidentiality and integrity.

### Configuration Considerations

*   **Interface Selection**: The FortiGate interface listening for SSL VPN connections should typically be an external (WAN) interface.
*   **Port Configuration**: Use a non-standard port (other than 443) for SSL VPN to reduce exposure to common port scans.
*   **Server Certificate**: A valid SSL certificate (either self-signed or from a trusted CA) is crucial for secure communication and to prevent browser warnings.
*   **User Groups and Realms**: Organize users into groups and assign them to specific SSL VPN realms and portals to enforce granular access control.
*   **Split Tunneling**: Decide whether to enable split tunneling. When enabled, only traffic destined for the corporate network goes through the VPN tunnel, while internet traffic goes directly. This reduces load on the FortiGate but might bypass corporate internet security policies. Full tunneling routes all traffic through the VPN.
*   **Firewall Policies**: Create explicit firewall policies to allow authenticated SSL VPN users to access necessary internal resources. Ensure NAT is handled correctly.
*   **Logging and Monitoring**: Enable comprehensive logging for SSL VPN sessions to monitor user activity and troubleshoot issues.

## 2. IPsec VPN Architecture Design

### Overview

IPsec VPN is primarily used for site-to-site connectivity, creating secure tunnels between two networks (e.g., headquarters and a branch office) or for remote access where a dedicated client establishes a secure connection to a central gateway. It operates at the network layer, encrypting and authenticating IP packets. FortiGate devices are highly capable of establishing robust IPsec VPN tunnels, supporting various encryption algorithms, authentication methods, and key exchange protocols.

### Key Components

*   **FortiGate Firewalls (or other IPsec-compliant devices)**: Act as VPN gateways at each end of the tunnel, responsible for encrypting/decrypting traffic and authenticating peers.
*   **Local Networks**: The internal networks at each site that need to communicate securely.
*   **Public Network (Internet)**: The untrusted medium over which the encrypted tunnel is established.

### Connection Flow (Site-to-Site)

1.  **Initiation**: Traffic from a host in one local network destined for a host in the remote local network triggers the IPsec tunnel establishment.
2.  **Phase 1 (IKE - Internet Key Exchange)**: The two FortiGate devices establish a secure, authenticated channel (IKE SA - Security Association) between themselves. This involves:
    *   **Negotiation**: Agreeing on encryption algorithms (e.g., AES256), hashing algorithms (e.g., SHA256), Diffie-Hellman group for key exchange, and authentication method (pre-shared key or certificates).
    *   **Authentication**: Verifying the identity of the peer FortiGate.
3.  **Phase 2 (IPsec SA)**: Once Phase 1 is complete, the FortiGate devices negotiate the parameters for the actual data tunnel (IPsec SA). This includes:
    *   **Negotiation**: Agreeing on IPsec protocols (ESP or AH), encryption algorithms, hashing algorithms, and Perfect Forward Secrecy (PFS) settings.
    *   **Key Exchange**: Generating session keys for encrypting and decrypting data traffic.
4.  **Data Transfer**: Encrypted data traffic flows securely between the two local networks through the established IPsec tunnel.
5.  **Policy Enforcement**: Firewall policies on both FortiGate devices control which traffic is allowed to traverse the IPsec tunnel.

### Configuration Considerations

*   **Phase 1 (IKE Gateway)**:
    *   **Remote Gateway**: Static IP address or dynamic DNS.
    *   **Interface**: The external (WAN) interface used for the VPN connection.
    *   **Authentication Method**: Pre-shared key (PSK) or digital certificates. PSK is simpler for small deployments, while certificates offer stronger security and scalability.
    *   **Encryption and Authentication**: Choose strong algorithms (e.g., AES256, SHA256).
    *   **Diffie-Hellman Group**: Select a strong group (e.g., Group 14 or higher).
    *   **Key Lifetime**: Define how long the Phase 1 SA remains valid.
*   **Phase 2 (IPsec Tunnel)**:
    *   **Local and Remote Subnets**: Define the networks that will communicate over the tunnel.
    *   **Encryption and Authentication**: Match Phase 1 settings or use different, equally strong algorithms.
    *   **Perfect Forward Secrecy (PFS)**: Enable PFS to ensure that if a session key is compromised, past and future session keys remain secure.
    *   **Key Lifetime**: Define how long the Phase 2 SA remains valid.
*   **Firewall Policies**: Create specific policies on both FortiGate devices to permit traffic between the local and remote subnets over the IPsec tunnel. Ensure NAT is disabled for VPN traffic if direct routing is desired.
*   **Routing**: Ensure proper static routes or dynamic routing protocols (e.g., OSPF, BGP) are configured to direct traffic into the IPsec tunnel.
*   **Dead Peer Detection (DPD)**: Enable DPD to detect unresponsive peers and tear down stale tunnels.

## 3. FortiGate VPN with SD-WAN Integration Design

### Overview

Integrating VPNs (especially IPsec VPNs) with SD-WAN on FortiGate allows organizations to optimize traffic flow, enhance performance, and provide redundancy for VPN connections. SD-WAN intelligently directs traffic over the best available link based on performance metrics (latency, jitter, packet loss) and defined policies. When combined with VPNs, it ensures that secure traffic leverages the most efficient path, improving user experience and application performance, particularly for branch offices or cloud connectivity.

### Key Components

*   **FortiGate Devices**: Acting as SD-WAN hubs and spokes, terminating VPN tunnels and performing intelligent path selection.
*   **Multiple WAN Links**: Diverse internet connections (e.g., MPLS, broadband, LTE) at each site, forming the SD-WAN fabric.
*   **IPsec VPN Tunnels**: Established over the various WAN links to provide secure connectivity between sites.
*   **SD-WAN Zone**: A logical grouping of WAN interfaces (including VPN interfaces) that the FortiGate uses for intelligent traffic steering.
*   **Performance SLAs (Service Level Agreements)**: Defined metrics (latency, jitter, packet loss) used by SD-WAN to evaluate link quality.
*   **SD-WAN Rules**: Policies that dictate how traffic is steered across the available WAN links based on application, destination, and performance requirements.

### Integration Flow

1.  **VPN Tunnel Creation**: Multiple IPsec VPN tunnels are established between FortiGate devices, typically one over each available WAN link. These tunnels provide the secure overlay network.
2.  **SD-WAN Zone Configuration**: The VPN tunnel interfaces are added as members to an SD-WAN zone. This allows the SD-WAN engine to manage and monitor these secure paths.
3.  **Performance SLA Monitoring**: Performance SLAs are configured to monitor the quality of each VPN tunnel. This involves sending probes (e.g., ICMP, HTTP) through the tunnels to measure latency, jitter, and packet loss to a defined target (e.g., a server in the remote network).
4.  **SD-WAN Rules for VPN Traffic**: SD-WAN rules are created to steer specific VPN traffic (e.g., critical business applications) over the best-performing VPN tunnel based on the real-time SLA metrics. Strategies like 


Lowest Cost SLA, Best Quality, or Maximum Bandwidth can be used.
5.  **Dynamic Path Selection**: Based on the SD-WAN rules and real-time performance data from the SLAs, the FortiGate dynamically selects the optimal VPN tunnel for each traffic flow, ensuring high availability and optimal performance.
6.  **Redundancy and Failover**: If a primary VPN tunnel or its underlying WAN link fails or degrades below a defined threshold, SD-WAN automatically steers traffic to an alternative healthy VPN tunnel, providing seamless failover.

### Configuration Considerations

*   **Manual VPN Interface Creation**: For SD-WAN integration, it is often recommended to create IPsec VPN interfaces manually rather than using the wizard, as wizard-created interfaces might not be directly usable as SD-WAN members.
*   **SD-WAN Zone**: Create a dedicated SD-WAN zone and add the IPsec VPN tunnel interfaces as members. This allows the SD-WAN engine to manage these secure paths.
*   **Performance SLA**: Configure Performance SLAs to monitor the health and performance of each VPN tunnel. Define targets (e.g., internal server IPs) and thresholds for latency, jitter, and packet loss.
*   **SD-WAN Rules**: Create granular SD-WAN rules to direct traffic. These rules can be based on source/destination IP, application, service, and leverage the Performance SLA metrics to choose the best path. Ensure VPN-related SD-WAN rules are prioritized correctly.
*   **Firewall Policies**: Configure firewall policies to allow traffic through the SD-WAN zone and across the VPN tunnels. Ensure NAT is disabled for VPN traffic within the SD-WAN context if direct routing is intended.
*   **Blackhole Routes**: Implement blackhole routes for destination subnets to prevent traffic from being routed over the internet if all VPN tunnels to a specific destination are down. This ensures traffic is dropped rather than sent unencrypted or to an incorrect destination.
*   **CLI for Advanced Configuration**: Some advanced configurations, such as adding source IPs to SD-WAN members for Performance SLA to function correctly, might require CLI commands.

This integrated approach ensures that VPN connectivity is not only secure but also highly optimized, resilient, and performs efficiently across diverse WAN infrastructures.


