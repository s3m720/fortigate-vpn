# FortiGate SSL VPN Configuration Guide

## Introduction

This guide provides detailed instructions for configuring an SSL VPN on a FortiGate firewall. SSL VPN allows remote users to securely access internal network resources using a web browser or a FortiClient VPN client. This configuration will focus on establishing a full-tunnel SSL VPN for remote users, ensuring all their traffic is routed through the corporate network for enhanced security and policy enforcement.

## Prerequisites

Before you begin, ensure you have:

*   A FortiGate firewall with administrative access.
*   FortiOS version 7.4.3 or later (as per the referenced documentation).
*   An external (WAN) interface configured and connected to the internet.
*   A valid SSL certificate (either self-signed or from a trusted Certificate Authority) installed on the FortiGate. For production environments, a CA-signed certificate is highly recommended.
*   Defined user groups or local users for VPN access.
*   Knowledge of your internal network subnets and resources to be accessed.

## Configuration Steps

### Step 1: Enable SSL-VPN Realms Feature

By default, the SSL-VPN Realms feature might not be visible in the GUI. You need to enable it first.

1.  Navigate to **System > Feature Visibility**.
2.  Scroll down and enable the **SSL-VPN Realms** feature.
3.  Click **Apply**.

### Step 2: Configure SSL VPN Portal

The SSL VPN portal defines the user experience and access methods. We will use a full-access portal for tunnel mode.

1.  Navigate to **VPN > SSL-VPN Portals**.
2.  Edit the `full-access` portal (or create a new one if you prefer).
3.  Ensure that under **Tunnel Mode**, **Split Tunneling** is disabled if you want all traffic to go through the VPN (full tunnel). If you want only corporate traffic through the VPN, enable split tunneling and define the routing.
4.  Configure other settings as needed, such as enabling **Web Mode** if you want to provide web-based access to internal resources.
5.  Click **OK** to save.

### Step 3: Configure SSL VPN Realm

SSL VPN realms allow you to provide different portals and authentication methods based on the URL path used by the client.

1.  Navigate to **VPN > SSL-VPN Realms**.
2.  Click **Create New**.
3.  **Name**: Enter a descriptive name (e.g., `remote_users_realm`).
4.  **URL Path**: Enter a unique URL path (e.g., `/remote`). This path will be appended to your FortiGate's public IP/hostname (e.g., `https://your.fortigate.ip:10443/remote`).
5.  **Portal**: Select the `full-access` portal configured in Step 2.
6.  Click **OK** to save.

### Step 4: Configure SSL VPN Settings

This section defines the core SSL VPN parameters, including the listening interface, port, and server certificate.

1.  Navigate to **VPN > SSL-VPN Settings**.
2.  **Enable SSL-VPN**: Ensure this is enabled.
3.  **Listen on Interface(s)**: Select the external (WAN) interface(s) where the FortiGate will listen for SSL VPN connections (e.g., `wan1`).
4.  **Listen on Port**: Change the default port (443) to a non-standard port (e.g., `10443`) for security reasons.
5.  **Server Certificate**: Select the appropriate SSL certificate installed on your FortiGate. For testing, you can use the `Fortinet_Factory` certificate, but for production, use a CA-signed certificate.
6.  **IP Ranges**: Define the IP address range that will be assigned to remote VPN clients. Ensure this range does not overlap with your internal network subnets.
7.  **DNS Server**: Specify your internal DNS servers so VPN clients can resolve internal hostnames.
8.  **Authentication/Portal Mapping**: This is where you map user groups to specific portals and realms.
    *   Click **Create New**.
    *   **Users/Groups**: Select the user group(s) that will have access to this SSL VPN (e.g., `VPN_Users`).
    *   **Realm**: Select `Specify` and choose the realm created in Step 3 (e.g., `/remote`).
    *   **Portal**: Select the `full-access` portal.
    *   Click **OK**.
    *   Edit the **All Other Users/Groups** entry and set its portal to `no-access` to restrict unauthorized access.
9.  Click **Apply** to save the SSL VPN settings.

### Step 5: Configure Firewall Policy

You need a firewall policy to allow authenticated SSL VPN users to access internal network resources.

1.  Navigate to **Policy & Objects > Firewall Policy**.
2.  Click **Create New**.
3.  **Name**: Enter a descriptive name (e.g., `SSL_VPN_to_Internal`).
4.  **Incoming Interface**: Select `ssl.root` (this is the virtual interface for SSL VPN tunnel traffic).
5.  **Outgoing Interface**: Select the internal interface(s) that connect to the resources VPN users need to access (e.g., `port2` or your `LAN` interface).
6.  **Source**: Select the user group configured for SSL VPN access (e.g., `VPN_Users`) and the `SSLVPN_TUNNEL_ADDR1` address object (which represents the IP range assigned to VPN clients).
7.  **Destination**: Define the internal network subnets or specific address objects that VPN users are allowed to reach.
8.  **Schedule**: `always`.
9.  **Service**: `ALL` (or specific services if you want to restrict access further).
10. **Action**: `ACCEPT`.
11. **NAT**: Ensure **NAT** is disabled if you want the original source IP of the VPN client to be preserved when accessing internal resources. If you want the traffic to appear from the FortiGate's internal interface, enable NAT.
12. **Log Allowed Traffic**: Enable logging for `All Sessions` for monitoring and troubleshooting.
13. Configure any other security profiles (e.g., Antivirus, Web Filter, Application Control) as per your security requirements.
14. Click **OK** to save the policy.

