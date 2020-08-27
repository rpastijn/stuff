# Creating a custom image from a Marketplace source # 

The following I created after a request Kay Malcolm and Troy Anthony because they needed a customized version of an OCI Marketplace image as a custom image.

## Disclaimer ##
The following is intended to outline our general product direction. It is intended for information purposes only, and may not be incorporated into any contract. It is not a commitment to deliver any material, code, or functionality, and should not be relied upon in making purchasing decisions. The development, release, and timing of any features or functionality described for Oracle’s products remains at the sole discretion of Oracle.

## Requirements ##

- Access to the original image (based on the Marketplace image)
- Original image has the OCI Cleanup scripts run (mandatory for MP images)
- The Original image is able to startup and run without any manual actions

## Step 1 - Note the source details ##

We need the following information:

- OS version
- Size of bootdisk
- Region where the source system resides
- AD where the source system resides

## Step 2 - Create a new target instance ##

Create a new, default based instance

- Take a small VM size (CPU does not matter)
- Choose an OS which is closest to the source image OS (and version)
- Make sure the bootdisk size is the same size as the source bootdisk size
- Make sure that the instance is created in the same region and AD as the source
- No need for an SSH key, we are not going to login

Start the new instance and continue

## Step 3 - Create a new dummy instance ##

- Take a small VM size (CPU does not matter)
- Accept the smallest (default) bootdisk size
- Make sure that the instance is created in the same region and AD as the source
- Put in your personal SSH key as you need to login

Start the new instance and continue

## Step 4 - Shutdown and detach ##

- Shutdown the new target system (from step 2) after creation has finished
- Shutdown the source system (if it is running)

We need to work with the bootdisks of both systems so we are going to connect the bootdisks of the source and target systems to the dummy instance we created.

- Navigate into the source instance details
	- Go to the left menu and select Boot Volume
	- Select 'Detach Boot Volume' from the menu to the right of the boot volume
- Navigate into the target instance details
	- Go to the left menu and select Boot Volume
	- Select 'Detach Boot Volume' from the menu to the right of the boot volume

> Please note you can only detach the boot volume after full stop of an instance.

Now wait until both volumes have been detached from their systems. You can check this in the bootvolumes overview page if needed (or in the individual instances details)

## Step 5 - Attach bootvolumes to dummy instance ##

To manipulate the source and the target instances, we need a 'in between' instance since we can only read and write on block level if the instances are not in use.

- Navigate to the dummy instance you created in step 3
- Navigate to the instance details -> Attached Block Volumes menu

The list should be empty as no additional volumes are attached by default.

- Click the button "Attach Block Volume"
	- Select the "Paravirtualized" Volume Attachment option
	- Select the SOURCE block volume for attachment
	- Keep access on READ/WRITE
	- Click 'Attach'
- Click the button "Attach Block Volume"
	- Select the "Paravirtualized" Volume Attachment option
	- Select the TARGET block volume for attachment
	- Keep access on READ/WRITE
	- Click 'Attach'
- Wait until both volumes have attached to your dummy instance

## Step 6 - Login and copy ##

After the volumes have been attached, you can start copying. We will use our dummy instance to do the copying:

Log in to the dummy instance using the key you provided as opc user:

````
login as: opc
Authenticating with public key "some-key-i-used"
[opc@dummy-system ~]$

````

Become the root user:

````
[opc@dummy-system ~]$ <copy>sudo -s</copy>
[root@dummy-system opc]#
````

We check if the source and the target volumes have been attached:

````
[root@dummy-system opc]# <copy>fdisk -l</copy>
WARNING: fdisk GPT support is currently new, and therefore in an experimental ph                                                                           ase. Use at your own discretion.

Disk /dev/sda: 50.0 GB, 50010783744 bytes, 97677312 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 1048576 bytes
Disk label type: gpt
Disk identifier: 6FA7C704-7B17-4999-BBFD-523636C6F01F


#         Start          End    Size  Type            Name
 1         2048       411647    200M  EFI System      EFI System Partition
 2       411648     17188863      8G  Linux swap
 3     17188864     97675263   38.4G  Microsoft basic
WARNING: fdisk GPT support is currently new, and therefore in an experimental ph                                                                           ase. Use at your own discretion.

Disk /dev/sdb: 50.0 GB, 50010783744 bytes, 97677312 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 1048576 bytes
Disk label type: gpt
Disk identifier: 6FA7C704-7B17-4999-BBFD-523636C6F01F


#         Start          End    Size  Type            Name
 1         2048       411647    200M  EFI System      EFI System Partition
 2       411648     17188863      8G  Linux swap
 3     17188864     97675263   38.4G  Microsoft basic
WARNING: fdisk GPT support is currently new, and therefore in an experimental ph                                                                           ase. Use at your own discretion.

Disk /dev/sdc: 50.0 GB, 50010783744 bytes, 97677312 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 1048576 bytes
Disk label type: gpt
Disk identifier: 6FA7C704-7B17-4999-BBFD-523636C6F01F


#         Start          End    Size  Type            Name
 1         2048       411647    200M  EFI System      EFI System Partition
 2       411648     17188863      8G  Linux swap
 3     17188864     97675263   38.4G  Microsoft basic
````

The first disk (bootvolume) that you have attached will be /dev/sdb. The second disk (bootvolume) will be /dev/sdc.

We do not need to mount the volumes to copy them; we can simply do a one-on-one copy of every bit that is on the source disk and write it to the same place on the target disk:

````
[root@dummy-system opc]# <copy>dd if=/dev/sdb of=/dev/sdc bs=1M status=progress</copy>
140509184 bytes (141 MB) copied, 4.104482 s, 34.2 MB/s
````

Now we simply have to wait for the dd to finish. You know what the size is of the source disk so you can calculate how long it will take.

- Getting a larger VM shape for the dummy environment will speed things up
- There are options in the console to increase the throughput on the disks

But basically, you have to wait until the copying is finished.

## Step 7 - Sync and detach ##

When the copying has finished, make sure all of the data has been written to the disks before we disconnect them:

````
[root@dummy-system opc]# <copy>sync; sync;</copy>
````

You can now logout of the dummy instance. Do not shut it down or delete it as this will delay the detachment of the boot volume that we need.

Instead..
- Go to the DUMMY instance
- Detach the TARGET bootvolume from the attached block volumes
- Wait until the volume has been detached

## Step 8 - Attach and create custom image ##

The bootvolume can now be attached again to its original instance:

- In the OCI console, go to the Target instance
- Navigate to the Boot Volume section of this instance
- Select to 'Attach' the Boot Volume again
- Wait for this to finish

> **DO NOT** start the Target instance at this moment because then you need to run the OCI cleanup scripts again..

- Go to the instance properties of the Target instance
- From the drop-down menu, select the creation of the Custom image
- Give it a name start the process
- Wait for it to finish
- Create a new instance based on the new Custom image
- Check if everything has worked
	- If not, go back to step 1 :-)

## Optional Step 9 - Export the custom image to object storage ##

If needed (and after the custom image creation) you can now export the custom image to Object Storage so that you can import it in another tenancy or region.


## Acknowledgements ##

- **Author** - Robert Pastijn, Database Product Management, PTS EMEA - August 2020

