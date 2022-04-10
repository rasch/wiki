# Network

This is an overview of my home network for my future self so I don't
have to fumble through setting this up next time. **Caution**: this
guide does NOT account for your router and unique network needs.

```txt
Internet------+   +------------+      +----------+      +--------PC
              |   |            |      |          |      |
              |   |        Internet  Eth1    Internet  Eth1
            +-+---+-+          |      |          |      |
            | Modem |        +-+------+-+      +-+------+-+
            +-------+        | Router A |      | Router B |
                             +-+------+-+      +-+------+-+
                               |      |          |      |
                              WiFi   WiFi       WiFi   WiFi
                             2.4GHz 5.0GHz     2.4GHz 5.0Ghz
```

| Router A       | routes traffic directly through ISP |
|---------------:|:------------------------------------|
| Model          | TP-Link Archer A7                   |
| IP Address     | http://192.168.1.1                  |
| SSID (5 GHz)   | day                                 |
| SSID (2.4 GHz) | day-2.4                             |

| Router B       | routes traffic through PIA VPN |
|---------------:|:-------------------------------|
| Model          | TP-Link Archer A7              |
| IP Address     | http://192.168.2.1             |
| SSID (5 GHz)   | night                          |
| SSID (2.4 GHz) | night-2.4                      |

## OpenWrt Installation

Download the latest [OpenWrt] firmware for router from the [TP-Link
Archer A7 reference page][router]. See the "Firmware OpenWrt Install
URL" section for the download link or the "Firmware OpenWrt Upgrade
URL" section for upgrading to a newer version of OpenWRT. Install the
firmware using the router's firmware upgrade page within the web
interface. If you are using a different router then be sure to check
the [supported table of hardware][supported] to get the proper firmware
for your device.

## OpenWrt Configuration

The following details how I configure my OpenWRT systems using the
[`uci` command line interface][uci]. Most of the following can be
configured using the web interface, but I prefer using the command line
since it is easier and faster to reproduce on multiple devices.

