# VPN Types and Use Cases

## From Palo Alto Networks: What Are the Different Types of VPN?

| VPN Type | Connection Type | Software Type | Use Cases |
| --- | --- | --- | --- |
| Remote Access VPN | User connects to a private network | Software installed on both a private device and the private network | Connecting to a private network from your home or personal location |
| Site-to-Site VPN | Private network connects to another private network | Software on both networks, users do not need apps | Creating a secure tunnel between two private networks |
| SSL VPN | Devices establish a secure remote access VPN connection with a web browser | Modern web browser or client applications that enable direct access to networks. There are two primary types: VPN portal and VPN tunnel | Enterprises use SSL VPNs to enable remote users to securely access organizational resources and to secure the internet sessions of users |
| Cloud Based Remote Access VPN | User connects to a company's applications, data, and files in the cloud | Accessible via a website or desktop/mobile application; No need for VPN infrastructure on user's end | Secure access to both the company’s cloud and data center-based applications and data |
| Double VPN | Use of two VPNs simultaneously, though practically challenging | Specialized software configuration or utility, but often results in conflicts | Enhanced security through multiple VPN layers, although challenging to implement properly |




## FortiGate SSL VPN Configuration

### SSL VPN Portal

*   Use default full-access or tunnel-access profile.
*   Ensure split tunneling is configured and enabled based on policy destination.

### SSL VPN Realm

1.  Go to _System > Feature Visibility_.
2.  Enable _SSL-VPN Realms_.
3.  Click _Apply_.
4.  Under _VPN > SSL-VPN Realms_, click _Create New_.
5.  Enter the URL path (e.g., `pki-ldap-machine`).
6.  Click _OK_ to save.

### SSL VPN Settings

1.  Go to _System > SSL-VPN Settings_.
2.  Configure the following:
    *   **Enable SSL-VPN**: Enable
    *   **Listen on Interface(s)**: `port3` (or appropriate interface)
    *   **Listen on Port**: `10443`
    *   **Server Certificate**: `ztna-wildcard` (or appropriate certificate)
    *   **DNS Server**: Specify `10.88.0.1` (or appropriate DNS server)
3.  Under _Authentication/Portal Mapping_, click _Create New_.
4.  Set _Users/Groups_ to `PKI-Machine-Group` (or appropriate user group).
5.  Set _Realm_ to _Specify_ and select the configured realm (e.g., `/pki-ldap-machine`).
6.  Set the portal to `full-access`.
7.  Click _OK_ to save.
8.  Edit the _All Other Users/Groups_ entry:
    *   Set portal to `no-access`.
    *   Click _OK_ to save.

### Firewall Policy

1.  From _Policy & Objects > Firewall Policy_, click _Create New_.
2.  Configure the following:
    *   **Name**: `VPN-Machine` (or appropriate name)
    *   **Incoming Interface**: `SSL-VPN tunnel interface (ssl.root)`
    *   **Outgoing Interface**: `port2` (or appropriate interface)
    *   **Source**: `all`, `PKI-Machine-Group` (or appropriate source)
    *   **Destination**: Create an address object for the web server (e.g., `10.88.0.3/32`) and any other servers that must be accessed.
    *   **Schedule**: `always`
    *   **Service**: `ALL`
    *   **Action**: `ACCEPT`
    *   **Log Allow Traffic**: Enabled, All Sessions
3.  Configure any other security profiles settings as needed.
4.  Click _OK_ to save.





## IPsec VPN

Virtual Private Network (VPN) technology lets remote users connect to private computer networks to gain access to their resources in a secure way. For example, an employee traveling or working at home can use a VPN to securely access the office network through the internet.

Instead of remotely logging into a private network using an unencrypted and unsecured internet connection, using a VPN ensures that unauthorized parties cannot access the office network and cannot intercept information going between the employee and the office. Another common use of a VPN is to connect the private networks of multiple offices.

IPsec VPN uses the Internet Protocol Security (IPsec) protocol to create encrypted tunnels on the internet. The IPsec protocol operates at the network layer of the OS model and runs on top of the IP protocol, which routes packets. All transmitted data is protected by the IPsec tunnel.

The Fortinet documentation provides instructions on configuring IPsec VPN connections in FortiOS 7.6.4, covering:

*   General IPsec VPN configuration
*   Site-to-site VPN
*   Remote access
*   Aggregate and redundant VPN
*   ADVPN
*   Fabric Overlay Orchestrator
*   Other VPN topics
*   VPN IPsec troubleshooting




## FortiGate SD-WAN and IPsec VPN Integration

This article describes the integration of IPsec VPN with SD-WAN to manage IPsec traffic flow and redundancy using SD-WAN rules.

