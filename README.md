# Nagios Installation Guide  

The Nagios Installation Guide provides step-by-step instructions for setting up Nagios, an open-source monitoring tool that helps track the health and performance of IT infrastructure. The guide covers downloading, installing, and configuring Nagios on various operating systems, along with setting up plugins, services, and alert notifications to monitor hosts, network devices, and services.

### This guide covers how to:  

1. Install Nagios on Ubuntu
2. Add a Remote Host for Monitoring in Nagios

## 1. Install Nagios on Ubuntu
### 1.1 Update Your System
Start by updating the package list and upgrading the installed packages:

```bash
sudo apt update && sudo apt upgrade -y
```
### 1.2 Install Prerequisites
Nagios requires some dependencies, including Apache, PHP, and additional libraries:

```bash
sudo apt install wget build-essential apache2 php libapache2-mod-php php-gd libgd-dev unzip -y
```

### 1.3 Create a Nagios User and Group
Create a Nagios user and group, and add the Apache user to the Nagios group:

```bash
sudo useradd nagios
```
```bash
sudo groupadd nagcmd
```
```bash
sudo usermod -aG nagcmd nagios
```bash
sudo usermod -aG nagcmd www-data
```

### 1.4 Download and Install Nagios Core
Download Nagios Core:

```bash
cd /tmp
```
```bash
wget https://assets.nagios.com/downloads/nagioscore/releases/nagios-4.4.6.tar.gz
```
```bash
tar -zxvf nagios-4.4.6.tar.gz
```
```bash
cd nagios-4.4.6
```

Compile and Install Nagios:

```bash
sudo ./configure --with-command-group=nagcmd
```
```bash
sudo make all
```
```bash
sudo make install
```
```bash
sudo make install-commandmode
```
```bash
sudo make install-init
```
```bash
sudo make install-config
```
```bash
sudo make install-webconf
```

### 1.5 Install Nagios Plugins
Download and Install Plugins:

```bash
cd /tmp
```
```bash
wget https://nagios-plugins.org/download/nagios-plugins-2.3.3.tar.gz
```
```bash
tar -zxvf nagios-plugins-2.3.3.tar.gz
```
```bash
cd nagios-plugins-2.3.3
```
```bash
sudo ./configure --with-nagios-user=nagios --with-nagios-group=nagios --with-openssl
```
```bash
sudo make
```
```bash
sudo make install
```

### 1.6 Configure Apache for Nagios
Create a Nagios Admin User:

```bash
sudo htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin
```
(You will be prompted to enter a password for the nagiosadmin user.)

Enable Apache Modules and Restart Apache:

```bash
sudo a2enmod rewrite cgi
```
```bash
sudo systemctl restart apache2
```

### 1.7 Start and Enable Nagios Service
Start Nagios and Enable it to Start on Boot:

```bash
sudo systemctl start nagios
```
```bash
sudo systemctl enable nagios
```

Verify Nagios is Running:

Open a web browser and navigate to http://<your-server-ip>/nagios. Log in with the username nagiosadmin and the password you set.

## 2. Add a Remote Host for Monitoring
To monitor another machine (let's call it Remote Host), you need to install Nagios NRPE (Nagios Remote Plugin Executor) on that machine and configure it.

### 2.1 Install NRPE on the Remote Host
Install NRPE and Nagios Plugins:

On the Remote Host (another Ubuntu server), run:

```bash
sudo apt update
```
```bash
sudo apt install nagios-nrpe-server nagios-plugins -y
```

Configure NRPE:

Edit the NRPE configuration file:

```bash
sudo nano /etc/nagios/nrpe.cfg
```

Add the IP address of your Nagios Server to the allowed_hosts directive:

```bash
allowed_hosts=<Nagios_Server_IP>,121.0.0.1
```
Save and exit the file.  

Restart NRPE Service:

```bash
sudo systemctl restart nagios-nrpe-server
```
```bash
sudo systemctl enable nagios-nrpe-server
```

### 2.2 Configure Nagios Server to Monitor Remote Host
Create a Configuration File for the Remote Host:

On the Nagios Server:

```bash
sudo nano /usr/local/nagios/etc/servers/<remote_host_name>.cfg
```

Replace <remote_host_name> with the actual name of the remote host.

Define the Remote Host and Services to Monitor:

Example configuration:

```bash
define host {
    use             linux-server
    host_name       <remote_host_name>
    alias           <remote_host_alias>
    address         <Remote_Host_IP>
    max_check_attempts 5
    check_period    24x1
    notification_interval 30
    notification_period  24x1
}

define service {
    use                     generic-service
    host_name               <remote_host_name>
    service_description     PING
    check_command           check_ping!100.0,20%!500.0,60%
}

define service {
    use                     generic-service
    host_name               <remote_host_name>
    service_description     Root Partition
    check_command           check_nrpe!check_disk
}

define service {
    use                     generic-service
    host_name               <remote_host_name>
    service_description     Current Users
    check_command           check_nrpe!check_users
}

define service {
    use                     generic-service
    host_name               <remote_host_name>
    service_description     Total Processes
    check_command           check_nrpe!check_procs
}

define service {
    use                     generic-service
    host_name               <remote_host_name>
    service_description     Load
    check_command           check_nrpe!check_load
}
```

Include the New Configuration in Nagios:  

Edit the main Nagios configuration file to include your new host configuration:  

```bash
sudo nano /usr/local/nagios/etc/nagios.cfg
```

Add the following line:  

```bash
cfg_file=/usr/local/nagios/etc/servers/<remote_host_name>.cfg
```

### 2.3 Define NRPE Commands in Nagios  
You need to define the NRPE commands in the commands configuration file. Usually, this is done in the commands.cfg file.  

### 2.3.1 Open the commands.cfg File
```bash
sudo nano /usr/local/nagios/etc/objects/commands.cfg
```

### 2.3.2 Add the NRPE Command Definitions
Add the following command definitions to the commands.cfg file:

```bash
define command {
    command_name    check_nrpe
    command_line    $USER1$/check_nrpe -H $HOSTADDRESS$ -c $ARG1$
}

define command {
    command_name    check_users
    command_line    $USER1$/check_nrpe -H $HOSTADDRESS$ -c check_users
}

define command {
    command_name    check_load
    command_line    $USER1$/check_nrpe -H $HOSTADDRESS$ -c check_load
}

define command {
    command_name    check_disk
    command_line    $USER1$/check_nrpe -H $HOSTADDRESS$ -c check_disk
}

define command {
    command_name    check_procs
    command_line    $USER1$/check_nrpe -H $HOSTADDRESS$ -c check_procs
}
```

The check_nrpe command is the generic NRPE command used to run a specified command on the remote host.
The other commands (check_users, check_load, etc.) are specific commands to check the number of users, load, disk usage, and processes on the remote host.
### 2.3.3 Save and Exit
Save the file and exit the editor.  

### 2.4 Verify the Configuration  

Before restarting Nagios, verify that your configuration is correct:

```bash
sudo /usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg
```

### 2.5 Restart Nagios
If the configuration verification passes without errors, restart Nagios:

```bash
sudo systemctl restart nagios
```

### 2.6 Check Nagios Web Interface
Go back to the Nagios web interface at http://<your-server-ip>/nagios. You should now see the Remote Host and the services you configured for monitoring. 
