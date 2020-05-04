# Changes to Vijays image #

## x ##

## Set PDBs to autostart ##



## Add systemd service for auto start ##

(section based on https://oracle-base.com/articles/linux/linux-services-systemd#creating-linux-services)

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
	- Change the password for the sys and system users in something random
	- Put all required information in the /home/opc directory

Change to the oracle user to create the scripts (as they will be executed as oracle user):

````
# <copy>su - oracle</copy>
````

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
<copy>
#!/bin/bash
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

  # update system, sys for CDB1

  ORACLE_SID=CDB1
  . /usr/local/bin/oraenv

  sqlplus / as sysdba @/home/oracle/scripts/change_password.sql $NEWPWD

  # update system, sys for CDB2

  ORACLE_SID=CDB2
  . /usr/local/bin/oraenv

  sqlplus / as sysdba @/home/oracle/scripts/change_password.sql $NEWPWD

  # write to password file

  echo "Your generated password is $NEWPWD" > $PWDFILE

  # Chance hostname in listener and tnsnames

  sed -i 's/(HOST =.*)/(HOST = '$HOSTNAME' )/g' $ORACLE_HOME/network/admin/tnsnames.ora
  sed -i 's/(HOST =.*)/(HOST = '$HOSTNAME' )/g' $ORACLE_HOME/network/admin/listener.ora

fi

# Startup listeners

  ORACLE_SID=CDB1
  . /usr/local/bin/oraenv

  lsnrctl start LISTCDB1
  lsnrctl start LISTCDB2

# End if file
````

Make sure the files are executable by oracle:

````
$ <copy>chmod u+x *_all.sh</copy>
````
