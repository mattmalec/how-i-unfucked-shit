# Migrating my personal infrastructure - Feb 15, 2022
I had to migrate my personal infrastructre because we're getting new nodes. So here's what I need to do to migrate everything.

## Rsync Example
```sh
rsync -rhvP folder user@ip:/directory
```
This recursively moves a folder into the remote directory, while showing human-readable progress.

## Things to install
- Pterodactyl
- Wings
- Docker & Docker Compose
- OpenJDK 8 & 11, make sure to install both JRE and JDK

## Things to do before moving

### MariaDB
All MariaDB users need to be recreated in the new database before proceeding.
- Export all databases from MariaDB
  - `mysqldump -u root -p --all-databases > backup.sql`
- Import all databases back into MariaDB
  `mysql -u root -p < backup.sql`

## Things to move
### Haste Server
Realistically, I just need to install Docker Compose, move the folder, and start. Mine requires Redis, so that should be running beforehand.

### InfluxDB
I use this for Nova, specifically version `1.8.9`. So that needs to be installed for analytics.

#### Installation
```sh
wget -qO- https://repos.influxdata.com/influxdb.key | gpg --dearmor > /etc/apt/trusted.gpg.d/influxdb.gpg
export DISTRIB_ID=$(lsb_release -si); export DISTRIB_CODENAME=$(lsb_release -sc)
echo "deb [signed-by=/etc/apt/trusted.gpg.d/influxdb.gpg] https://repos.influxdata.com/${DISTRIB_ID,,} ${DISTRIB_CODENAME} stable" > /etc/apt/sources.list.d/influxdb.list
sudo apt update
sudo apt install influxdb
```

Steps:
- After moving `/var/lib/influxdb`, assign permissions to the folder on the new machine
  - `chown -R influxdb:influxdb /var/lib/influxdb`, then start the service

### Pterodactyl
I really just need to move these directories and that's it. Just make sure the database credentials match the new MariaDB.
Directories:
- `/var/www/pterodactyl`
- `/var/lib/pterodactyl`
- `/etc/pterodactyl`

### Nginx Configuration
Directories:
- `/etc/nginx/ssl`
- `/etc/nginx/conf.d`

### Jenkins
Directory:
- `/var/lib/jenkins`

#### Installation
```sh
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt update
sudo apt install jenkins
```

Steps:
- Set Jenkins to run as root (this is a dedicated VM for Jenkins and I'm not worried)
  - In `/etc/default/jenkins`, set 
```
JENKINS_USER=root
JENKINS_GROUP=root
```
And additionally in the same file, set `JAVA_ARGS="-Djava.awt.headless=true -Dhudson.model.DirectoryBrowserSupport.CSP="`.
This allows the [P4J](https://github.com/mattmalec/Pterodactyl4J) javadocs search functionality to work.

### Nexus
Directories:
- `/opt/nexus`
- `/opt/sonatype-work`

Steps:
- Create the Nexus user:
  - `useradd -M -d /opt/nexus -s /bin/bash -r nexus`
- Allow Nexus to run commands without sudo prompt:
  `echo "nexus   ALL=(ALL)       NOPASSWD: ALL" > /etc/sudoers.d/nexus`
- Assign correct permissions
  - `chown -R nexus:nexus /opt/nexus`
  - `chown -R nexus:nexus /opt/sonatype-work`
- Start the Nexus server:
  - `sudo -u nexus /opt/nexus/bin/nexus start`

Read Nexus logs: `tail -f /opt/sonatype-work/nexus3/log/nexus.log`

