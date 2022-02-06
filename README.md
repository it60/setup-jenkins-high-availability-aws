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
    
Shared disk
2Install openjdk/jre on CentOs 7
   34  sudo yum install java-1.8.0-openjdk-devel
   35  java -version

3Create a user "jenkins", create a password when prompted
   36  useradd jenkins
   37  passwd jenkins
   
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
