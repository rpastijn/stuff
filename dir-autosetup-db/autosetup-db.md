# Changes to Vijays image #

## Description ##

The following steps need to be executed on the MT-WS image created by Vijay before the image can be uploaded to the OCI Marketplace.

## Add systemd service for auto start ##

(section based on https://oracle-base.com/articles/linux/linux-services-systemd#creating-linux-services)

The following commands need to be executed as root user:

````
$ <copy>sudo -s</copy>
````

Add new file to the /lib/systemd/system directory called oradb.service:

````
# <copy>vi /lib/systemd/system/oradb.service</copy>
````

Copy the following content into the new file:

````
<copy>[Unit]
Description=Startup database and initiate services
After=syslog.target network.target

[Service]
LimitMEMLOCK=infinity
LimitNOFILE=65535

Type=idle
RemainAfterExit=yes
User=oracle
Group=oinstall
Restart=no
ExecStart=/bin/bash -c '/home/oracle/scripts/start_all.sh'
ExecStop=/bin/bash -c '/home/oracle/scripts/stop_all.sh'

[Install]
WantedBy=multi-user.target
````

Save the file and reload systemd so it can see the new service.

````
# <copy>systemctl daemon-reload</copy>
````

We can now enable the service for the next reboot:

````
# <copy>systemctl enable oradb.service</copy>

Created symlink from /etc/systemd/system/multi-user.target.wants/oradb.service to /usr/lib/systemd/system/oradb.service.

````

After these steps, the following commands will be executed as the `oracle` user:

````
# <copy>su - oracle</copy>
````

## Remove all hand-made scripts from all locations ##

Remove all scripts that are added in existing files en locations like:

- $ORACLE_HOME/network/admin/a.sh
- $ORACLE_HOME/network/admin/createCDBs.sh
- $ORACLE_HOME/labs/envprep.sh
- $ORACLE_HOME/labs.zip

Delete the line that calls `a.sh` inside the file

- $ORACLE_HOME/bin/dbstart 

## Set PDBs to autostart ##

Make sure the PDBs inside the 2 CDBs automatically start when the CDBs are started:

````
$ <copy>. oraenv</copy>
````

Fill in the name of the database you want to change (like CDB1 or CDB2). Then startup all PDBs in the CDB:

````
SQL> <copy>alter pluggable database all open;</copy>

Pluggable database altered.
````

Make sure the PDBs are open the next time the CDB starts by executing the following command:

````
SQL> <copy>alter pluggable database all save state;</copy>

Pluggable database altered.
````

## Assumed contents .ora files ##

Make sure the following two files have the contents as described below. The startup script will replace any `(HOST= xxx)` with the new value like `(HOST = <newhostname>)`.

### $ORACLE_HOME/network/admin/listener.ora ###

````
<copy># listener.ora Network Configuration File: /u01/app/oracle/product/19c/dbhome_1/network/admin/listener.ora

LISTCDB1 =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS =
         (PROTOCOL = TCP)
         (HOST = myhostname )
         (PORT = 1523 )
      )
    )
  )

LISTCDB2 =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS =
         (PROTOCOL = TCP)
         (HOST = myhostname )
         (PORT = 1524 )
      )
    )
  )</copy>
````

### $ORACLE_HOME/network/admin/tnsnames.ora ###

````
<copy># TNSnames.ora for the workshops setup

## CDBs

CDB1 = (DESCRIPTION =
          (ADDRESS =
             (PROTOCOL = TCP )
             (HOST = myhostname )
             (PORT = 1523 ) )
          (CONNECT_DATA =
             (SERVER = DEDICATED) (SERVICE_NAME = CDB1))
       )

CDB2 = (DESCRIPTION =
          (ADDRESS =
             (PROTOCOL = TCP )
             (HOST = myhostname )
             (PORT = 1524 ) )
          (CONNECT_DATA =
             (SERVER = DEDICATED) (SERVICE_NAME = CDB2))
       )

## PDBs


PDB1 = (DESCRIPTION =
          (ADDRESS =
             (PROTOCOL = TCP )
             (HOST = myhostname )
             (PORT = 1523 ) )
          (CONNECT_DATA =
             (SERVER = DEDICATED) (SERVICE_NAME = PDB1))
       )

