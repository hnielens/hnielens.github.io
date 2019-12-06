---
layout: post
title: How to gear up a CA demo server on a cloud VM
---

This tech note might help you with what steps to take to gear up a CA demo server on a IBM Cloud VM (or any other Cloud)

[This site helped me to get it right](https://www.sslshopper.com/ssl-certificate-tools.html){:target='_blank'}.

# Set up an IBM Cloud VM for CA 11.1.4 walkthrough

## Order a server

- Have both a private and public address
- Private network policy (allow_all, allow_outbound, allow_ssh), Public Network Policy (allow_outbound)

## Prepare CentOS base

### Install ibmcloud cli
- Get and install available updates
`yum update`
`yum upgrade`
- Get and install the ibmcloud cli
`curl -fsSL https://clis.cloud.ibm.com/install/linux | sh`

### Install Docker CE for CentOS
https://docs.docker.com/v17.09/engine/installation/linux/docker-ce/centos/

### Set up Cloud Object Storage plugin
**Note** that you only need this step when you have uploaded the Cognos software to a COS service
https://cloud.ibm.com/docs/cloud-object-storage-cli-plugin?topic=cloud-object-storage-cli-ic-cos-cli
- Log in to ibmcloud
`ibmcloud login -sso`
- Install the cos plugin
`ibmcloud plugin install cloud-object-storage`
- Configure cos with the bucket's crn (found in the bucket properties)
`ibmcloud cos config crn`
- Configure the hmac authentication using credentials (access_key_id/secret_access_key) generated for the cos service
`ibmcloud cos config hmac`
- Make sure hmac is the default active authentication method
`ibmcloud cos config auth`
- Configure where downloaded files will land (your default download folder)
`ibmcloud cos config ddl`


### Set up and test X11 forwarding
Note that in this section we only make sure the 64bit X11 config is working correctly. We will set up the 32bit side of things later in the section where we are going to install Cognos components with the 32bit installer. This document might help for configuring x11.
https://www.businessnewsdaily.com/11035-how-to-use-x11-forwarding.html
- Check the `/etc/ssh/sshd_config` file
- Make sure e.g. xauth is installed
`yum update`
`yum install xauth`
- Optionally test with xeyes. First make sure you ssh-ed in into your server with the -Y handle and you did not get specific errors.
`yum update`
`yum install xeyes`
`xeyes`

## Set up Apache Directory Server
- Install java
`yum install java`
- Get the latest download link for the 64bit binary installer for Apache DS
http://directory.apache.org/apacheds/download/download-linux-bin.html
- Download and install the binary installer (in a well chosen directory)
`wget "use_latest_download_URL"`
`chmod 755 ./installer_file_name.bin`
`./installer_file_name.bin`
- For good measure chown the apacheds dirs BEFORE starting the service and maybe have wrapper.conf point (absolute ref) to your java executable
`chown apacheds:apacheds /var/lib/apacheds_version_you_installed/*`
`chown apacheds:apacheds /opt/apacheds_version_you_installed/*`
- start Apache Directory Server with user apacheds
`/etc/init.d/apacheds_version_you_installed-default start`
- Check if you can connect to the Directory Server using Directory Studio and if not maybe you need to repair the default instance
`/etc/init.d/apacheds_version_you_installed-default repair`
- Change the password for uid=admin,ou=system
- Remove the installer --so the bin file you downloaded earlier-- to keep your system neat

## Deploy  Developer-C Edition 11.1.4.4 container
- If not yet done, register for Docker Hub
https://hub.docker.com/
- Look for Db2 Developer-C, add to basket and check out
- In your _My Content_ (in your account) look for Db2 Developer-C and _Setup_ for instructions on how to get and deploy the Db2 container
- Make sure to login to docker hub on the server before you start with retrieval, configuration and run of the container:
`docker login`
- Make sure you create a Db2 data folder on your hostname for persisting the db data, e.g.:
`mkdir /root/db2data`
- Make sure that you have a customised `.env_lis` as described as in the setup instructions, e.g.:
`vi /root/config/.db2_env_list`
- Here is an example command to run the  container
`docker run -h db2server --name db2server --restart=always --detach --privileged=true -p 50000:50000 -p 55000:55000 --env-file /root/config/.db2_env_list -v /root/db2data/:/database store/ibmcorp/db2_developer_c:11.1.4.4-x86_64`

## Install and Configure Cognos Analytics
### Get the software and install
- Make sure you have the installation files on your system, e.g. by getting them from your cloud object storage bucket
`ibmcloud login --sso`
`ibmcloud cos download -bucket bucket_name_here -key file_name_here`
- Install Cognos Analytics using the installer and the payload zip
- Do not forget to put the Db2 JDBC driver files in the `../drivers` folder, you can find them in the Db2 container. Try something like this:
`docker exec -ti db2server bash -c "cp /database/config/db2inst1/sqllib/java/db2jcc* /database/."`
`cp /root/db2data/db2jcc* /opt/ibm/cognos/analytics/drivers/.`

### Configure Cognos Analytics
Use cogconfig.sh to configure the Cognos Analytics deployment
`/opt/ibm/cognos/analytics/bin64/cogconfig.sh`
- Change hostname and full qualified domain name mentions to the host ip
- Configure the content store to point to the database running in the Db2 container
- Make sure to switch the _Report Server execution mode_ parameter to _64-bit_ (under _Environment_), otherwise you will need to get 32-bit libraries installed on CentOS.

## Set up SFTP and SFTP users
This is a good base article:
https://www.vultr.com/docs/setup-sftp-only-user-accounts-on-centos-7

### Set up SFTP for exchanging import/export of deployments

- Create a group for the sftp users e.g. _sftpusers_
`groupadd sftpusers`
- Edit the `/etc/ssh/sshd_config` file for use with `internal-sftp`
`cp /etc/ssh/sshd_config /etc/ssh/sshd_config._backup_suffix
`vi /etc/ssh/sshd_config`
- Find the line:
`Subsystem sftp /usr/libexec/openssh/sftp-server`
and replace it with (or comment it and add):
`Subsystem sftp internal-sftp`
- At the bottom of the file add following directives
`Match Group sftpusers`
`X11Forwarding no`
`AllowTcpForwarding no`
`ChrootDirectory %h`
`ForceCommand internal-sftp`
Notice that we set the ChrootDirectory to match the home directory of each respective SFTP user (`%h`)

### Make the CA deployment directory accessible to the SFTP users
- Change the ownership and permissions of the folder:
`chown root:sftpusers /opt/ibm/cognos/analytics/deployments`
`chmod 775 /opt/ibm/cognos/analytics/deployments`


### Set up SFTP users (do below for every sftp user)
- Create each sftp user, add them to the _sftpusers_ group, and make sure they can not login on the server using `ssh`
`useradd sftp_user -g sftpusers -s /usr/sbin/nologin`
- Change the user pw:
`passwd sftpuser`
- Make sure the root of the home directory --which is the ChrootDirectory we defined earlier-- is not accessible by the user:
`chown root:root /home/sftpuser`
- Make sure all hidden files in the root directory of the sftpuser are read/execute only:
`chmod -R 666 .*`
- Make a _deployment_ directory in the sftpuser's home directory and make sure the user owns it:
`mkdir deployment`
`chown sftpuser. deployment`
-Mount the CA deployment directory on the ftpuser's deployment directory:
`mount -o bind /opt/ibm/cognos/analytics/deployment /home/ftpuser/deployment`
