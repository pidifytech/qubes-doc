---
layout: doc
title: VPN
permalink: /doc/vpn/
redirect_from:
- /doc/privacy/vpn/
- /en/doc/vpn/
- /doc/VPN/
- /wiki/VPN/
---

How To make a VPN Tunnel in Qubes
==================================

Although setting up a VPN connection is not by itself Qubes specific, Qubes includes a number of tools that can make the client-side setup of your VPN more versatile and secure. This document is a Qubes-specific outline for choosing the type of VM to use, and shows two methods for preparing the preferred VM type (ProxyVM). The first method uses NetworkManager, while the second method if the most fail-safe and demonstrates the new qubes-tunnel feature.

When considering the specific configuration options for your connection(s) you may also need to refer to your guest OS and VPN service documentation ; The relevant documentation for the Qubes default guest OS (Fedora) is [Establishing a VPN Connection.](https://docs.fedoraproject.org/en-US/Fedora/23/html/Networking_Guide/sec-Establishing_a_VPN_Connection.html)

### NetVM

It is possible to set up a VPN connection using the NetworkManager service already running inside your NetworkVM (i.e. sys-net). However this has some disadvantages:

- You have to place (and probably save) your VPN credentials inside the NetVM, which is directly connected to the outside world
- All your AppVMs which are connected to the NetVM will be connected to the VPN (by default)

### AppVM

While the NetworkManager service is not started here (for a good reason), you can configure any kind of VPN client in your AppVM as well. However this is only suggested if your VPN client has special requirements.

### ProxyVM

One of the best unique features of Qubes OS is its special type of VM called a ProxyVM. The special thing is that your AppVMs see this as a NetVM (or uplink), and your NetVMs see it as a downstream AppVM. Because of this, you can place a ProxyVM between your AppVMs and your NetVM. This is how the default sys-firewall VM functions.

Using a ProxyVM to set up a VPN client gives you the ability to:

- Isolate your VPN credentials from your Network VM and the Internet.
- Isolate your VPN credentials from downstream AppVMs.
- Easily control which of your AppVMs are connected to your VPN by simply setting it as a NetVM of the desired AppVM.
- Put firewall restrictions on tunnel traffic without adding another ProxyVM.

In Qubes R4.0, an AppVM that 'provides networking' takes the place of ProxyVM, although they function in the same roles. Here the term 'ProxyVM' will be used to refer to both this type of R4.0 qube as well as its R3.2 counterpart.
 

Set up a ProxyVM as a VPN gateway using NetworkManager
------------------------------------------------------

1. Create a new VM, name it, click the ProxyVM radio button, and choose a color and template.

   ![Create\_New\_VM.png](/attachment/wiki/VPN/Create_New_VM.png)

   If your choice of TemplateVM doesn't already have the VPN client software (i.e. network-manager-openvpn-gnome), you'll need to [install](https://www.qubes-os.org/doc/software-update-vm/#installing-or-updating-software-in-the-templatevm) its packages in the template before proceeding.

2. Add the `network-manager` service to this new VM.

   ![Settings-services.png](/attachment/wiki/VPN/Settings-services.png)

3. Set up your VPN as described in the NetworkManager documentation linked above.

4. Configure your AppVMs to use the new VM as a NetVM.

   ![Settings-NetVM.png](/attachment/wiki/VPN/Settings-NetVM.png)

5. Set anti-leak protection in the firewall. Open a terminal in the ProxyVM and type:

   ~~~
   sudo /usr/lib/qubes/qtunnel-setup --config-nm
   ~~~

6. Optionally, you can install some [custom icons](https://github.com/Zrubi/qubes-artwork-proxy-vpn) for your VPN


Set up a ProxyVM as a VPN gateway using the *qubes-tunnel* service
------------------------------------------------------------------

This method has extended anti-leak features that also make the connection _fail closed_ should it be interrupted. It also has the advantage of using configuration files tailored by your VPN service provider so it may be the easiest way to setup a working and secure link.

The following has been tested with OpenVPN on Fedora 28 and Debian 9 templates:

1. Create a new VM, name it, click the "ProxyVM" radio button (Qubes 3.2) or "provides network" checkbox (Qubes 4.0), and choose a color and template.

   ![Create\_New\_VM.png](/attachment/wiki/VPN/Create_New_VM.png)

   Note: Do not enable NetworkManager in the ProxyVM, as it can interfere with the scripts' DNS features.
   If you enabled NetworkManager or used other methods in a previous attempt, do not re-use the old ProxyVM...
   Create a new one according to this step.

   If your choice of TemplateVM doesn't already have the VPN client software (i.e. OpenVPN), you'll need to [install](https://www.qubes-os.org/doc/software-update-vm/#installing-or-updating-software-in-the-templatevm) its package in the template before proceeding. If your TemplateVM is one of the ["minimal" types](https://www.qubes-os.org/doc/fedora-minimal-template-customization/#proxyvm-for-vpns) then you may need to install additional networking packages.

2. Set up the VPN client.
   Make sure the VPN VM and its TemplateVM is not running.
   Run a terminal (CLI) in the VPN VM -- this will start the VM.
   Then create a new `/rw/config/qtunnel` folder with:

       sudo mkdir /rw/config/qtunnel

   
    Obtain the configuration files from your VPN service provider then copy them to the `/rw/config/qtunnel` folder. Finally, copy or link the desired config file to `qtunnel.conf`. For example:

   ~~~
   cd /rw/config/qtunnel
   sudo unzip ~/ovpn-configs-example.zip
   sudo ln -s US_East.ovpn qtunnel.conf
   ~~~

   Notes on configuration contents:

      * Files accompanying the main config such as `*.crt` and `*.pem` should also go to `/rw/config/qtunnel` folder.
      * Files referenced in `qtunnel.conf` should not use absolute paths such as `/etc/...`.
      * Typical VPN configs will specify a `tun` interface; note that `tap` interfaces are untested with qubes-tunnel.
      * The configuration should automatically route all traffic through your VPN's tunnel interface once a connection is started. See footnote[3] for information.

3. Test your client configuration!

   Run the VPN client from a CLI prompt in the 'qtunnel' folder, preferably as root.
   For example:

       sudo openvpn --cd /rw/config/qtunnel --config qtunnel.conf --verb 3

   Watch for status messages that indicate whether the connection is successful and test from another VPN VM terminal window with `ping`.

       ping 8.8.8.8

   `ping` can be aborted by pressing the two keys `ctrl` + `c` at the same time.
   DNS may be tested at this point by replacing addresses in `/etc/resolv.conf` with ones appropriate for your VPN (although this file will not be used when setup is complete).
   Diagnose any connection problems using resources such as client documentation and help from your VPN service provider.
   Proceed to the next step when you're sure the basic VPN connection is working.

4. Qubes-specific setup.

   On the Services tab of the proxyVM settings, add `qubes-tunnel-openvpn` and make sure it is checked, then click 'OK'. This starts the qubes-tunnel service for OpenVPN when the proxyVM starts.

   Next, at a CLI prompt in the ProxyVM type the following to automatically enable firewall rules and save your login information:

       sudo /usr/lib/qubes/qtunnel-setup --config

   If username and password aren't used by your VPN provider you may leave these blank.
   
5. Restart the new VM!
   The link should then be established automatically with a popup notification to that effect.


Usage
-----

- Configure your AppVMs to use the VPN VM as a NetVM...

![Settings-NetVM.png](/attachment/wiki/VPN/Settings-NetVM.png)

- If you wish to use the [Qubes firewall](/doc/firewall), the settings can be added directly to the AppVM; The VPN VM functions as a firewall VM.

- FIXME FOR R4.0....... If you want to update your TemplateVMs through the VPN, enable the `qubes-updates-proxy` service in your new FirewallVM.
You can do this in the Services tab in Qubes VM Manager or on the command-line:

    qvm-service -e <name> qubes-updates-proxy

Then, configure your templates to use your new FirewallVM as their NetVM.

- The `systemctl` command can be used to control the VPN client, which is running as a systemd service called `qubes-tunnel.service`.

Troubleshooting
---------------

* To view the log, use `journalctl -u NetworkManager` or `journalctl -u qubes-tunnel` as needed.
* A non-functioning, blocked link may result from failure of the qubes-tunnel service, which can be checked with `systemctl status qubes-tunnel`. The service will try to recover from failure by restarting after ten seconds.
* Test DNS: Ping a familiar domain name from an appVM. It should print the IP address for the domain.
* Use `iptables -L -v` and `iptables -L -v -t nat` to check firewall rules. The latter shows the critical PR-QBS chain that enables DNS forwarding.

Notes
-----

[1] This firewall and DNS configuration prevents any packets from being forwarded to or from the 'bare' upstream interface (eth0). It will also isolate the VPN VM configured with `qubes-tunnel` from unnecessary network access.

[2] Most VPN providers with OpenVPN access offer tailored configuration files for the user to download and use. FIXME: maybe show download links for popular/respected VPN services.

[3] Routing of traffic through the tunnel is usually preconfigured by the VPN provider; This happens automatically and can be seen in the openvpn log output as `redirect-gateway def1`. If necessary, the user can manually add this command to the ovpn/conf file.
