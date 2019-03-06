## Intro
This Ansible role will create SoftEther VPN server and configure a SSL-VPN service running on port TCP 443.

## SoftEther and SSL-VPN
- SoftEther supports unlimited VPN sessions and various of VPN protocols including:
    - <b>L2TP over IPSec Most:</b> for portable device such as mobile phone, tablet and native MacOSX VPN client application.
    - <b>OpenVPN and SSTP VPN:</b> enable SSL-VPN on port 443 similar to what OpenVPN Server and Microsoft SSTP VPN Server do. That capability gives the VPN service better firewall penetration, since TCP/UDP on 443 is usually accessible on most ISP/Firewall.

## Environment
- Ubuntu 16.04.5 LTS (Xenial Xerus)

## Installation
- Following tutorial to install SoftEther (ver: v4.28-9669-beta-2018.09.11) in a Ubuntu 16.04 LTS server and add systemctl service with auto start enabled.
- Run the ansible playbook to install and config your server.
- Refer to [Setting up SoftEther VPN Server on Ubuntu 16.04 Xenial Xerus Linux](https://linuxconfig.org/setting-up-softether-vpn-server-on-ubuntu-16-04-xenial-xerus-linux) for detail explaining. In brief, it downloads SoftEther server source, make install, configure as a systemd daemon, start it.

## Verification
- Check installation is success via:
```
# cd /usr/local/vpnserver
# ./vpncmd
Select 3 to choose Use of VPN Tools and then type:
VPN Tools> check
```

## Configuration
### 1. Setup Admin Password
```
./vpncmd
Press 1 to select "Management of VPN Server or VPN Bridge"
Then press Enter without typing anything to connect to the localhost server
Then press Enter without inputting anything to connect to server by server admin mode. You will see:
VPN Server>_
```
Then use command below to change admin password:
```
VPN Server> ServerPasswordSet
```

### 2. Start the VPN server for configuration
```
# ./vpnserver start
or
# systemctl restart sslvpn.service
```


### 3. Create A Virtual Hub
```
VPN Server> HubCreate [VIRTUAL_HUB_NAME]
```
you will be asked to enter an administrator password for the new virtual hub

### 4. Enable SecureNAT in a Virtual Hub
- The SecureNAT function virtualizes the NAT and DHCP server features. 
- The SecureNAT function can be set and enabled/disabled for each Virtual Hub. Obviously, it is the easiest manner to setup NAT and DHCP for VPN clients, which will gives default NAT Gateway IP and default DHCP IP-Range. To enable it, just run against a Virtual Hub
    ```
    VPN Server/[VIRTUAL_HUB_NAME]> SecureNatEnable 
    ```
    

- However, you could customize NAT Gateway IP and DHCP IP-Range via:
    ```
    VPN Server/[VIRTUAL_HUB_NAME]> SecureNatHostSet
    VPN Server/[VIRTUAL_HUB_NAME]> DhcpSet
    ```
    For example with `SecureNatHostSet` you can setup NAT Gateway:
    ```
    Item       |Value
    -----------+-----------------
    MAC Address|AA-BB-CC-DD-EE-FF
    IP Address |10.0.x.1
    Subnet Mask|255.255.255.0
    The command completed successfully.
    ```
    With `DhcpSet`, you can setup DHCP rules as:
    ```
    Item                           |Value
    -------------------------------+-------------
    Use Virtual DHCP Function      |Yes
    Start Distribution Address Band|10.0.x.3
    End Distribution Address Band  |10.0.x.100
    Subnet Mask                    |255.255.255.0
    Lease Limit (Seconds)          |7200
    Default Gateway Address        |10.0.x.1
    DNS Server Address 1           |10.0.x.1
    DNS Server Address 2           |None
    Domain Name                    |
    Save NAT and DHCP Operation Log|Yes
    Static Routing Table to Push   |
    The command completed successfully.
    ```
    Replace the `10.0.x.1` value for your circumstance.

- You might need to reboot your `sslvpn.service` for above changes
- Also, you could use [VPN Server Manager GUI tool](https://www.softether.org/4-docs/1-manual/2._SoftEther_VPN_Essential_Architecture/2.4_VPN_Server_Manager) to setup NAT Gateway and DHCP

### 5. Create VPN users
- Create new user for a virtualhub:

    ```
    VPN Server/[VIRTUAL_HUB_NAME]> UserCreate [YOUR_USER_NAME]
    VPN Server/[VIRTUAL_HUB_NAME]> UserPasswordSet [YOUR_USER_NAME]
    ```

### 6. Enable OpenVPN function
- SoftEther can replace OpenVPN Server. It has the OpenVPN Server Clone Function so that any OpenVPN clients, including iPhone and Android, can connect to SoftEther VPN easily.
- For enable the OpenVPN Clone Function, run
    ```
    # ./vpncmd
    Press 1 to select "Management of VPN Server or VPN Bridge"
    Then press Enter without typing anything to connect to the localhost server
    Then press Enter without inputting anything to connect to server by server admin mode. You will see:
    VPN Server>_

    VPN Server>OpenVpnEnable
    OpenVpnEnable command - Enable / Disable OpenVPN Clone Server Function
    Enables OpenVPN Clone Server Function (yes / no): yes
    UDP Ports to Listen for OpenVPN (Default: 1194 / Multiple Accepted): 443
    The command completed successfully.
    ```

### 7. Self-signed SSL certificate
- For SSL-VPN running on port 443, a SSL certificate will be needed for your server. Use following command to generate and register a Self-signed SSL certificate.
    ```
    VPN Server> ServerCertRegenerate [YOUR_HOST_COMMON_NAME_OR_IP_ADDRESS]
    ```

- The certificate can be download and added as trusted in VPN client.

### 8. Download configuration for OpenVPN Client
- Using configuration file to setup OpenVPN client is easier than manually
    ```
    VPN Server/[VIRTUAL_HUB_NAME]> OpenVpnMakeConfig ~/my_openvpn_client_config.zip
    ```
Unzip it you will find `x_openvpn_remote_access_l3.ovpn` file. Review this ovpn file and update: 
- Replacing the dynamic generated `x.y.softether.net` domain.
    ```
    remote [YOUR_VPN_YOUR_HOST_COMMON_NAME_OR_IP_ADDRESS] 443
    ```
- Update the `proto udp` to `probe tcp` since "[Any TCP ports which are defined as listeners on the VPN server accepts OpenVPN Protocol respectively and equally](https://www.vpnusers.com/viewtopic.php?t=5638)"
    ```
    proto tcp
    ```


## OpenVPN Client in MacOSX
- Download and install `OpenVPN Connect Client` for MacOSX from: [Connecting to Access Server with macOS](https://openvpn.net/vpn-server-resources/connecting-to-access-server-with-macos/)
- Click task bar icon `Import/From local file...` to import your `x_openvpn_remote_access_l3.ovpn` given by VPN server administrator
- Connect to the new VPN, input your username as <b>`"[USERNAME]@[VIRTUAL_HUB_NAME]"`</b> and password
- Accept the 'Untrusted Certificate warning' if you use a self-signed ceritificate
- You shall able to connect to the VPN immediately


## TODO
- Request CA sign it the certificate
- Generate a formal certificate and certificate request via OpenSSL
- Install CA signed SSL certificate to SSL-VPN server 
- Automate above manual steps


### References
- [https://linuxconfig.org/setting-up-softether-vpn-server-on-ubuntu-16-04-xenial-xerus-linux](https://linuxconfig.org/setting-up-softether-vpn-server-on-ubuntu-16-04-xenial-xerus-linux)
- [https://www.digitalocean.com/community/tutorials/how-to-setup-a-multi-protocol-vpn-server-using-softether](https://www.digitalocean.com/community/tutorials/how-to-setup-a-multi-protocol-vpn-server-using-softether)
- About SecureNAT: [https://www.softether.org/index.php?title=4-docs/1-manual/3._SoftEther_VPN_Server_Manual/3.7_Virtual_NAT_%26_Virtual_DHCP_Servers](https://www.softether.org/index.php?title=4-docs/1-manual/3._SoftEther_VPN_Server_Manual/3.7_Virtual_NAT_%26_Virtual_DHCP_Servers)
- VPN Server Manager: [https://www.softether.org/4-docs/1-manual/2._SoftEther_VPN_Essential_Architecture/2.4_VPN_Server_Manager](https://www.softether.org/4-docs/1-manual/2._SoftEther_VPN_Essential_Architecture/2.4_VPN_Server_Manager)
- Replacements of OpenVPN: [https://www.softether.org/4-docs/2-howto/7.Replacements_of_Legacy_VPNs/2.Replacements_of_OpenVPN](https://www.softether.org/4-docs/2-howto/7.Replacements_of_Legacy_VPNs/2.Replacements_of_OpenVPN)
- [https://www.softether.org/4-docs/1-manual/6._Command_Line_Management_Utility_Manual/6.3_VPN_Server_%2F%2F_VPN_Bridge_Management_Command_Reference_(For_Entire_Server)#6.3.75_.22OpenVpnGet.22:_Get_the_Current_Settings_of_OpenVPN_Clone_Server_Function](https://www.softether.org/4-docs/1-manual/6._Command_Line_Management_Utility_Manual/6.3_VPN_Server_%2F%2F_VPN_Bridge_Management_Command_Reference_(For_Entire_Server)#6.3.75_.22OpenVpnGet.22:_Get_the_Current_Settings_of_OpenVPN_Clone_Server_Function)
- [https://github.com/SoftEtherVPN/SoftEtherVPN_Stable](https://github.com/SoftEtherVPN/SoftEtherVPN_Stable)

