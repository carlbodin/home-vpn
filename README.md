# Home VPN server on Raspberry Pi

This guide sets up a home VPN server hosted on a Raspberry Pi using DuckDNS as DDNS, PiVPN for wireguard installation and authentication setup, and the Wireguard mobile app to connect clients. A VPN may be used for controlling your home devices or safe internet browsing when you are on the go.

## Create a domain and setup DDNS

I am using DuckDNS as DDNS since it is free and easy to use.

1. Create account and a free subdomain on duckdns.org.
2. Install duckdns on your vpn server following this [guide](https://www.duckdns.org/install.jsp), or the steps below.
3. Connect to server device `ssh username@hostname`.
4. Make a directory and cd into it.

```
mkdir duckdns
cd duckdns
```

5. Create a bash script `nano duck.sh` and copy this line into it.

```
echo url="https://www.duckdns.org/update?domains=EXAMPLEDOMAIN&token=EXAMPLETOKEN&ip=" | curl -k -o ~/duckdns/duck.log -K -
```

Also, make the file executable running `chmod 700 duck.sh`.

6. Make this script run every 5 minutes using the cron process. Edit user's crontab running `crontab -e` and add the line

```
*/5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1
```

to the bottom of the file.

7. Test the script `./duck.sh`. We can also see if last attempt succeded (OK) or not (KO) running `cat duck.log`. If KO, check domain and token in duck.sh.
8. Finally, ensure cron autostart on reboot. This should be on by default on Raspberry Pi OS. If not, try `sudo service cron start`.

## Setup router port forwarding UDP

1. Enable port forwarding on home router. There are no official guides for this since it is almost a unique interface for each router and model.
2. Wireguard default it 51820. Use this unless you have a good reason not to.
3. Make sure the protocol is set to UDP, not TCP.

## Install and setup PiVPN on server

1. Have an ssh-able unit with Raspberry PI OS installed. Preferrably the "Light" and latest OS version according to pivpn-io, but any will do as long as ssh access is enabled at the start.
2. Connect to the device using ssh and install pivpn using `curl -L https://install.pivpn.io | bash`.
3. Run the pivpn setup process choosing wireguard, port number 51820, and the chosen domain name for the ddns server. If unsure, just hit enter since it is designed to make you have a working config at the end.

Wireguard will default to Curve25519 public key authentication method, which provides 128-bit security. Use the command `pivpn` to manage the server.

## Setup authentication between server and clients

Perhaps the PiVPN setup deals with this for you.

1. Generate keys.
2. Save in config files.

## Setup mobile client

1. Install the Wireguard app.
2. On the Rasperry Pi OS server, run `pivpn add` and select a username to connect a new device. Then, run `pivpn -qr` and a QR code will be displayed in the terminal window.
3. Scan the code through the Wireguard app, and it is all setup.
4. Tips: add Wireguard shortcut to the Android dropdown menu for quick VPN access.

## Setup Linux client

### 1. Create a new client profile on the host.

Use PiVPN on the Raspberry Pi to generate a new config.

```bash
pivpn add
```

Add a name for your connection. It says hyphens are allowed, but they did not work for me.

### 2. Install Wireguard on the client.

For Ubuntu and Debian:

```bash
sudo apt install wireguard resolvconf
```

See guides for other distros [here](https://www.wireguard.com/install/).

### 3. Setup Wireguard on the client.

As root user, create the /etc/wireguard folder and prevent anyone but root to enter it:

```bash
sudo -i
mkdir -p /etc/wireguard
chown root:root /etc/wireguard
chmod 700 /etc/wireguard
logout
```

Then, move the config file from the pi to your client device.

```bash
scp pi@raspberry:~/configs/client-profile.conf /etc/wireguard/
```

### 4. Activate the VPN tunnel.

```bash
sudo wg-quick up client-profile
```

Notice how you should not need to enter the absolute path, nor the `.conf` extension.

To exit the VPN.

```bash
sudo wg-quick down client-profile
```

I prefer aliases for faster use.

```bash
alias wgu='sudo wg-quick up client-profile'
alias wgd='sudo wg-quick down client-profile'
```

## Troubleshooting

### General

Note that you cannot create a VPN tunnel within the same network you currently are connected to. I.E., I cannot activate the VPN on my client when on the same LAN as my host.

For troubleshooting, run `pivpn debug` to debug.

### Client profile name and usage

Do you encounter any of the following errors?

```plaintext
wg-quick: PROFILE-NAME does not exist

wg-quick: The config file must be a valid interface name, followed by .conf
```

Try not using hyphens (-) in you client profile config file name.

Also, make sure you run the `wg-quick up` command only follwed by the stem of the client profile file name, no extension.
