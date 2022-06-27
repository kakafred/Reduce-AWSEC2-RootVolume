# Reduce-AWSEC2-RootVolume

## Summary
**Amazon Elastic Block Store (EBS)** is an easy-to-use, scalable, high-performance block-storage service designed for **Amazon Elastic Compute Cloud (Amazon EC2)**. 
In actual user scenarios, since many customers have not used the *Linux operating system* or are inexperienced in disk management, they will directly create a ***root volume*** with a large capacity will cause the system and data to be read and written on the same disk, which will also increase the difficulty of reducing EBS. for In order to reduce the cost of running on AWS, users may want to reduce the capacity of the system volume.
Steps for EBS capacity. The following example shows how to use the ***rsync*** to replace a large-capacity EBS system volume with a small-capacity EBS volume.

## Solution Arch && Introduction
⚠️ The operating system of the experimental environment is - ***amazon linux 2 (HVM) Kernel 5.10*** ami-033a6a056910d1137 (AWS AMI ID), the operating system of all instances in the lab environment must be the same, AWS will automatically rewrite the device name of all attached volumes (for centos,
Linux operating systems such as ubuntu **have not been tested**, users of these operating systems should use this method with caution.

![BootVolume](https://user-images.githubusercontent.com/82210954/175901163-1ed0274f-25eb-4a62-8933-a630cc45ab2d.jpg)

When EC2 starts, it will initialize the configured EBS root volume according to the operating system AMI selected by the user and assign a default device name ***/dev/xvda*** is mounted to the EC2 instance as a root volume. In this experimental scenario, 3 EC2s will be created, Instance A as an instance that needs to shrink the EBS system volume (in a real case, this instance already exists and does not need to be created), Instance B is assumed as instances after EBS reduction (⚠ Instance B root volume capacity should not be less than the total occupied space of Instance A root volumes), Instance
C as an instance of shrinking EBS root volumes and performing related shrinking work.

![OperateVolume](https://user-images.githubusercontent.com/82210954/175902289-5eaac63d-bb19-4282-ba82-64809e508442.jpg)

1. create **Instance A** and attach a 20Gb root volume, then create an Instance B with the same operating system as **Instance A**, and attach an 8Gb root volume.
2. Wait for **Instance B** to be created, stop the instance and detach the root volume.
3. Stop the instance **Instance A** that needs to be scaled down.
4. Select the root volume of **Instance A** and create a snapshot of the volume. After the snapshot is completed, select the snapshot to create a "General Purpose SSD" type volume as the root volume of **Instance A**.
5. Create **Instance C**, which will be used as an instance for incremental data synchronization (rsync) and execution of related reduction work (bash commands). Of course, **Instance A** can also be used for these tasks, but I prefer to use a separate instance to complete these tasks.
Attach the snapshot volume of **Instance A** and the detached root volume of **Instance B** to **Instance C**. Use */dev/xvdf* as the device name of the **Instance B** volume and */dev/xvdg* as the device name of the **Instance A** volume.

## AWS Web Console Sreenshots && Steps tutorial
Create Instance A and attach a 20Gb root volume, and then create an instance with the same operating root as Instance A. Instance B, and attach an 8Gb root volume.
![webconsole_screenshot1](https://user-images.githubusercontent.com/82210954/175905797-d76809ad-e0d9-4f18-8342-891601618b28.png)
![webconsole_screenshot2](https://user-images.githubusercontent.com/82210954/175905805-fe8c37d9-8097-41fe-8729-142f8c884e77.png)

Wait for Instance B to be created, stop the instance and detach the root volume.
![webconsole_screenshot3](https://user-images.githubusercontent.com/82210954/175905898-dbbd9d96-a3d3-4b1f-955e-f9d987a30844.png)

Stop the instance Instance A that needs to be reduced.
![webconsole_screenshot4](https://user-images.githubusercontent.com/82210954/175905990-f9ea2c74-4400-4599-a498-ccbdbec11b73.png)

Select the root volume of Instance A and create a volume snapshot (snapshot), and select the snapshot to create a "General" after the snapshot is completed. A volume of type "Purpose SSD" is used as the root volume of Instance A.
![webconsole_screenshot5](https://user-images.githubusercontent.com/82210954/175906145-150075c5-a7bb-4722-9fca-bc0f76ec6685.png)
![webconsole_screenshot6](https://user-images.githubusercontent.com/82210954/175906171-0d9e5fce-92f2-46f4-b045-b150c98a0592.png)
![webconsole_screenshot7](https://user-images.githubusercontent.com/82210954/175906447-247e0fcf-faca-4a92-9907-b0b9060b08b5.png)

Create Instance C, which will be used as the source for incremental data synchronization (rsync) and related reduction work (bash commands).
instance. Of course, Instance A can also be used for these tasks, but I prefer to use an independent instance to complete these tasks.
Attach the snapshot volume of Instance A and the detached root volume of Instance B to Instance C, and use /dev/xvdf as the device name of the Instance B volume, and /dev/xvdg is used as the device name of the Instance A volume.
![webconsole_screenshot8](https://user-images.githubusercontent.com/82210954/175906411-f149f643-6e9b-4db5-9304-c0b55a9027d3.png)

## Using SSH client to exec bash commands
1. Create a directory for mounting the volume.
```
sudo mkdir /source /target
```
2. Check FileSystem Type.
```
sudo blkid
```

3. Format Instance B volume with xfs as filesystem.
```
sudo mkfs.xfs /dev/nvme1n1p1
OR
sudo mkfs.ext4 /dev/nvme1n1p1
```

4. Mount the volume to the target directory.
```
sudo mount -t xfs /dev/nvme1n1p1 /target
OR
sudo mount -t ext4 /dev/nvme1n1p1 /target
```

5. The file system needs to use e2label to let linux recognize and start it, execute 'e2label on Instance C
/dev/nvme1n1p1' to view the file system label, which is '/' in this experimental environment. **this step is very important**
```
xfs:
sudo umount /target
sudo xfs_admin -L / /dev/nvme1n1p1
sudo mount -t xfs /dev/nvme1n1p1 /targ
```

6. Mount the volume to the source directory.
```
sudo mount -t xfs /dev/nvme2n1p1 /source
OR
sudo mount -t ext4 /dev/nvme2n1p1 /source
```

7. Check volume mount status and start synchronizing volumes.
```
sudo lsblk
sudo rsync -vaxSHAX /source/ /target (⚠ Be sure to add / after source here)
```

8. After completing the synchronization of the volume, you need to correct the **UUID of the volume** and then detach the volume, otherwise AWS will start the EC2 instance because the UUID of the volume is *changed*. Modified, grub cannot find the volume UUID and cannot mount it, causing the system to fail to boot normally. **this step is very important**

8.1 View the UUID of the instance volume of Instance A and copy it.
```
sudo blkid
```
8.2 Assign the UUID of the instance volume of Instance A to the instance volume of Instance B.
```
sudo xfs_admin -U ${UUID} /dev/nvme1n1p1
```
8.3 Also check whether the uuid in /etc/fstab and /boot/grub2/grub.cnf are consistent
View the changed UUID of each volume.
```
sudo blkid
```

9. Unmount the instance volume of Instance B.
```
cd
sudo umount /target
```
If the partition is busy, check whether the current bash is in the target directory. If so, execute cd to exit the target directory.

10. Detach Instance B instance volume from Instance C.
11. Attach the Instance B instance volume to Instance A and start the Instance A instance.
12. Use ssh to connect to Instance A, start the application and check system integrity.

## Workflow Arch
![Workflow Arch](https://user-images.githubusercontent.com/82210954/175910951-4f385679-a9c3-4eb3-b6e0-c559d5035340.png)

### If there are no problems starting the instance and the application runs normally, the task is complete!