## Testing SSL VPN Connectivity

To test the SSL VPN connection:

1.  **Download FortiClient**: Instruct remote users to download and install the FortiClient VPN client from the Fortinet support website.
2.  **Configure FortiClient**: In FortiClient, create a new VPN connection:
    *   **VPN Type**: `SSL-VPN`.
    *   **Remote Gateway**: Enter the public IP address or hostname of your FortiGate, followed by the configured port and realm path (e.g., `your.fortigate.ip:10443/remote`).
    *   **Authentication**: Enter the username and password of an authorized VPN user.
3.  **Connect**: Initiate the connection from FortiClient.
4.  **Verify Connectivity**: Once connected, verify that you can access internal network resources (e.g., ping internal servers, access internal web applications).
5.  **Verify Traffic Routing**: If full tunneling is configured, verify that your public IP address (as seen by external websites) is that of your FortiGate's WAN interface.
6.  **Check FortiGate Logs**: Monitor the FortiGate logs (**Log & Report > VPN Events**) to confirm successful VPN connections and traffic flow.

## Best Practices for SSL VPN

*   **Strong Authentication**: Implement Multi-Factor Authentication (MFA) for all VPN users.
*   **Least Privilege**: Grant VPN users access only to the resources they absolutely need.
*   **Regular Updates**: Keep FortiGate firmware and FortiClient software up to date.
*   **Logging and Monitoring**: Continuously monitor VPN logs for suspicious activity.
*   **Security Profiles**: Apply appropriate security profiles (Antivirus, IPS, Web Filter) to SSL VPN traffic.
*   **Non-Standard Ports**: Use a non-standard port for SSL VPN to reduce automated attacks.
*   **Certificates**: Use CA-signed certificates for production environments.

This concludes the FortiGate SSL VPN configuration guide. Following these steps will enable secure remote access for your users.



## SSL VPN Test Scenarios

To ensure the SSL VPN is functioning correctly and securely, perform the following test scenarios:

### Scenario 1: Successful Connection and Internal Resource Access (Full Tunnel)

**Objective**: Verify that a legitimate remote user can successfully establish a full-tunnel SSL VPN connection and access internal network resources.

**Steps**:
1.  From a remote location, launch FortiClient and attempt to connect to the configured SSL VPN gateway (e.g., `your.fortigate.ip:10443/remote`).
2.  Enter valid credentials for a user belonging to the `VPN_Users` group.
3.  Once connected, verify that the FortiClient status indicates a successful connection.
4.  **Test Internal Access**: Attempt to ping an internal server (e.g., `ping 192.168.1.100`).
5.  **Test Internal Application Access**: Attempt to access an internal web application (e.g., `http://internal-app.yourdomain.local`).
6.  **Test Internet Access (Full Tunnel)**: Browse to a public website (e.g., `whatismyip.com`) and verify that the public IP address shown is the FortiGate's external WAN IP, confirming full tunneling is active.
7.  **Verify FortiGate Logs**: Check the FortiGate logs (**Log & Report > VPN Events** and **Traffic Logs**) to confirm the successful connection and traffic flow from the VPN client to internal resources and the internet.

**Expected Results**:
*   FortiClient connects successfully.
*   Internal servers are reachable via ping.
*   Internal web applications are accessible.
*   Public IP address reflects the FortiGate's WAN IP.
*   FortiGate logs show successful VPN authentication and traffic matching the configured firewall policy.

### Scenario 2: Unsuccessful Connection - Invalid Credentials

**Objective**: Verify that users with invalid credentials are denied access.

**Steps**:
1.  From a remote location, launch FortiClient and attempt to connect to the SSL VPN gateway.
2.  Enter an incorrect username or password.

**Expected Results**:
*   FortiClient displays an authentication failure message.
*   FortiGate logs show authentication failure events.

### Scenario 3: Unsuccessful Connection - Unauthorized User

**Objective**: Verify that users not part of the `VPN_Users` group are denied access.

**Steps**:
1.  Attempt to connect to the SSL VPN gateway using credentials for a user account that is *not* a member of the `VPN_Users` group.

**Expected Results**:
*   FortiClient displays an authentication or authorization failure message.
*   FortiGate logs show access denied events, possibly due to portal mapping restrictions.

### Scenario 4: Access to Restricted Internal Resources

**Objective**: Verify that VPN users can only access resources permitted by the firewall policy.

**Steps**:
1.  Establish a successful SSL VPN connection with a user from the `VPN_Users` group.
2.  Attempt to access an internal resource that is *not* included in the destination address objects of the `SSL_VPN_to_Internal` firewall policy (e.g., `ping 192.168.2.100` if `192.168.2.0/24` is not allowed).

**Expected Results**:
*   Access to the restricted resource fails (e.g., ping timeout, connection refused).
*   FortiGate traffic logs show `deny` actions for attempts to access unauthorized resources.

### Scenario 5: SSL VPN Disconnection and Reconnection

**Objective**: Verify stable connection and proper session termination.

**Steps**:
1.  Establish a successful SSL VPN connection.
2.  Disconnect the VPN connection from FortiClient.
3.  Wait a few moments and then attempt to reconnect.

**Expected Results**:
*   The VPN disconnects cleanly.
*   The VPN reconnects successfully.
*   FortiGate logs show disconnection and subsequent reconnection events.

These test scenarios cover basic functionality, security, and policy enforcement for the FortiGate SSL VPN configuration.

