## Implementing a basic web solution using WordPress ##

In this tutorial you will be shown how to prepare storage infrastructure on two Linux servers and implement a basic web solution using WordPress. WordPress is a free and open-source content management system written in PHP and paired with MySQL or MariaDB as its backend Relational Database Management System (RDBMS).

Project 6 consists of two parts:

Configure storage subsystem for Web and Database servers based on Linux OS. The focus of this part is to give you practical experience of working with disks, partitions and volumes in Linux.

Install WordPress and connect it to a remote MySQL database server. This part of the project will solidify your skills of deploying Web and DB tiers of Web solution.

As a DevOps engineer, your deep understanding of core components of web solutions and ability to troubleshoot them will play essential role in your further progress and development.

**Three-tier Architecture**
Generally, web, or mobile solutions are implemented based on what is called the **Three-tier Architecture**.

**Three-tier Architecture** is a client-server software architecture pattern that comprise of 3 separate layers.


![Three tier Architecture](./images/Three%20tier%20architecture.jpg)


**Presentation Layer (PL)**: This is the user interface such as the client server or browser on your laptop.
**Business Layer (BL)**: This is the backend program that implements business logic. Application or Webserver
**Data Access or Management Layer (DAL)**: This is the layer for computer data storage and data access. Database Server or File System Server such as FTP server, or NFS Server

In this project, you will have the hands-on experience that showcases **Three-tier Architecture** while also ensuring that the disks used to store files on the Linux servers are adequately partitioned and managed through programs such as gdisk and LVM respectively.

Requirements:

**Your 3-Tier Setup**
- A Laptop or PC to serve as a client

- An EC2 Linux Server as a web server (This is where you will install WordPress)

- An EC2 Linux server as a database (DB) server

**Note**: We are using RedHat OS for this project, you should be able to spin up an EC2 instance on your own. Also when connecting to RedHat you will need to use ec2-user user. Connection string will look like ec2-user@public-ip-address.

**Step 1**

## Launch an EC2 instance that will server as Webserver 

1. Launch an EC2 instance that will serve as "Web Server". Create 3 volumes in the same AZ as your Web Server EC2, each of 10 GiB.

    ![ Add EBS Volume to an EC2 instance](./images/Add%20EBS%20Volume%20to%20an%20EC2%20instance%20.jpg)


    ![ Attach the volumes](./images/attach%20volume.jpg)


2. Open up the Linux terminal to begin configuration, use `lsblk` command to inspect what block devices are attached to the server. Notice names of your newly created devices. All devices in Linux reside in /dev/ directory. Inspect it with ls /dev/ and make sure you see all 3 newly created block devices there – their names will likely be xvdf, xvdh, xvdg.

    ![ Inspect volumes](./images/inspect%20devices%20attached.jpg)

3. Use `df -h` command to see all mounts and free space on your server.

    ![ Mounts and free space](./images/check%20all%20mounts%20and%20freespace.jpg)

4. Use the fdisk utility to create a single partition on each of the 3 disks `sudo fdisk /dev/xvdf`. Repeat the steps for xvdg and xvdh

    ![Create a new partition](./images/create%20new%20partition%20xvdf1.jpg)

5. Use `lsblk` utility to view the newly configured partition on each of the 3 disks.

    ![Check configured partition](./images/view%20newly%20created%20partitions.jpg)

6. Install lvm2 package using `sudo yum install lvm2`. Run `sudo lvmdiskscan` command to check for available partitions.

    ![Install lvm2](./images/install%20lvm2%20package.jpg)

    ![lvmdiskscan](./images/lvmdiskscan.jpg)

7. Use pvcreate utility to mark each of 3 disks as physical volumes (PVs) to be used by LVM

    `sudo pvcreate /dev/xvdf1`
    `sudo pvcreate /dev/xvdg1`
    `sudo pvcreate /dev/xvdh1`

      ![pvcreate](./images/mark%20each%20of%20the%20volumes%20as%20physical%20volume.jpg)

8. Verify that your Physical volume has been created successfully by running `sudo pvs`

    ![verify physical volumes](./images/verify%20physical%20volume.jpg)

9. Use vgcreate utility to add all 3 PVs to a volume group (VG). Name the VG webdata-vg - `sudo vgcreate webdata-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1`

   ![volume group](./images/use%20vg%20create%20to%20add%20all%203%20PV.jpg) 

10. Verify that your VG has been created successfully by running `sudo vgs`

    ![verify volume group](./images/verify%20volume%20group.jpg) 

11. Use lvcreate utility to create 2 logical volumes. apps-lv (Use half of the PV size), and logs-lv Use the remaining space of the PV size. NOTE: apps-lv will be used to store data for the Website while, logs-lv will be used to store data for logs.

    `sudo lvcreate -n apps-lv -L 14G webdata-vg`
    `sudo lvcreate -n logs-lv -L 14G webdata-vg`

    ![create two logical volumes](./images/create%20two%20logical%20volumes.jpg)

