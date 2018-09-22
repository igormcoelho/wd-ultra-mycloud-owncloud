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

7.1) Enter docker container (docker exec ..... /bin/bash). The mysql should have been destroyed too because of the data container volume... so, to rebuild it:
`mysql_install_db --user=mysql --ldata=/var/lib/mysql`

`/usr/bin/mysqladmin -u root password 'root123'`

`cat /etc/mysql/debian.cnf # view debian password, suppose its MPxBDvZrJKq99eJS` 

`mysql -u root -p`

`GRANT ALL PRIVILEGES ON *.* TO 'debian-sys-maint'@'localhost' IDENTIFIED BY 'MPxBDvZrJKq99eJS';`

`service mysql start`

7.2) Maybe you will need to `chown -R mysql:mysql /var/lib/mysql` at some point, I don't know exactly now...

8) Enter docker container (docker exec ..... /bin/bash), go to `/var/www` folder and download owncloud: `cd /var/www && wget -q -O - http://download.owncloud.org/community/owncloud-latest.tar.bz2 | tar jx -C .`  setup permissions too: `chown -R www-data:www-data owncloud`

9) Exit docker container, and verify your owncloud data exists on local volume folder: `ls -la /mnt/HD/HD_a2/owncloud_www/`. It should display: `drwxr-xr-x   13 33       33            4096 Sep 16 23:55 owncloud`, or `drwxr-xr-x 13 www-data www-data 4096 Sep 17 02:55 owncloud`

10) Go to your home system, suppose MyCloud is running at 192.168.1.102, so you can find owncloud at 192.168.1.102:8000. Create an admin password and select MariaDB. Default user can be root, password root123, database ownclouddb. Database is not being kept on volume for now, so take care of not destroying the database and container (this should be improved in the future).

## Configuring no-ip

1) You need a router that supports NAT Lookback: http://opensimulator.org/wiki/NAT_Loopback_Routers My D-LINK DIR-809 router did not support, so I changed to a TP-LINK 740 that supports it.

2) Log-in to no-ip and register your dynamic domain. On the NAS device (via SSH), create a file on mounted disk `/mnt/HD/HD_a2/noipupdater/noipupdater.sh`, and create auxiliary folders `noipupdater/configdir` and `noipupdater/logdir`. Example is here: `https://raw.githubusercontent.com/AntonioCS/no-ip.com-bash-updater`. Use encoded email and password (x@gmail.com => x%40gmail.com), this may be useful: `https://meyerweb.com/eric/tools/dencoder/`

3) Add execution permission `chmod +x /mnt/HD/HD_a2/noipupdater/noipupdater.sh` and add line to `crontab -e`: `*/15 * * * * /mnt/HD/HD_a2/noipupdater/noipupdater.sh` . It will refresh IP after 15 minutes, and only submit DNS request if IP changes.

4) Edit `vi owncloud_www/owncloud/config/config.php`, and add to `trusted_domains` : `1 => 'xxx.ddns.net:8000`, if your port is 800.

5) Enter docker container and edit `/etc/apache2/apache2.conf`, adding `ServerName xxx.ddns.net`. For this to work, `start.sh` script must also use `xxx.ddns.net` instead of `$(hostname)`, to generate a correct ssl certificate

6) Forward port 8000 and 4430 to your NAS server, from your router

## memcaching with apcu/redis (server becomes MUCH faster!!)

https://doc.owncloud.org/server/9.0/admin_manual/configuration_server/caching_configuration.html#id4

1) Enter server with `docker exec ... /bin/bash`

2) `apt install redis-server php5-redis php5-apcu`

3) edit `/var/www/owncloud/config/config.php` (using Redis/APCu)  and add:
```
   'memcache.locking' => '\OC\Memcache\Redis',
   'memcache.local' => '\OC\Memcache\APCu',
   'redis' => array(
     'host' => 'localhost',
     'port' => 6379,
      ),
```
4) Edit `/etc/supervisor/conf.d/lamp.conf` and add:
```
[program:redis]
command=/usr/bin/redis-server
autorestart=true
```