### Solution Overview

To manage IPsec VPN with SD-WAN instead of relying on route priority, follow these steps:

1.  **Configure VPN Interface Manually**: Do not use the IPsec Wizard for creating the VPN interface if it's intended for SD-WAN members, as interfaces created via the wizard cannot be directly called in SD-WAN members. If a wizard was used, existing configuration references must be removed or redirected.

2.  **Create SD-WAN Zone**: Navigate to `Network -> SD-WAN`, then select `Create New -> SDWAN Zone`. Give it a name (e.g., "VPN"). Do not add members at this stage.

3.  **Create SD-WAN Member**: Navigate to `Network -> SD-WAN`, then select `Create New -> SDWAN Member`.
    *   Select `+VPN` in the Interface drop-down to open the `Create IPsec VPN for SD-WAN members` pane.
    *   Enter the required information:
        *   **Name**: Appropriate Tunnel Name (e.g., `VPN_1`)
        *   **Remote Device IP address/DDNS**: The IP address of the remote device.
        *   **Outgoing Interface**: The WAN interface (e.g., `port3`).
        *   **Authentication Mode**: Pre-Shared Key/Signature (e.g., Pre-Shared Key).
        *   **Pre-Shared Key**: Define a pre-shared key.
    *   Review settings and create the interface.

4.  **Add Interface to Zone**: Add the newly created VPN interface to the respective SD-WAN zone (e.g., "VPN"). Create a second VPN tunnel following the same process and add it to the same zone for redundancy.

5.  **Change VPN Traffic Selector**: Adjust the VPN traffic selector as required. The SD-WAN Wizard typically creates `any` and `any` selectors. Configure specific local and remote addresses (e.g., local - `10.24.0.0/20` and remote `10.25.0.0/20`). If the IPsec connection should not go to the Internet, modify the phase 2 selector under `VPN -> Ipsec Tunnel -> Respected Tunnel`.

6.  **Assign IP Address to VPN Interface**: Go to `Network -> Interface` and edit the VPN interface (e.g., `VPN_1`). Define local and remote interface IPs (e.g., `1.1.1.1` and `1.1.1.2` for `VPN_1` and `VPN_2` respectively, with remote IPs `2.2.2.1` and `2.2.2.2`).
    *   **Note**: Use random IP addresses that do not conflict with existing subnets. Subsequent IP addresses are ideal for easy identification.

7.  **Create Static Route for VPN Traffic**: If running FortiOS v7.0 and above, create a static route for VPN traffic using the VPN SD-WAN zone. For v6.4.x, static routes can be created for individual VPN interfaces or the entire SD-WAN interface, but not for individual VPN SD-WAN zones.
    *   Go to `Network -> Static Route`, select `Create New`, enter the required information, and click `OK`.

8.  **Create Firewall Policy for VPN Traffic**: Go to `Policy & Object -> Firewall Policy`, select `Create New`. Select the appropriate interface for LAN to VPN communication and add the required attributes.
    *   **Note**: Ensure NAT is disabled in the policy to prevent traffic from using the tunnel interface IP, which could lead to dropped traffic at the peer end if no route exists.

9.  **Configure Performance SLA for VPN Interface in SD-WAN**: Go to `SDWAN -> Performance SLA`, select `Create New`, and define parameters (e.g., using a peer-end Local LAN machine IP like `10.25.12.3`).
    *   To get the SLA working, add the source in the SD-WAN VPN member from CLI:
        ```cli
        config system sdwan
            config members
                edit 4
                    set interface "VPN_2"
                    set zone "VPN"
                    set source 10.24.3.109 <----- Added LAN interface IP.
                next
                edit 5
                    set interface "VPN_1"
                    set zone "VPN"
                    set source 10.24.3.109 <----- Added LAN interface IP.
                next
        end
        ```
    *   **Note 1**: Source IP must be an interface IP and included in the VPN phase 2 traffic selector.
    *   **Note 2**: If the peer-end configuration is incomplete, the SLA will not come up.

10. **Create SD-WAN Rule**: Go to `Network -> SD-WAN Rules`, select `Create New`, and define parameters (e.g., `Lowest cost SLA` strategy). Ensure the VPN rule is above the `all to all` rule.

11. **Create Blackhole Route**: Create a blackhole route for destination subnets to prevent traffic from being routed to the ISP link if both tunnels are down. Use CLI to configure it, ensuring the AD value is higher than the static route.
    ```cli
    config router static
        edit 4
            set dst 10.25.0.0 255.255.240.0
            set distance 15
            set blackhole enable
    end
    ```