12. Verify that your Logical Volume has been created successfully by running `sudo lvs`

    ![Verify logical volumes](./images/verify%20logical%20volume.jpg)

13. Verify the entire setup

    `sudo vgdisplay -v #view complete setup - VG, PV, and LV`
    `sudo lsblk`

    ![Verify entire setup](./images/verify%20entire%20setup.jpg)

14. Use mkfs.ext4 to format the logical volumes with ext4 filesystem

    `sudo mkfs -t ext4 /dev/webdata-vg/apps-lv`
    `sudo mkfs -t ext4 /dev/webdata-vg/logs-lv`

    ![Format logical volumes](./images/format%20the%20logical%20volumes%20for%20the%20database%20server.jpg)

15. Create /var/www/html directory to store website files
    `sudo mkdir -p /var/www/html`

    ![Create /var/www/html directory](./images/create%20varhtml%20.jpg)

16. Create /home/recovery/logs to store backup of log data

    `sudo mkdir -p /home/recovery/logs`

    ![Create /home/recovery/logs to store backup of log data](./images/create%20home%20logs.jpg)

17. Mount /var/www/html on apps-lv logical volume

    `sudo mount /dev/webdata-vg/apps-lv /var/www/html/`

    ![Mount /var/www/html on apps-lv logical volume](./images/Mount%20varhtml.jpg)

18. Use rsync utility to backup all the files in the log directory /var/log into /home/recovery/logs (This is required before mounting the file system)

    `sudo rsync -av /var/log/. /home/recovery/logs/`

    ![Use rsync utility to backup all the files in the log directory](./images/Use%20rsync%20utility%20to%20backup%20all%20the%20files.jpg)

19. Mount /var/log on logs-lv logical volume. (Note that all the existing data on /var/log will be deleted. That is why step 15 above is very important)

      `sudo mount /dev/webdata-vg/logs-lv /var/log`

      ![Mount var on logs-lv logical volume](./images/Mount%20varlog%20on%20logs-lv%20logical%20volume.jpg)


20. Restore log files back into /var/log directory

    `sudo rsync -av /home/recovery/logs/. /var/log`

    ![Restore log files back](./images/Restore%20log%20files%20back%20into%20varlog.jpg)

21. Update /etc/fstab file so that the mount configuration will persist after restart of the server.

    The UUID of the device will be used to update the /etc/fstab file;

    `sudo blkid`

    ![The UUID of the device will be used to update the /etc/fstab file](./images/The%20UUID%20of%20the%20device%20will%20be%20used%20to%20update%20the%20fstab.jpg)

    `sudo vi /etc/fstab`

    Update /etc/fstab in this format using your own UUID and rememeber to remove the leading and ending quotes.

    ![Update /etc/fstab in this format using your own UUID and rememeber to remove the leading and ending quotes](./images/Update%20etcfstab%20in%20this%20format%20using%20your%20own%20UUID.jpg)

    1.  Test the configuration and reload the daemon

         `sudo mount -a`
        
        `sudo systemctl daemon-reload`

        ![Test config](./images/test%20the%20config%20and%20reload.jpg)

    2.  Verify your setup by running `df -h`, output must look like this:

        ![Verify your setup](./images/Verify%20your%20setup%20by%20running%20df%20-h.jpg)


## Step 2 


**Prepare the Database Server**

1. Launch a second RedHat EC2 instance that will have a role – ‘DB Server’

2. Repeat the same steps as for the Web Server, but instead of `apps-lv` create `db-lv` and mount it to `/db` directory instead of `/var/www/html/`.


    ![Verify db setup](./images/verify%20database%20server%20setup.jpg)


## Step 3

**Install WordPress on your Web Server EC2**

1. Update the repository

    `sudo yum -y update`

    ![Update the repository](./images/Update%20the%20repository.jpg)

2. Install wget, Apache and it’s dependencies

    `sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json`

    ![Install wget, Apache and it’s dependencies](./images/Install%20wget%2C%20Apache%20and%20it%E2%80%99s%20dependencies.jpg)

3. Start Apache

    `sudo systemctl enable httpd` 
    
    `sudo systemctl start httpd`

    ![Start Apache](./images/start%20apache.jpg)

4. To install PHP and it’s dependencies

    `sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm`
    
    `sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm`

    `sudo yum module list php`

    `sudo yum module reset php`

    `sudo yum module enable php:remi-7.4`

    `sudo yum install php php-opcache php-gd php-curl php-mysqlnd`

    `sudo systemctl start php-fpm`

    `sudo systemctl enable php-fpm`

    `sudo setsebool -P httpd_execmem 1`



    ![Install PHP and its dependencies](./images/install%20PHP%20and%20it%E2%80%99s%20dependencies.jpg)



