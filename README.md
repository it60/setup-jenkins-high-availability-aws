# setup-jenkins-high-availability-aws
Jenkins High Availability setup

# Prepare 3 EC2 instances:
1 Master
1 Slave
1 HAProxy

# Master
```
We shall install samba on CentOs 7
    sudo yum install samba samba-client
start the services
    sudo systemctl start smb.service
    sudo systemctl start nmb.service
enable auto start on reboot
    sudo systemctl enable smb.service
    sudo systemctl enable nmb.service
configure directories
    sudo mkdir /sambashares
create samba group
    sudo groupadd sambashare
change group ownership of samba directories
    sudo chgrp sambashare /sambashares
create samba user
    sudo useradd -M -d /sambashares/jenkins_admin -s /usr/sbin/nologin -G sambashare jenkins_admin
create the user's home directory and set the ownership
    sudo mkdir /sambashares/jenkins_admin
    sudo chown jenkins_admin:sambashare /sambashares/jenkins_admin
everyone in the group can access files
    sudo chmod 2770 /sambashares/jenkins_admin
create the user on samba database. create a password when prompted.
    sudo smbpasswd -a jenkins_admin
enable the user on samba
    sudo smbpasswd -e jenkins_admin
configure samba shares
    sudo vi /etc/samba/smb.conf

    [jenkins_admin]
        path = /sambashares/jenkins_admin
        browseable = no
        read only = no
        force create mode = 0660
        force directory mode = 2770
        valid users = jenkins_admin
restart the services
    sudo systemctl restart smb.service
    sudo systemctl restart nmb.service

INSTALL JENKINS:

Install openjdk/jre on CentOs 7
   sudo yum install java-1.8.0-openjdk-devel
   java -version

Create a user "jenkins", create a password when prompted
   sudo useradd jenkins
   sudo passwd jenkins
   
Create a directory jenkins under /home/jenkins
    mkdir -p /home/jenkins/jenkins
Change the directory
    cd /home/jenkins/jenkins
Download jenkins.war
    wget http://mirrors.jenkins.io/war/latest/jenkins.war
Change permission
    chmod 744 jenkins.war
Configure sambashare password file
    vi /home/jenkins/.smbcredentials

        username=jenkins_admin
        password=<your samba password>
Install samba client
    sudo yum install samba-client
Install cifs utils
    sudo yum install cifs-utils
Mount the Samba share
    //<Samba Server IP>/jenkins_admin /home/jenkins/.jenkins/jobs cifs uid=jenkins,gid=jenkins,credentials=/home/jenkins/.smbcredentials 0 0
Start the Jenkins instance and note down the hash-key for creating the first admin user that appears on the console
    java -jar jenkins.war &
Launch the web-ui from a browser http://<this_server_ip>:8080
Use the hash-key from the console log to create the user and password for the admin user

Example: mount point
[root@ip-172-31-28-122 jobs]# cat /etc/fstab 
#
//172.31.28.122/jenkins_admin /root/.jenkins/jobs cifs uid=jenkins,gid=jenkins,credentials=/home/jenkins/.smbcredentials  0 0
#//172.31.28.122/jenkins_admin /home/jenkins/jenkins/ cifs uid=jenkins,gid=jenkins,credentials=/home/jenkins/.smbcredentials  0 0
UUID=0a56d206-2a6d-46e6-b65f-11b7052f72cf     /           xfs    defaults,noatime  1   1
```

# Slave
Follow the same process as detailed for primary jenkins master. Please ensure the admin user/password for both the instances (primary and secondary) are same.
Note: skip installing smbshare server 
```
Install openjdk/jre on CentOs 7
   sudo yum install java-1.8.0-openjdk-devel
   java -version

Create a user "jenkins", create a password when prompted
   sudo useradd jenkins
   sudo passwd jenkins
   
Create a directory jenkins under /home/jenkins
    mkdir -p /home/jenkins/jenkins
Change the directory
    cd /home/jenkins/jenkins
Download jenkins.war
    wget http://mirrors.jenkins.io/war/latest/jenkins.war
Change permission
    chmod 744 jenkins.war
Configure sambashare password file
    vi /home/jenkins/.smbcredentials

        username=jenkins_admin
        password=<your samba password>
Install samba client
    sudo yum install samba-client
Install cifs utils
    sudo yum install cifs-utils
Mount the Samba share
    //<EC2-Private-IP-Master/jenkins_admin /home/jenkins/.jenkins/jobs cifs uid=jenkins,gid=jenkins,credentials=/home/jenkins/.smbcredentials 0 0
Start the Jenkins instance and note down the hash-key for creating the first admin user that appears on the console
    java -jar jenkins.war &
Launch the web-ui from a browser http://<this_server_ip>:8080
Use the hash-key from the console log to create the user and password for the admin user

ISSUE:
mount error(115): Operation now in progress smb sambashare
Solution: Open security group inbound for smb service port 139 and 445

netstat -lnpt | grep smb
tcp        0      0 0.0.0.0:139             0.0.0.0:*               LISTEN      3092/smbd           
tcp        0      0 0.0.0.0:445             0.0.0.0:*               LISTEN      3092/smbd           
tcp6       0      0 :::139                  :::*                    LISTEN      3092/smbd           
tcp6       0      0 :::445                  :::*                    LISTEN      3092/smbd    
```

# HAPROXY
```
yum install haproxy
vi /etc/haproxy/haproxy.cfg

    // remove everything apart from the "global section" and add the following instead

    defaults
        log global
        maxconn 2000
        mode http
        option redispatch
        option forwardfor
        option http-server-close
        retries 3
        timeout http-request 10s
        timeout queue 1m
        timeout connect 10s
        timeout client 1m
        timeout server 1m
        timeout check 10s

    frontend ft_jenkins
        bind *:8080
        default_backend bk_jenkins
        reqadd X-Forwarded-Proto:\ http

    backend bk_jenkins
        server jenkins1 <primary_jenkins_ip>:8080 check
        server jenkins2 <secondary_jenkins_ip>:8080 check backup
```

# On the secondary jenkins master: write a cron-job to reload the configuration.
```
vi /home/jenkins/jenkins/jenkins_credentials
    admin:<password_of_admin_user>

chmod 600 /home/jenkins/jenkins/jenkins_credentials
cd /home/jenkins/jenkins
wget http://<jenkins_secondary_server_ip>:8080/jnlpJars/jenkins-cli.jar

vi /home/jenkins/jenkins/jenkins-reload.sh
    #!/bin/sh
    # check if primary is up or not
    java -jar /home/jenkins/jenkins/jenkins-cli.jar -s http://52.77.238.245:8080 -auth @/home/jenkins/jenkins/jenkins_credentials list-jobs
    PRIMARY_STATUS=$?

    echo ${PRIMARY_STATUS}

    if [ 0 == ${PRIMARY_STATUS} ]
    then
        echo "Primary is up"
        echo "Reloading secondary"
        java -jar /home/jenkins/jenkins/jenkins-cli.jar -s http://13.213.62.139:8080/ -auth @/home/jenkins/jenkins/jenkins_credentials reload-configuration
    else
        echo "Primary is down"
        echo "Skipping reload"
    fi

Change permissions
    chmod 700 /home/jenkins/jenkins/jenkins-reload.sh
Configure cron daemon
    vi /etc/cron.d/jenkins-reload
    */1 * * * * jenkins /bin/bash /home/jenkins/jenkins/jenkins-reload.sh
```
# Reference
https://skalable.net/blogs/technology/jenkins_ha.html
