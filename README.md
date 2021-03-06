# Qubes-vpn-support
Secure VPN VMs in [Qubes OS](https://www.qubes-os.org)

---
---
#### Help test the new VPN software being submitted for inclusion with Qubes OS:
#### See [qubes-tunnel](https://github.com/tasket/qubes-tunnel) for info!
---
---

Features
-
* Provides a **fail-closed**, anti-leak VPN tunnel environment
* Isolates the tunnel client within a dedicated Proxy VM
* Isolates programs local to VPN VM from network
* Separate firewall VM not required

### Easy setup
  * Simple install script; No file editing or IP numbers necessary
  * Flexible installation into template or to individual proxyVMs
  * Uses configuration files from VPN service provider
  * Less risk of configuration errors

### New in this version, v1.4.0
  * Qubes 4.0 support
  * Anti-leak for IPv6
  * All DNS requests re-addressed to VPN's DNS
  * Firewall integrity checked before connecting
  * Quicker re-connection
  * Supports passwordless cert authentication

---

Quickstart setup guide
-

1. Create a proxyVM using a template with VPN/tunnel software installed (i.e. OpenVPN). (In Qubes 4.0 a proxyVM is called an `AppVM` with the `provides network` option enabled; this document will use the more descriptive `proxyVM` term...)

   Make a choice for the networking/netvm setting, such as `sys-net`.

   Next, add `vpn-handler-openvpn` to the proxyVM's Settings / Services tab. Do not add other network services such as Network Manager.

2. Copy the VPN config files from your service provider to the proxyVM's /rw/config/vpn folder, then copy or link the desired config to 'vpn-client.conf':

   ```
   cd /rw/config/vpn
   sudo unzip ~/ovpn-configs-example.zip
   sudo ln -s US_East.ovpn vpn-client.conf
   ```

   Note: This is a good point to test the connection. See the Testing section below for tips.

3. Decide whether you want a template or proxyVM-only installation. Copy the Qubes-vpn-support folder to the template or proxy VM, then run install. You will be prompted for your VPN login credentials either in this step (proxyVM) or next step (template):

   ```
   cd Qubes-vpn-support
   sudo bash ./install
   ```

4. If installed to a template, shutdown the templateVM then start the proxyVM and finish setup with:

   ```
   sudo /usr/lib/qubes/qubes-vpn-setup --config
   ```

Restart the proxyVM. This will autostart the VPN client and you should see a popup notification 'LINK IS UP'!

Regular usage is simple: Just link other VMs to the VPN VM and start them!

### Updating from prior versions

Download the new Qubes-vpn-support release from github to your VM as before, then run the `sudo bash ./install` command to reinstall. The username/password entry can be skipped by pressing Ctrl-C at the prompt.

### Locating and downloading VPN config files

Some popular VPN providers:
* PIA
      https://www.privateinternetaccess.com/pages/client-support/#fifth
* Mullvad (choose platform: Linux)
      https://mullvad.net/en/download/config/
* NordVPN
      https://nordvpn.com/tutorials/linux/openvpn/

---

Technical notes
-

### Operating system support

Qubes-vpn-support is tested to run on Debian 9 and Fedora 28 template-based VMs under Qubes OS releases 3.2 and 4.0. It is further tested to operate in tandem with [Whonix](https://www.whonix.org) gateway VMs to tunnel Tor traffic and/or tunnel over Tor.

Note that upcoming VPN tunnel support packaged with Qubes OS will likely contain most
of the features in Qubes-vpn-support v1.4. Therefore, most users should
consider using the Qubes built-in feature instead when it becomes available. This project will continue for people looking to experiment with custom tweaks and new features.

### OpenVPN
* The OpenVPN version tested here is 2.4.x.

* It is assumed that 'tun' mode will be used by the VPN as this is by far the most common. The 'tap' mode may work, however it is currently untested.

* Routing details are determined by the VPN provider and can be viewed in the service log if required for troubleshooting. They will appear as references to "route" and "gateway".

### Testing
* Connections can be manually tested with a command like `sudo openvpn --cd  /rw/config/vpn --config vpn-client.conf --auth-user-pass userpassword.txt` but using `systemctl status qubes-vpn-handler` and `journalctl` commands also work to monitor auto-started connections.

* For manual DNS testing you can set DNS addresses in a CLI with:
  ```
  # Use 'up' for downstream VMs or 'test-up' for internal proxyVM tests
  sudo env vpn_dns="<dnsaddress1> <dnsaddress2>" /usr/lib/qubes/qubes-vpn-ns up
  ```

  Similarly, you can use `vpn_dns` to permanently override the DNS that your provider assigns. For openvpn use `setenv` in the config file like this:
  ```
  setenv vpn_dns 'dnsaddress1 dnsaddress2'

  ```

* You should be able to use `ping` and `traceroute` from a downstream appVM after connecting.

### Tor/Whonix notes
Qubes-vpn-support can handle either Tor-over-VPN (configuring sys-whonix `netvm` setting to use VPN VM) or the reverse, VPN-over-Tor (configuring VPN VM `netvm` setting to use sys-whonix). The latter requires the VPN client to be configured for TCP instead of UDP protocol, and a different port number such as 443 may be required by your VPN provider; For openvpn this can all be specified with the `remote` directive in the config file.

### Using clients other than OpenVPN
The main issue with using another client is how you run it. In most cases, adding a conf file under qubes-vpn-handler.d to change the relevant variables and options should be sufficient. An experimental conf for wireguard is included and can be activated by removing '.example' from the filename; it has been tested with Mullvad.net.

Passing the DNS addresses to `qubes-vpn-ns` is another issue: If your client doesn't automatically pass `foreign_option` vars in the same format as openvpn, then on connection set the `vpn_dns` environment variable to one or more DNS addresses separated by a space. In the wireguard example this is accomplished by adding override functions to `wg-quick`.

Since it is the job of a VPN vendor to focus tightly on __link__ security, you should be wary of VPN clients that manipulate iptables or nftables in an attempt to secure the system's communications profile (i.e. prevent leaks); they probably don't take Qubes' unusual network topology into account in which case anti-leak would fail. Normally, security should be added to a VPN setup from the OS or specialty scripts (like these) or by the admins and users themselves. An exception to this is the LEAP bitmask client, which alters iptables with its own anti-leak rules that account for Qubes.

### Link security
A secure VPN service will use a certificate configuration, usually meaning `remote-cert-tls` is used in the openvpn config; This is the best way to protect against MITM attacks and ensure you are really connecting to your VPN service provider. Conversely, restricting access to particular addresses via the firewall is probably not going to substantially improve link security as IP addresses can be spoofed by an attacker.

### About proxy-firewall-restrict
This script builds on the internal rules already set by Qubes firewall in a Proxy VM, and puts the VM in a very locked-down state for networking.

On Qubes 3.x this script is linked to qubes-firewall-user-script. Adding your own iptables rules is most easily done by copying the original script to /rw/config and inserting new rules near the top. The section that 'Corrects nameserver IPs' corrects a Qubes 3.x issue that prevented the VPN VM from being used as a "Deny except" whitelist firewall.

On Qubes 4.x this script is linked to /rw/config/qubes-firewall.d/90_tunnel-restrict and you can add a custom script in the qubes-firewall.d folder to include your own rules.

When any `vpn-handler*` Qubes service is active, outgoing traffic is controlled by group ID of the running process; only `qvpn` group is granted access. So even if your VPN provider has dozens of IPs randomly-assigned via DNS or uses a client other than openvpn then no editing of the firewall script should be necessary. However, this group-based control can be safely removed if necessary by simply changing OUTPUT policy to ACCEPT; the restriction exists only to prevent accidental clearnet access from within the VPN VM and does not affect anti-leak rules for connected downstream VMs.

If `proxy-firewall-restrict` is used to secure another connection scheme (i.e. Network Manager), this arrangement should work as long as a `vpn-handler*` service is *not* specified. Otherwise, Network Manager would need some way of running the VPN client under the `qvpn` group.

Also, local traffic to and from tun0 and vif+ is disallowed. However, the current version allows ICMP packets so if you think blocking these is necessary you can un-comment the ICMP section of the script.

### About qubes-vpn-ns
This handler script is tested to work with OpenVPN v2.3 and v2.4, but should be easily adaptable to other VPN clients with one or two variable-handling changes.

DNS addresses are automatically acquired from DHCP without altering the local resolv.conf settings, so the VPN's DNS is normally only available to attached downstream VMs. This is intended as a privacy measure to prevent any inadvertant access by local (VPN VM) programs over the VPN tunnel.

A change was made in 1.4beta2 to ensure that misconfiguration or malware in an appVM does not address DNS requests to other DNS servers. Previously all DNS requests were (as they are now) either sent through the tunnel or blocked, but the dnat rules mirrored the Qubes default and the destination address of the DNS request was not always forced.

### Basic concepts

* The VPN VM is generally trusted. It is assumed its other programs won't try to impersonate openvpn (send data via port 1194), for example.
* Everything outside the VPN VM and VPN server is essentially untrusted (from the VPN client's point of view): This means the sys-net, local router or Wifi access point, ISP and downstream VMs are potential threats. (This doesn't affect the users POV of whether individual appVMs or sites are trusted.)
* Everything that is downstream from VPN VM communicates through the VPN tunnel only.
* The purpose of the programs in the VPN VM is to support the creation of the VPN link. Their net access is either null or clearnet only; they should not send packets through the VPN tunnel and potentially get published.
* Configuration of the VPN client details (server address, protocols, etc) should be downloaded from the VPN provider's support page; the user can simply drop the config file into the /rw/config/vpn folder and rename it.

### Releases
v1.4 rc2, July 2018

v1.3 beta, July 2017

v1.0.2, June 2016

### See also
[OpenVPN documentation](https://openvpn.net/index.php/open-source/documentation.html)

[Whonix - Tor networking in Qubes](https://www.qubes-os.org/doc/whonix/install/)

[Qubes VM hardening](https://github.com/tasket/Qubes-VM-hardening)

