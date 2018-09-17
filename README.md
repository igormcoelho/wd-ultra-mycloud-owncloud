# OwnCloud on Docker on a Western Digital Ultra (MyCloud Home Server)

This project makes it possible to run OwnCloud on Docker on a Western Digital Ultra (inspired by a Respberry Pi idea).

This is a very experimental project and will certainly void your warranty on WD Ultra... but it is not intended to destroy your MyCloud, just to put OwnCloud working together (because it is an amazing software!).

These are the steps I have done to achieve that (please contribute if you think it's not good enough!):

1) setup your MyCloud normally (including RAID), and log-in via SSH to your WD Ultra (mine is precisely WD My Cloud EX2 Ultra)

2) discover where your data is stored with `df -h` (mine is on `/mnt/HD/HD_a2`)

3) create a folder for storing your ownCloud software and data: `mkdir /mnt/HD/HD_a2/owncloud_www`

4) Discover your docker version (mine is 1.7.0/API 1.19). Since it's a little old, it doesn't have command `docker volume`.. so, this is a workaround

4.1) Create a data volume container for your data: `docker create -v /mnt/HD/HD_a2/owncloud_www:/var/www/ --name owncloud_www resin/rpi-raspbian:latest`

4.2) Create a data volume container for your mysql data: `docker create -v /mnt/HD/HD_a2/mysql_data:/var/lib/mysql --name mysql_data resin/rpi-raspbian:latest`


5) Download these Dockerfile and configurations (I used wget for this). I put these in a folder `new_owncloud_docker`, but it doesn't matter.

6) Build Dockerfile (mine is slightly the same as the original): `docker build -t comzone/rpi-owncloud:latest .`

7) Run owncloud daemon: `docker run --restart=always --volumes-from owncloud_www --volumes-from mysql_data -d -i
 -t -p 4430:443 -p 8000:80 comzone/rpi-owncloud`

8) Enter docker container (docker exec ..... /bin/bash), go to `/var/www` folder and download owncloud: `cd /var/www && wget -q -O - http://download.owncloud.org/community/owncloud-latest.tar.bz2 | tar jx -C .`  setup permissions too: `chown -R www-data:www-data owncloud`

9) Exit docker container, and verify your owncloud data exists on local volume folder: `ls -la /mnt/HD/HD_a2/owncloud_www/`. It should display: `drwxr-xr-x   13 33       33            4096 Sep 16 23:55 owncloud`, or `drwxr-xr-x 13 www-data www-data 4096 Sep 17 02:55 owncloud`

10) Go to your home system, suppose MyCloud is running at 192.168.1.102, so you can find owncloud at 192.168.1.102:8000. Create an admin password and select MariaDB. Default user can be root, password root123, database ownclouddb. Database is not being kept on volume for now, so take care of not destroying the database and container (this should be improved in the future).

## Configuring OpenVPN (testing)

1) Select a volume name for the openvpn data: `OVPN_DATA="ovpn-data"`

2) Create the data volume container: `docker create --name $OVPN_DATA -v /mnt/HD/HD_a2/openvpn_data:/etc/openvpn hypriot/armhf-busybox`

3) Create a public domain server. I'm using no-ip.com service, selected name `xxxx.ddns.net`

4) Run config openvpn: `docker run --volumes-from $OVPN_DATA --rm evolvedm/openvpn-rpi ovpn_genconfig -u udp://xxxx.ddns.net`



Read about docker at http://docker.com

This project is inspired by:  
git://github.com/comzone/rpi-owncloud.git
http://dischord.org/2013/07/10/docker-and-owncloud/
http://dischord.org/2013/08/13/docker-and-owncloud-part-2/
