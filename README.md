# Raeos
Download the official Operating System for the robot autonomy effector
# Connect 
## Via Ethernet
To setup an connection to the rae via Ethernet give your interface the right IP. It has to be `10.10.0.10`.
For Linux based Systems it is:
```bash
sudo ifconfig IF 10.10.0.10 netmask 255.255.255.0
```
IF is for example eth0. After setting the correct IP Adress you can connect via SSH.
The Standard-user is `romzn` and the password is `raeisgreat`. 
Rae hosts an little DNS Server which enables you to reach the System via `rae.local`.


```bash
ssh romzn@rae.local
```

# Via Wifi
The rae opens an access point with the name "rae" with the password "raeisawesome".
After successfully connected to the hotspot you can connect via ssh in the same manner like before:

```bash
ssh romzn@rae.local
```

# Via VSCODE SSH Extension
Install the following extension and setup an ssh connection with the `rae.local` domain and the user romzn
https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh

# Enable Internet through Host Computer
To route the Internet from your Host to the rae several steps are necessary. 
The following internet configuration is for the Ubuntu linux distribution:

First of all you have to activate Port forwarding on your Host and not on the RAE
```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

after that you have to set some entries into your iptables to forward the packages correctly
```bash
sudo iptables -t nat -A POSTROUTING -s 10.7.0.1/24 -j MASQUERADE
sudo iptables -t nat -A POSTROUTING -s 10.10.0.1/24 -j MASQUERADE
```
To save them permanently onto your file System do:

```bash
sudo apt install iptables-persistent
sudo iptables-save  > /etc/iptables/rules.v4
```
Now you can test with `ping www.google.de` if the connections works.

# Add a new user
Add a new user and add them to the groups __dialout__ and __tty__

```bash
sudo adduser otto
sudo usermod -a -G tty otto
sudo usermod -a -G dialout otto
```



## Backup an Image
Write SD-Card to an Image
```bash
sudo dd bs=4M if=/dev/sdb | pv | sudo dd of=raeos18-`date +%d%m%y`.img
```
Install PiShrink
```
wget https://raw.githubusercontent.com/Drewsif/PiShrink/master/pishrink.sh
chmod +x pishrink.sh
sudo mv pishrink.sh /usr/local/bin
```
Example for Shrinking an Image
```
sudo pishrink.sh pi.img
```
