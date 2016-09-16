# AWS DNS Updater for LINUX

Every time you start an Instance in AWS you will be assigned a dynamic public IP, which remains associated with the instance until it is stopped or terminated. 
Usually for cost purposes we shutdown our instances after the demonstrations, in the evening or on weekends. When we power them back it would be good to have the same IP to avoid changing our Remote Desktop connection settings, FTP connections, URL Bookmarks etc.
This document helps you setting up a couple of scripts to automatically update Route 53 entries, using the AWS APIs. 
This document relies on the CentOS procedure, but the scripts and the procedure should adapt to other distributions without major issues. 


## Pre-Requisites

This document procedure requires:
-	Linux OS (CentOS used as a base in this document)
-	root access

## Installation
### OS Packages Intallation
To execute the automatic update, the scripts rely on command line utilities that must be installed in the OS. 
In CentOS 7 we use the package manager to install these tools as follows

```
yum install bind-utils
yum install git
yum install python
```

Note: Some of the packages might be already installed in your installation. The install command will return a message like “Package git-1.8.3.1-6.el7_2.1.x86_64 already installed and latest version”. This means it is installed and it is fine.

Check the python version installed to make sure we have Python 2.7.x installed. 

```
python --version
Python 2.7.5
```

### Install AWS CLI
The AWS API we will be using is a Command Line Interface (CLI) command and this API is written in Python. 
Python uses a package manager called PIP which makes it easier to install new applications and libraries into python. 

To install PIP is is necessary to download the installer as follows
```
curl -O https://bootstrap.pypa.io/get-pip.py
```
Execute the installer
```
python get-pip.py
```
Install the AWS CLI
```
pip install awscli --ignore-installed six
```
Check if AWS CLI is installed correctly
```
> aws --version
aws-cli/1.10.65 Python/2.7.5 Linux/3.10.0-327.28.3.el7.x86_64 botocore/1.4.55
```

AWS documents the procedure in the following link: [http://cs.aws.amazon.com/cli/latest/userguide/installing.html](http://cs.aws.amazon.com/cli/latest/userguide/installing.html)


###Install DNS Update Shell Script
Create a folder for the script
```
mkdir /opt/scripts
cd /opt/scripts
```

Download the shell script from the GIT repository
```
git clone https://github.com/hugomcruz/aws-dns-updater.git
```

Update the shell scrip
```
vi /opt/scripts/aws-dns-updater/updatedns.sg
```
Update the variables in the script referenced in bold. 

```
#!/bin/bash

# SET THE AWS KEYS
export AWS_SECRET_ACCESS_KEY=xxxxxxxxxxxxxxxxxxxx
export AWS_ACCESS_KEY_ID=xxxxxxxxxxxx

# (optional) You might need to set your PATH variable at the top here
# depending on how you run this script
#PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Hosted Zone ID e.g. BJBK35SKMM9OE
ZONEID="xxxx"

# The CNAME you want to update e.g. hello.example.com
RECORDSET="demo.domain.com"
```

For the AWS security keys please refer to the IAM service documentation. 

The ZONE ID can be obtained from the Route 53 console as below. 
If you do not have a DNS A record already created, you can create one in the Route53 console.

Important:
-	Make sure the Type is “A – IPv4 address”
-	Set any IP to create the record. You can set a private IP like 192.168.x.x


Execute the script
```
./updatedns.sh
```

If there are no errors it should have been executed successfully. Check the log file and it should look like:
```
IP has changed to 54.251.135.115
{
    "ChangeInfo": {
        "Status": "PENDING", 
        "Comment": "Auto updating @ Fri Sep 16 11:00:24 SGT 2016", 
        "SubmittedAt": "2016-09-16T03:00:26.598Z", 
        "Id": "/change/C23P5IP3CZHQ2E"
    }
}
```

Check the Route 53 console to confirm that the IP was updated. 

##Run script on server startup

Edit the startup  file (this changes from Linux distribution, so adapt to your distribution as this is the CentOS)
```
vi /etc/rc.d/rc.local
```
Append the following to the end of the file
```
#Added by xxx on xx/xx/xxxx to update external DNS on Route 53
/opt/scripts/aws-dns-updater/updatedns.sh
```
Make sure the rc.local is executable
```
chmod +x /etc/rc.d/rc.local
```

To test if the installation is successfully:
-	shutdown the Instance
-	wait a few seconds
-	start again from the console
-	after start check the Route53 console to confirm that the DNS was updated

If the DNS was updated you can start using the DNS name instead of the IP address. 


##Troubleshooting

####DNS does not update and there is no error
While testing the update script, if the AWS call fails it might not update the DNS again, at least until the IP changes.
The script creates a temporary file with the IP address from the last change on the script folder:
-	update-route53.ip

If you have errors while testing the update script, you need to delete this file. The script uses this file to compare the current public IP with the last time the script executed to avoid calling the API. 


####DNS was not updated on server startup
By executing the script on the serve startup this means it will only be executed once. If there is an error determining the external IP or calling the AWS API, it will not be retried. This is a rare occurrence but it might happen. 

A work around is to also calling the script in a cron job. The script has a mechanism to call avoid calling the AWS IP when the IP is still the same, so it is fine to runt his every couple of minutes to ensure that you do not have to fix manually. 