- [Set root Password](<#set-root-password>)
- [Change LAN IP Address](<#change-lan-ip-address>)
- [Configure Wi-Fi](<#configure-wi-fi>)
- [Configure VPN](<#configure-vpn>)
- [Configure SSH](<#configure-ssh>)
- [Finish](<#finish>)

### Set root Password

Connect to the router over SSH.

```sh
ssh root@192.168.1.1
```

Set the root password.

```sh
passwd
```

### Change LAN IP Address

This step is only performed on "Router B" so that it has a different
address than "Router A". Skip this section when configuring "Router A".

Change the LAN IP address and restart the network.

```sh
uci set network.lan.ipaddr=192.168.2.1
uci commit network
/etc/init.d/network restart
```

After the network restarts reconnect to the router over SSH at its
new address.

```sh
ssh root@192.168.2.1
```

### Configure Wi-Fi

My router has two radios (5.0 GHz and 2.4 GHz) and I enable both of
them on both routers. This makes it easy to connect all of my devices
on the network with the option of connecting through the VPN or not.

Enable 5.0 GHz radio and set country. Change "WIFI_COUNTRY" if you are
not in the US.

```sh
uci set wireless.@wifi-device[0].disabled=0
uci set wireless.@wifi-device[0].country="${WIFI_COUNTRY:-US}"
```

Enable wi-fi interface on the 5.0 GHz radio. Set "WIFI_SSID_5G" and
"WIFI_PASSWORD" before running these commands.

```sh
uci set wireless.@wifi-iface[0].disabled=0
uci set wireless.@wifi-iface[0].ssid="${WIFI_SSID_5G:?}"
uci set wireless.@wifi-iface[0].key="${WIFI_PASSWORD:?}"
uci set wireless.@wifi-iface[0].encryption=psk2
```

Enable 2.4 GHz radio and set country. Change "WIFI_COUNTRY" if you are
not in the US.

```sh
uci set wireless.@wifi-device[1].disabled=0
uci set wireless.@wifi-device[1].country="${WIFI_COUNTRY:-US}"
```

Enable wi-fi interface on the 2.4 GHz radio. Set "WIFI_SSID" and
"WIFI_PASSWORD" before running these commands.

```sh
uci set wireless.@wifi-iface[1].disabled=0
uci set wireless.@wifi-iface[1].ssid="${WIFI_SSID:?}"
uci set wireless.@wifi-iface[1].key="${WIFI_PASSWORD:?}"
uci set wireless.@wifi-iface[1].encryption=psk2
```

Commit changes.

```sh
uci commit wireless
```

### Configure VPN

This step is only performed on "Router B". Skip this section when
configuring "Router A" so that one router accesses the internet
directly from the ISP and the other through the VPN.

Install OpenVPN and other dependencies.

```sh
opkg update
opkg install openvpn-openssl curl unzip
```

Download "Private Internet Access" OpenVPN configuration files.

```sh
cd /etc/openvpn
curl -sLO https://www.privateinternetaccess.com/openvpn/openvpn-strong.zip
unzip openvpn-strong.zip
rm openvpn-strong.zip
```

Set auth-user-pass to "authuser" in OpenVPN configuration files.

```sh
sed -i 's/^auth-user-pass$/auth-user-pass authuser/' /etc/openvpn/*.ovpn
```

Create "authuser" file with "Private Internet Access" credentials and
limit permissions on the file. Be sure to set "PIA_USERNAME" and
"PIA_PASSWORD" before running these commands.

```sh
printf '%s\n%s\n' "${PIA_USERNAME:?}" "${PIA_PASSWORD:?}" >> /etc/openvpn/authuser
chmod 400 /etc/openvpn/authuser
```

Create VPN network interface.

```sh
uci set network.vpn=interface
uci set network.vpn.ifname=tun0
uci set network.vpn.proto=none
uci commit network
```

Create firewall zone for PIA and forward LAN to it.

```sh
uci add firewall zone
uci set firewall.@zone[-1].name=pia
uci set firewall.@zone[-1].network=vpn
uci set firewall.@zone[-1].input=REJECT
uci set firewall.@zone[-1].output=ACCEPT
uci set firewall.@zone[-1].forward=REJECT
uci set firewall.@zone[-1].masq=1
uci set firewall.@zone[-1].mtu_fix=1
uci set firewall.@forwarding[0].dest=pia
uci commit firewall
```

Use privateinternetaccess.com's DNS servers.

```sh
uci add_list dhcp.lan.dhcp_option="6,10.0.0.241,10.0.0.242"
uci commit dhcp
```

Enable OpenVPN on router.

```sh
uci set openvpn.custom_config.enabled=1
uci set openvpn.custom_config.config="/etc/openvpn/${PIA_SERVER:-us_chicago}.ovpn"
uci commit openvpn
```

Reboot router to ensure all changes take effect.

```sh
reboot
```

### Configure SSH

Add public key to the router. Note that you may have to modify the
path to your public key.

```sh
ssh root@192.168.2.1 "tee -a /etc/dropbear/authorized_keys" < ~/.ssh/id_rsa.pub
```

**OR** if using GPG agent as SSH agent then use the following command
instead. Be sure to set "GPG_KEY".

```sh
gpg --export-ssh-key "${GPG_KEY:?}" | ssh root@192.168.2.1 "tee -a /etc/dropbear/authorized_keys"
```

Make sure that you can log in over SSH using key based authentication
rather than password authentication before continuing.

```sh
ssh root@192.168.2.1
```

If key based login is successful, then disable password logins.

```sh
uci set dropbear.@dropbear[0].PasswordAuth=0
uci set dropbear.@dropbear[0].RootPasswordAuth=0
uci commit dropbear
/etc/init.d/dropbear restart
```

### Finish

- Ensure SSH and Wi-Fi are both working.
- Make sure the VPN is working and check for leaks.
  - <https://ipleak.net/>
  - <https://dnsleak.com/>
  - <https://dnsleaktest.com/>
  - <http://ipv6leak.com/>
  - <https://ipv6-test.com/>
- Check network speed (optional).
  - <http://www.speedtest.net/>

[openwrt]: https://openwrt.org
[router]: https://openwrt.org/toh/hwdata/tp-link/tp-link_archer_a7_v5
[supported]: https://openwrt.org/toh/start
[uci]: https://openwrt.org/docs/guide-user/base-system/uci