PDB2 = (DESCRIPTION =
          (ADDRESS =
             (PROTOCOL = TCP)
             (HOST = myhostname )
             (PORT = 1524 ) )
          (CONNECT_DATA =
             (SERVER = DEDICATED) (SERVICE_NAME = PDB2) )
       )

## Listeners

LISTCDB1 = (ADDRESS =
              (PROTOCOL = TCP)
              (HOST = myhostname )
              (PORT = 1523 ))

LISTCDB2 = (ADDRESS =
              (PROTOCOL = TCP)
              (HOST = myhostname )
              (PORT = 1524 ))</copy>
````

## Create the scripts that we need ##

In the previous Systemd file we will call 2 scripts:

````
/home/oracle/scripts/start_all.sh
/home/oracle/scripts/stop_all.sh
````

First, we will create the start_all script. This script needs to do the following:

- Start up the databases
- Check if this is the first boot. If so
	- Change the hostname in the tnsnames.ora and listener.ora
	- Change the password for the sys, system and pdbadmin users in something random
	- Put all required information in the /home/oracle directory
- Startup the listeners

Create the new directory in case it does not exist:

````
$ <copy>mkdir /home/oracle/scripts</copy>
````
 
Create the start_all.sh file:

````
$ <copy>vi /home/oracle/scripts/start_all.sh</copy>
````

Paste the following code into the file:

````
<copy>#!/bin/bash
#
# Startup script for Oracle Database
#

ORAENV_ASK=NO

# First, start database from the /etc/oratab

export ORACLE_HOME=/u01/app/oracle/product/19c/dbhome_1
${ORACLE_HOME}/bin/dbstart

# Check if passwordfile exists (= first boot), if not create it

PWDFILE="/home/oracle/instance_passwords.txt"

if [ ! -f "$PWDFILE" ]
then

  # Generate new password

    NEWPWD=` < /dev/urandom tr -dc "A-Za-z0-9" | head -c12`

  # update system, sys, pdb_admin for CDB1 and PDB1

    ORACLE_SID=CDB1; . /usr/local/bin/oraenv
    sqlplus / as sysdba @/home/oracle/scripts/change_password.sql $NEWPWD PDB1

  # update system, sys for CDB2

    ORACLE_SID=CDB2; . /usr/local/bin/oraenv
    sqlplus / as sysdba @/home/oracle/scripts/change_password.sql $NEWPWD PDB2

  # write to password file

    echo "Your generated password is $NEWPWD" > $PWDFILE

  # Change hostname in listener and tnsnames

    sed -i 's/(HOST =.*)/(HOST = '$HOSTNAME' )/g' $ORACLE_HOME/network/admin/tnsnames.ora
    sed -i 's/(HOST =.*)/(HOST = '$HOSTNAME' )/g' $ORACLE_HOME/network/admin/listener.ora

fi

# Startup listeners

  ORACLE_SID=CDB1; . /usr/local/bin/oraenv

  lsnrctl start LISTCDB1
  lsnrctl start LISTCDB2

# End of file</copy>
````

Create the .sql script that will change the passwords:

````
$ <copy>vi /home/oracle/scripts/change_password.sql</copy>
````

Paste the following code into the file:

````
<copy>
alter user sys identified by &1;
alter user system identified by &1;
alter session set container=&2;
alter user pdbadmin identified by &1;
exit</copy>
````

Create the stop_all.sh file:

````
$ <copy>vi /home/oracle/scripts/stop_all.sh</copy>
````

Paste the following code into the file:

````
<copy>#!/bin/bash
#
# Shutdown script for Oracle Database
#

ORAENV_ASK=NO
ORACLE_SID=CDB1; . /usr/local/bin/oraenv

# Shutdown all databases and listeners

${ORACLE_HOME}/bin/dbshut $ORACLE_HOME

# End of file</copy>
````

Make sure the files are executable by oracle:

````
$ <copy>chmod u+x *_all.sh</copy>
````

## Cleanup, update and shutdown for image ##

Before we can create an image out of this environment, we need to run some cleanup tasks as root user:

````
$ <copy>sudo -s</copy>
````

Update to the latest OS version:

````
# <copy>yum -y update</copy>
````

Execute the oci-image-cleanup.sh script:

````
# <copy>/root/oci-image-cleanup.sh</copy>