5. Restart Apache

    `sudo systemctl restart httpd`

    ![Restart Apache](./images/restart%20apache.jpg)


6. Download wordpress and copy wordpress to var/www/html

    `mkdir wordpress`

    `cd   wordpress`

    `sudo wget http://wordpress.org/latest.tar.gz`

    `sudo tar xzvf latest.tar.gz`

    `sudo rm -rf latest.tar.gz`

    `sudo cp wordpress/sudo wp-config-sample.phpwordpress/wp-config.php`

    `sudo cp -R wordpress /var/www/html/`

    ![Download wordpress and copy wordpress to var/www/html](./images/Download%20wordpress%20and%20copy%20wordpress%20to%20var.jpg)

7. Configure SELinux Policies

    `sudo chown -R apache:apache /var/www/html/wordpress`

    `sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R`

    `sudo setsebool -P httpd_can_network_connect=1`

    ![Configure SELinux Policies](./images/Configure%20SELinux%20Policies.jpg)



## Step 4

1. Install MySQL on your DB Server EC2**


    `sudo yum update`
    `sudo yum install mysql-server`


    ![Install MySQL on your DB Server EC2](./images/install%20mysqlserver%20on%20dbserver.jpg)

2. Verify that the service is up and running by using `sudo systemctl status mysqld`, if it is not running, restart the service and enable it so it will be running even after reboot:

    `sudo systemctl restart mysqld`

    `sudo systemctl enable mysqld`

    ![Verify that the service is up and running](./images/verify%20that%20mysql%20services%20are%20running.jpg)


## Step 5 ## 

Configure DB to work with WordPress

`sudo mysql`

`CREATE DATABASE wordpress;`

`CREATE USER `myuser`@`<Web-Server-Private-IP-Address>` IDENTIFIED BY 'mypass';`

`GRANT ALL ON wordpress.* TO 'myuser'@'<Web-Server-Private-IP-Address>';`

`FLUSH PRIVILEGES;`

`SHOW DATABASES;`

`exit`

![Configure DB to work with WordPress](./images/Configure%20DB%20to%20work%20with%20WordPress.jpg)

## Step 6 ## 

Configure WordPress to connect to remote database.

**Note**: Open MySQL port 3306 on DB Server EC2. For extra security, to allow access to the DB server ONLY from your Web Server’s IP address, so in the Inbound Rule configuration specify source as /32.

1. Install MySQL client and test that you can connect from your Web Server to your DB server by using mysql-client

    `sudo yum install mysql`

    `sudo mysql -u myuser -p -h <DB-Server-Private-IP-address>`

    ![Configure WordPress to connect to remote database](./images/install%20mysql.jpg)


    ![Connect to remote database](./images/login%20remotely%20to%20mysqlserver.jpg)


2. Change permissions and configuration so Apache could use WordPress:

    Here we need to create a configuration file for wordpress in order to point client requests to the wordpress directory.

    `sudo vi /etc/httpd/conf.d/wordpress.conf`

    ![create a configuration file for wordpress](./images/create%20a%20configuration%20file%20for%20wordpress.jpg)

    copy and paste the lines below:

        <VirtualHost *:80>
        ServerAdmin myuser@3.88.215.221
        DocumentRoot /var/www/html/wordpress

        <Directory "/var/www/html/wordpress">
        Options Indexes FollowSymLinks
        AllowOverride all
        Require all granted
        </Directory>

        ErrorLog /var/log/httpd/wordpress_error.log
        CustomLog /var/log/httpd/wordpress_access.log common
        </VirtualHost>

    ![copy and paste](./images/and%20copy%20and%20paste%20the%20lines%20below.jpg)

3. Restart Apache 

    `sudo systemctl restart httpd`

    ![restart apache](./images/restart%20apache.jpg)

4. Edit the wp-config file

    `sudo vi /var/www/html/wordpress/wp-config.php`

    add the following values

        define('DB_NAME', 'wordpress');
        define('DB_USER', 'myuser');
        define('DB_PASSWORD', 'mypass');
        define('DB_HOST', '<db-Server-Private-IP-Address>');
        define('DB_CHARSET', 'utf8mb4');
        define('DB_COLLATE', '');

    ![add values](./images/add%20values.jpg)

5. Configure SELinux for wordpress

    `sudo semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/html/wordpress/.*?"`

    ![semange command](./images/semanage%20command.jpg)


 6. Try to access from your browser the link to your WordPress

    `http://<Web-Server-Public-IP-Address>/`

    ![Access over the browser](./images/try%20to%20access%20from%20the%20browser.jpg)
