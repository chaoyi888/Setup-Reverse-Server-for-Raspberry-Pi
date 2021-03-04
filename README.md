# Setup-Reverse-Server-for-Raspberry-Pi

The followin steps are to create a relay service from decentral units to one central  unit. So that you can access the remote raspberry pi (not in the same LAN network) via a central server by SSH instead of using Teamviewer or VNC Server.

**Note: the condition is: **
1. the central server needs to have a public IP address and can be accessable via SSH from internet.
2. The Remote Raspberry Pi needs to have internet access via a network (WiFi or fixed line)

## setup keypair at remote station (the Raspberry Pi at remote network)
  1. Create key pair using the following command:
  ssh-keygen -t rsa -b 4096
  2. use empty pass phrases.
  3. copy the public key to the central server under the username which you are using to login, e.g.:
     /home/pi/.ssh/authorized_keys

## setup a user and keypair at central server (e.g. a server on AWS which has public IP address)
**Note: if you can already login to the central server via SSH by using your own public key, this step might not be necessary.**
  1. Craete a new user if you want to use this user to login. In the case for Eindhoven, the username is EC2-user which is already configured at the central server on AWS. 
      sudo adduser newuser
  2. setup a key pair for this user
      ssh-keygen -t rsa -b 4096
  3. use empty passphrases
  4. copy the public key of this user to the file `/home/newuser/.ssh/authorized_keys`. You might also want to use an oher use name :)

## setup service file for callback service to central service at Raspberry Pi
  1. create service file `/etc/systemd/system/ssh-relay.service` with following content (but change the port number 2200 to a unique port):

  [Unit]  
  Description=Enable relay from central location  
  After=network-online.target
  Wants=network-online.target

  [Service]  
  Type=simple 
  ExecStart=/usr/bin/ssh -l Username_On_Central_Server Public_IP_Central_Server -R 2001:localhost:22 -L 8086:localhost:8086 -p22 -N -v -i /home/pi/.ssh/id_rsa(the location of   your private key on the raspberry pi) -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no

  [Install] 
  WantedBy=multi-user.target

  **note: please be carefully of the username, IP of the central server. Also the port number needs to be unique, e.g. 2001**

## Afterwards, run the following commands on your raspberry Pi
  1. sudo systemctl daemon-reload
  2. sudo systemctl enable ssh-relay
  3. sudo systemctl start ssh-relay

make sure that for every remote unit you use a different port. In above example it is 2200. You can increment this for each new unit. Please keep an administration on this!!

## testing on the remote raspberry Pi
look at the logfile of the remote unit to see if everything is working:

journalctl -f

## login to the raspberry Pi via Central server
  1. login via SSH to the central server
  2. When you log in from the central server to the remote unit use (change the port number to the one choosen before):
      ssh username_on_your_Pi@localhost -p2001
  3. then you should be able to login to the remote raspberry Pi via a central server without using Teamviewer or VNC_Server. 
  4. But it's still good to have Teamviewer and VNC_Server in case of urgency.

also please change the username to an appropiate user.

## The SSH tunnel on the remote Raspberry Pi is always hanging there after a certain minutes inactivity. The following steps can be taken so that the tunnel won't hang there 
## anymore.
  1. In /etc/ssh/ssh_config, append the following two lines at the last. 
    @ sudo nano /etc/ssh/ssh_config
    @ sudo /etc/init.d/ssh restart # restart 
    ServerAliveInterval 60
    ServerAliveCountMax 10
  2. In /etc/ssh/sshd_config, uncomment the following lines:
    ClientAliveInterval 60
    ClientAliveCountMax 10
    TCPKeepAlive yes
    Close and save the file, then restart sshd, e.g.:/etc/init.d/ssh restart or: service sshd restart 
  3. In my case, the SSH tunnel is never hanging there anymore. The reverse SSH connection from central server (AWS server) to the remote Raspberry Pi is always alive. 

## For the remote connection, there is always some unexpected behavior happening. So it's better that we define a crontab job to reboot the Pi every day at a certain time. 
## I have used the combinations below:
    1.  Add the following line in the /etc/crontab to schedule reboot at certain moment of the day. 
    00 00 * * * root reboot                     #Every day at 00:00, Pi will be reboot
    2.  Add the following lines by using 'Sudo crontab -e' so that the following 4 commands will be executed after 5 minutes at every reboot.
      @reboot sleep 120 && sudo /usr/sbin/route del default dev eth0
      @reboot sleep 130 && sudo /usr/bin/systemctl daemon-reload
      @reboot sleep 140 && sudo /usr/bin/systemctl enable ssh-relay
      @reboot sleep 150 && sudo /usr/bin/systemctl start ssh-relay  