[root@mt-ws-v2 ~]# ./oci-image-cleanup.sh

Deleting network config files...
Deleting /etc/hosts and /etc/resolv.conf...
Deleting SSH host keys in /etc/ssh/ ....
Deleting SSH keys for /root ...
Deleting SSH keys for /home/opc ...
Locking root account and clearing root passwd...
...
````

Remove the instance_passwords.txt file in the /home/oracle directory

````
# <copy>rm -f /home/oracle/instance_passwords.txt</copy>
````

Shutdown the environment from the prompt:

````
# <copy>shutdown -h now</copy>
````

- Stop the instance in the OCI Console
- Create a new custom image based on this environment

## Checklist ##

Checklist based on https://docs.oracle.com/en/cloud/marketplace/partner-portal/ocmpd/how-to-publish-an-oci-image-listing.pdf:

### Mandatory image guidelines ###

- SSH Host Keys MUST be unique to each instance. Use the oci-image-cleanup utility provided by the oci-utils package on GitHub.
	- Script execution is in the lab
- Images MUST ingest an SSH public key provided by a customer as part of the instance launch process.
	- Source image was already setup to do this, not changed
	- Key will be inserted into the root and opc .ssh/authorized keys
- Ensure the image is cloud-init enabled.
	- Is cloud-init enabled
- Any authorized_keys files MUST only contain keys provided by the user when the instance is launched. Use the oci-imagecleanup utility provided by the oci-utils package on GitHub.
	- Checked for root, opc and oracle user
- The SSH service MUST be configured to prevent password-based logins.
	- Checked
- All entries in the /root/.ssh/authorized_keys file MUST contain no-port-forwarding, no-agent-forwarding, no-X11-forwarding.
	- Checked (root .ssh dir is empty before image creation)
- The root user MUST NOT have usable entries in the authorized_keys file. 
	- Checked
- Image MUST boot for all compatible shapes.
	- Checked - Default shape is from OCI so already tested.
- Image MUST NOT have any hard-coded MAC addresses, Empty the /etc/udev/rules.d/70-persistent-net.rules file.
	- Checked
- DHCP MUST be enabled, Ensure it is configured manually.
	- Checked, done by oci-clean and cloud-init 

### RECOMMENDED IMAGE GUIDELINES ###

- /etc/ssh/sshd_config
	- PasswordAuthentication no
	- ChallengeResponseAuthentication no
	- UsePAM no (niet changed this one because of a warning for RH)
- Mandatory Access Control (MAC) SHOULD be enabled. 
	- SELinux is on 'ENFORCING'
- An Operating System (OS) Firewall SHOULD be enabled and configured to block any ports not specifically required as indicated in the listing documentation.
	- Firewalld is running, only port 22 open (not 1523 and 1524)
- All sensitive data such as passwords and private keys SHOULD be removed. This type of data can often be found in log files, source code, or build artifacts.
	- Checked
- cloud-init packages SHOULD be available for use during instance launch.
	- Checked
- The standard SSH access account SHOULD use the username opc. Enable cloud-init for the image.
	- Checked
- The SSH service config /etc/ssh/sshd_config SHOULD NOT permit root logins. 
	- Checked
- The root user's login shell SHOULD be set to /sbin/nologin.
	- Checked
- Image software SHOULD be updated as part of the final packaging process.
	- Checked, added as a step before image creation
- Images SHOULD NOT have any operating system level users configured with a password.
	- Checked
- If a system level user is configured with a password, it MUST be uniquely generated the first time the instance launches. 
	- Checked, sys/system/pdbadmin are all generated at initial boot
- Application passwords SHOULD NOT be hard-coded. 
	- Checked, sys/system/pdbadmin are all generated at initial boot
- Images SHOULD run in paravirtualized mode. Images MAY run in native mode. Images SHOULD NOT run in emulated mode. 
	- Running in native mode
- Any network managers SHOULD be stopped.
	- No network managers running in the image
- Images SHOULD utilize the NTP service provided by OCI.
	- Chrony is installed as indicated by the manual
- Images SHOULD have iSCSI timeout values set for proper boot volume connectivity.
	- Default image so I asume it is ok. 

## Acknowledgements ##

- **Author** - Robert Pastijn, Database Product Management, PTS EMEA - May 2020 