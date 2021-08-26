- [How to activate the access point with dns and dhcp](#how-to-activate-the-access-point-with-dns-and-dhcp)
  - [Install the tools](#install-the-tools)
  - [Setup the access-point in ac hw-mode](#setup-the-access-point-in-ac-hw-mode)
    - [Generate hostapd.conf](#generate-hostapdconf)
    - [link hostapd.conf](#link-hostapdconf)
    - [Start the access point manually](#start-the-access-point-manually)
    - [Start acces-point during boot](#start-acces-point-during-boot)
      - [1. Make a script which starts the service](#1-make-a-script-which-starts-the-service)
      - [2. Make an entry to crontab](#2-make-an-entry-to-crontab)
      - [3. Call without sudo](#3-call-without-sudo)
      - [4. Change the netplan](#4-change-the-netplan)
  - [Setup DNS and DHCP for WLAN0 Interface](#setup-dns-and-dhcp-for-wlan0-interface)
    - [1. Diasble the inbuilt resolver](#1-diasble-the-inbuilt-resolver)
    - [2. Create the config file](#2-create-the-config-file)
    - [3. Link the config](#3-link-the-config)
    - [4. Start dnsmasq during boot](#4-start-dnsmasq-during-boot)
    - [3. Test the Wifi Connection](#3-test-the-wifi-connection)
  - [Troubleshooting](#troubleshooting)
# How to activate the access point with dns and dhcp
The project uses an raspberry pi 4B with an Ubunutu 18.04 Operating OS.
Instead of using the inbuilt *netplan* and *systemd-resolve* programms we want to use *dnsmasq* and *hostapd*. This allows us to use special configurations like the ac-mode. 

## Install the tools

```bash
sudo apt-get install hostapd dnsmasq
```

## Setup the access-point in ac hw-mode
### Generate hostapd.conf
`sudo touch /etc/hostapd/hostapd.conf`

```bash
ssid=rae
wpa_passphrase=raeiswaesome

country_code=US

interface=wlan0
driver=nl80211

wpa=2
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP

macaddr_acl=0

logger_syslog=0
logger_syslog_level=4
logger_stdout=-1
logger_stdout_level=0

hw_mode=a
wmm_enabled=1

# N
ieee80211n=1
require_ht=1
ht_capab=[MAX-AMSDU-3839][HT40+][SHORT-GI-20][SHORT-GI-40][DSSS_CCK-40]

# AC
ieee80211ac=1
require_vht=1
ieee80211d=0
ieee80211h=0
vht_capab=[MAX-AMSDU-3839][SHORT-GI-80]
vht_oper_chwidth=1
channel=36
vht_oper_centr_freq_seg0_idx=42
```
### link hostapd.conf
In the file `/etc/default/hostapd` set the aproppriate variable to link the config with the service:

```bash
...
DAEMON_CONF="/etc/hostapd/hostapd.conf"
...
```

### Start the access point manually
```bash
sudo service hostapd start
```
Check the status:

```bash
sudo service hostapd status
```
If successful it should output:
```bash
Aug 25 10:34:09 rae systemd[1]: Starting Advanced IEEE 802.11 AP and IEEE 802.1X/WPA/WPA2/EAP Authenticator...
Aug 25 10:34:09 rae hostapd[1035]: Configuration file: /etc/hostapd/hostapd.conf
Aug 25 10:34:09 rae hostapd[1035]: wlan0: interface state UNINITIALIZED->COUNTRY_UPDATE
Aug 25 10:34:09 rae hostapd[1035]: wlan0: interface state COUNTRY_UPDATE->HT_SCAN
Aug 25 10:34:09 rae systemd[1]: Started Advanced IEEE 802.11 AP and IEEE 802.1X/WPA/WPA2/EAP Authenticator.

```

You can now connect to the access-point but can't interact with it because you won't get an IP. Therefore an DHCP is necessary which will be enabled later.

### Start acces-point during boot
>Normally if an service gets enabled with sudo service hostapd enable it starts automatically during startup. In this case it does not. Various articles in the internet are describing the same problem with the hostapd service. I tried all solutions i found but nothing helps except an entry into crontab which is not the best solution but it works.

#### 1. Make a script which starts the service
The following lines creating a file in system-wide directory with corresponding access-rights for boot operations.
```bash
sudo mkdir -p /etc/startupscripts &&
sudo touch /etc/startupscripts/delayedhostapd.sh &&
sudo chmod a+rx /etc/startupscripts/delayedhostapd.sh
```

`/etc/startupscripts/delayedhostapd.sh` looks like:

```bash
#!/bin/bash
sudo service hostapd restart
```

#### 2. Make an entry to crontab

Type `crontab -e` to start editing the crontab file and add the following statement:

```bash
@reboot sleep 10 && /etc/startupscripts/delayedhostap.sh > /tmp/delayed-restart-hostapd.log 2>&1
```
This statements ensures that the script gets called after 25 seconds since reboot is initiated.

#### 3. Call without sudo
Normally it would require to type in the password when using the service command. But this is not possible during startup. So an entry in the sudoers-file is necessary. Open the file with `sudo visudo` and add to the end of the file:
```bash
romzn ALL=(ALL) NOPASSWD: ALL 
```

#### 4. Change the netplan
`/etc/netplan/10-rae.yaml` looks like
```bash
network:
    version: 2
    renderer: NetworkManager
    ethernets:
        eth0:
            dhcp4: false
            gateway4: 10.10.0.10
            addresses:
            - 10.10.0.1/24
        wlan0:
            dhcp4: false
            addresses:
            - 10.7.0.1/24
```
Afterwards generate and apply the netplan with:

```bash
sudo netplan generate
sudo netplan apply
```
Then reboot the system
## Setup DNS and DHCP for WLAN0 Interface

### 1. Diasble the inbuilt resolver
```bash
sudo systemctl unmask systemd-resolved
sudo systemctl disable systemd-resolved
```

### 2. Create the config file

The config ``/etc/dnsmasq.d/rae.conf`` looks like:
```bash
interface=wlan0
no-dhcp-interface=eth0
dhcp-range=interface:wlan0,10.7.0.2,10.7.0.17,infinite
port=5353
domain-needed
bogus-priv
strict-order
expand-hosts
domain=rae.local
listen-address=127.0.0.1 # Set to Server IP for network responses

```

### 3. Link the config

Ensure that the following statement is in the file`/etc/default/dnsmasq`:
```bash
DNSMASQ_OPTS="--conf-file=/etc/dnsmasq.d/rae.conf"
```

### 4. Start dnsmasq during boot

Again there is problem by starting the service automatically. So we need again an entry to the crontab.
`crontab -e` looks now like this:

```bash
@reboot sleep 10 && /etc/startupscripts/delayedhostapd.sh  > /tmp/delayed-restart-hostapd.log 2>&1
@reboot sleep 15 && /etc/startupscripts/delayeddnsmasq.sh  > /tmp/delayed-restart-dnsmasq.log 2>&1
```

The script /etc/startupscripts/delayeddnsmasq.sh has to look like this:

```bash
#!/bin/bash
sudo ifconfig wlan0 10.7.0.1 netmask 255.255.255.0 up &&
sudo service dnsmasq restart
```
We have to give the wlan0 interface an ip because the inbuilt `netplan` tool was not capable anymore to do this.

Ensure that the access rights are set correctly:
```sudo chmod a+rx /etc/startupscripts/delayeddnsmasq.sh```

Now the server is should be up after the boot sequence.

### 3. Test the Wifi Connection
Connect to the rae hotspot with the password `raeisawesome`. After login check if you have an ip address. The ifconfig-output on the host computer lookslikes this:
```bash
...
wlp3s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.7.0.15  netmask 255.255.255.0  broadcast 10.7.0.255
...

```

I had no connections problems when i started the realsense camera. I measured a signal strength of around 80-95% in the same room. When the realsense is active it drops to 65%-75%.

## Troubleshooting
* If the wifi ssid named rae is not visible to you. Check if your WiFi adapter support ac-mode (5Ghz)