5) Restart web server. The best way I found was: `pkill start.sh`. That killed my SSH session, but everything was restarted (I tried other ways, but didn't succeed in truly restarting all apache sessions).

## limiting number of apache instances (MPM)

Apache was consuming too much memory (I only have 1GB) and leaving a lot of work to SWAP (even my SSH sessions suddently got slow...). So, default MPM was giving too much instances on memory (around 13, each with 180MB), so I limited that. Edit `/etc/apache2/mods-enabled/mpm_prefork.conf`, my values are (around 8 servers now):
```
<IfModule mpm_prefork_module>
	StartServers	          3
	MinSpareServers		  3
	MaxSpareServers		 5
	MaxRequestWorkers	  100
	MaxConnectionsPerChild   0
</IfModule>
```

## Configuring OpenVPN (testing)

1) Select a volume name for the openvpn data: `OVPN_DATA="ovpn-data"`

2) Create the data volume container: `docker create --name $OVPN_DATA -v /mnt/HD/HD_a2/openvpn_data:/etc/openvpn hypriot/armhf-busybox`

3) Create a public domain server. I'm using no-ip.com service, selected name `xxxx.ddns.net`

4) Run config openvpn: `docker run --volumes-from $OVPN_DATA --rm evolvedm/openvpn-rpi ovpn_genconfig -u udp://xxxx.ddns.net`

## Manually Adding Files do ownCloud

If you want to add GigaBytes of files, please don't use sync, it will take years!! Use SSH or USB to copy your files directly to `/mnt/HD/.../owncloud_www/data/USERNAME/files/NEW_DIRECTORY`. To index these files, perform a `docker exec ... /bin/bash` into your container, and execute: `cd /var/www/owncloud`, `sudo -u www-data php occ files:scan --path "USERNAME/files/NEW_DIRECTORY"`

## setup Calendar (CalDav)

Enter admin user and install Calendar app. Install some Android Calendar (such as SimpleCalendar) and Android CalDav connector (such as OpenSync).

## setup Collabora Office ("Google Docs")

First of all, that won't work in your MyCloud device, unfortunately... it seems the whole software is still not compatible with ARM (by 2018), it consumes several GB of RAM and occupies a lot of disk space. In the future, that may be compatible with Raspberry Pi, but for now, best thing is to install in on a 3rd party computer (it could be a Cloud Computer on DigitalOcean, for example).

1) On x86_64 computer: `docker run -t -d -p 9980:9980 -e "domain=xxx.ddns.net" -e "cert_domain=xxx.ddns.net" -e "username=admin" -e "password=S3cRet" --restart always --cap-add MKNOD collabora/code`

2) Wait a few minutes and try in this computer: `curl -v https://localhost:9980`. If answer is `OpenSSL SSL_connect: SSL_ERROR_SYSCALL in connection to localhost:9980`, wait a little longer, until answer is: `curl: (60) SSL certificate problem: self signed certificate in certificate chain`

3) Adjust your router to manage your domain and port 9980, then try `curl -v https://xxx.ddns.net:9980` on your MyCloud device.

3.1) open `https://xxx.ddns.net:9980` on your browser and make sure you accept the self-signed certificate (works on Firefox, but not too good on Chrome... it would be better to have a lets encrypt certificate)

4) Install Collabora (richdocuments) with admin on owncloud, and configure domain: `nano /var/www/owncloud/apps/richdocuments/lib/appconfig.php` with `'wopi_url' => 'https://xxx.ddns.net:9980'` 

5) Get self-signed certificate on x86_64 machine: `docker exec -it YOUR_CONTAINER cat /etc/loolwsd/ca-chain.cert.pem`

6) Add certificate on owncloud docker: `nano /var/www/owncloud/resources/config/ca-bundle.crt`, go to last line and add the contents of the certificate from last step.

7) Open owncloud Collabora (Rich Documents) in any user, and that should work.

## future advices

Should have started from noip configuration first! So all scripts are already done correctly with noip server. I had to do it twice, first locally, then realized it wouldn't work in the outside world... must think on this before everything starts.

Perhaps it was nice to have a volume also for apache certificates...

## More information

Read about docker at http://docker.com

This project is inspired by:  
git://github.com/comzone/rpi-owncloud.git
http://dischord.org/2013/07/10/docker-and-owncloud/
http://dischord.org/2013/08/13/docker-and-owncloud-part-2/
