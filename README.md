# How to run Microsoft SQL Server 2019 in a Docker container (running on an Ubuntu server) 

### Author: [Olav Tollefsen](https://www.linkedin.com/in/olavtollefsen/)

# Introduction

This article documents the process of how to setup and run Microsoft SQL Server 2019 in a Docker container running on an Ubuntu server.

# What you need

You will one machine (physical or virtual) running Linux. We will be using Ubuntu Server 18.04.4 LTS. This article assumes that the Linux server have mounted one data disk (in addition to the operating system disk).

# Installing Ubuntu Server 18.04.4 LTS

Don't select to install Docker as a part of the operating system installation, since we want to have full control over the version we will install.

# Prepare the server

## Update operating system

After installing the operating system, make sure it's updated.

```
$ sudo apt-get update
$ sudo apt-get upgrade
```

## Install Docker

### Set up the repository

Install packages to allow apt to use a repository over HTTPS:

```
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```

Add Docker’s official GPG key:

```
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

Verify that you now have the key with the fingerprint 9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88, by searching for the last 8 characters of the fingerprint.

```
$ sudo apt-key fingerprint 0EBFCD88
```

Set up the stable repository:

```
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

### Install a specific version of Docker (19.03.8)

Update the apt package index

```
$ sudo apt-get update
```

 List the versions available in your repo:

```
$ apt-cache madison docker-ce
```

Install Docker version 19.03.8:

```
$ sudo apt-get install docker-ce=5:19.03.8~3-0~ubuntu-bionic docker-ce-cli=5:19.03.8~3-0~ubuntu-bionic containerd.io
```

To be able to issue Docker commands without sudo:
```
$ sudo usermod -aG docker <your-username>
```

Logout from the current user and login again for the above command to take effect.

# Prepare the filesystem for the SQL Server data files

List the disks on the server:

```
$ sudo lshw -class disk -short
```

The output should look something like this:
```
H/W path      Device       Class      Description
=================================================
/0/1/0.0.0    /dev/cdrom   disk       Virtual CD/ROM
/0/2/0.0.0    /dev/sda     disk       136GB Virtual Disk
/0/3/0.0.0    /dev/sdb     disk       136GB Virtual Disk
```

In this case the /dev/sda is refering to the operating system disk and /dev/sdb is the data disk.

Warning! Make sure to understand what disks you have mounted to make sure you are not deleting data unintentionally.

## Format the data disk

This assumes that the data disk is blank.

Create a new partition by issuing the following command:
```
$ sudo fdisk /dev/sdb
```

Enter 'n' at the fdisk command prompt to add a new partition. Select defaults for all the inputs and remember to select "w" when returning to the fdisk command prompt in order to write table to disk and exit.

Format the partition as XFS:

```
sudo mkfs.xfs -L sqlserver /dev/sdb1
```

## Mount the new filesystem

Create the mountpoint

```
$ sudo mkdir /mnt/sqlserver
```

Mount the new filesystem

```
$ sudo mount -t xfs /dev/sdb1 /mnt/sqlserver
```

## Make sure new filesystem is mounted at boot

Get UUID for newly created file system

```
$ sudo blkid /dev/sdb1
```

Add this to "/etc/fstab":

```
UUID=<UUID> /mnt/sqlserver xfs defaults 1 1
```

## Create directories for SQL Server on host volume

```
sudo mkdir /mnt/sqlserver/data
sudo mkdir /mnt/sqlserver/log
sudo mkdir /mnt/sqlserver/secrets
```

# Run SQL Server 2019 in a Docker container

```
$ docker run --env 'ACCEPT_EULA=Y' --env 'MSSQL_SA_PASSWORD=<YourStrong!Passw0rd>' --env MSSQL_AGENT_ENABLED=True -p 1433:1433 -v /mnt/sqlserver/data:/var/opt/mssql/data -v /mnt/sqlserver/log:/var/opt/mssql/log -v /mnt/sqlserver/secrets:/var/opt/mssql/secrets -u 0:0 -d --restart always mcr.microsoft.com/mssql/server:2019-latest
```

Now you should be able to connect to your server by using SQL Server Management Studio

