# Change Audit Vault image for training #


## Description ##

Steps that need to be taken to take an AV appliance image and update it so it can be run for the Oracle PTS Security Workshop.

## Get access to a new image ##

- Provision a new image based on the latest source
- Connect via VNC console to the image to change the IP address
- Reboot the device to make ssh access possible
- Connect to the image using SSH and become the root user

## Change sudo for easier root access ##

Once logged in, I sometimes had problems accessing the root user. To make this easier, I changed the `sudoers` file to include support.

When logged in as root, execute the following file using your favourite editor:

````
# <copy>sudoedit /etc/sudoers</copy>
````

Add the following line below the `root` entry:

````
<copy>support         ALL = (ALL) NOPASSWD: ALL</copy>
````

Save the file and exit.

## Add script to init.d for change ##

The MAC address is used in some parts of the DBFW image. If the MAC address is not the same as the actual MAC address during boot, the system will start asking for the correct values.

We can need to change the MAC address before the DBFW starts (but after the network has been started):

Navigate to the /etc/init.d directory so we can add a script:

````
# <copy>cd /etc/init.d</copy>
````

Create a new script called `dbfw-oraclepts` in this directory:

````
# <copy>vi /etc/init.d/dbfw-oraclepts</copy>
````

Copy and paste the following contents:

````
<copy>#!/bin/bash
#############################################################################
# Copyright © 2010, Oracle and/or its affiliates. All rights reserved.
#############################################################################
#
# dbfw-oraclepts
#
# chkconfig: 35 6 93
# description: Configure the networking for DBFW in WS environment
#############################################################################

case "$1" in
start)
        /root/changemac.sh
        ;;
esac

exit $?</copy>
````

After creating the new startup-script, we need to enable it for the next reboot:

````
# <copy>chkconfig --add dbfw-oraclepts
````
## Add script to /root to change mac address ##

The actual change of values is a script we have placed in the /root directory. Navigate to the /root directory and create a new script called `changemac.sh`:

````
# <copy>cd /root</copy>
````

````
# <copy>vi changemac.sh
````

Enter the following script into this new file:

````
<copy>#!/bin/bash

for x in {1..20}; do

  NEWMAC=`/bin/cat /sys/class/net/eth0/address`

  if test -z "$NEWMAC"
  then
    sleep 30
  else
    /bin/sed -i '/NIC_MAPPING/s/".*"/"eth0\/'$NEWMAC'"/' /usr/local/dbfw/etc/dbfw.conf
    break
  fi
done</copy>
````

Save the file and close the editor. Make sure the script can be executed by the root user:

````
# <copy>chmod u+x /root/changemac.sh</copy>
````

## Add script to /root to cleanup before shutdown ##

In order for the system to pick a new network adaptor as eth0, the old adaptor needs to be removed. A script has been written to do this called `cleanip.sh`:

````
# <copy>vi cleanip.sh</copy>
````

Copy the following script into the new file:

````
<copy>#!/bin/bash

sed -i '/SUBSYSTEM/d' /etc/udev/rules.d/70-persistent-net.rules
sed -i '/NIC_MAPPING/s/".*"/"eth0\/99:99:99:99:99:99"/' /usr/local/dbfw/etc/dbfw.conf
sed -i '/HWADDR/d' /etc/sysconfig/network-scripts/ifcfg-eth0
</copy>
````

Save the file and exit the editor. After this, make sure the file is executable:

````
# <copy>chmod u+x /root/cleanip.sh</copy>
````

## Cleanup environment, shutdown and make image ##

Before you shutdown the environment to create a new custom image, execute the following command:

````
# <copy>/root/cleanip.sh</copy>
````

After this, initiate shutdown of the image from the SSH connection:

````
# <copy>shutdown -h now</copy>
````

After disconnect, execute a STOP from the OCI console. After full shutdown, create a new image to use as a source for the workshop.

## Acknowledgements ##

- **Author** - Robert Pastijn, Database Product Management, PTS EMEA - April 2020

