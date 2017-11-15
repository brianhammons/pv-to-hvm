# Paravirtual to Hardware Virtual Machine image
Steps to convert Linux Amazon Machine Images from paravirtual (PA) to hardware virtual machine (HVM)

Introduction:

With the introduction of the new 'HVM only' instances types, there are many users that want to convert their existing PV instances to HVM instance. This article provides the required steps to create a new HVM AMI based on the original PV instance.

Requirements:

Original instance must have grub installed.
Temporary 'working' Amazon Linux instance
Time - for larger root volumes, this can take a long time.
Temporary IAM user with 'Power User' access.
All steps below to be performed as root user.
Summary:

Attach the source and destination volumes to the working instance.
Partition the destination instance and dd the data from the source volume to the destination volume.
Make required changes to the configuration files on the destination volume and install grub.
Create a snapshot of the destination volume and register an AMI.
Create a new HVM instance from the AMI.
Clean up temporary volumes and instance
The end :)
Detailed, step by step instructions:

These steps are for a RHEL based PV instances - Amazon, CentOS, Red Hat. (See notes below for Ubuntu)

Part 1: Preparation

1.  Prepare the source PV instance
     - Install grub
    yum install grub -y
     - Stop the instance and create snapshot of the root volume

2. Launch an Amazon Linux 'working' instance

3. Configure AWS CLI tools if not already configured
   aws configure
    AWS Access Key ID [None]: XXXXXXXXXXXXXXXXXXXX
    AWS Secret Access Key [None]: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    Default region name [None]: us-east-1
    Default output format [None]: json

4. Create a new 'source' volume (in the same AZ) from the snapshot created in step 1 and attach it to the 'working' instance as /dev/sdm.

5. Create a new 'destination' volume (in the same AZ) and attach it to the 'working' instance as /dev/sdo.

Part 2: Conversion

1. Partition the 'destination' volume.
   parted /dev/xvdo --script 'mklabel msdos mkpart primary 1M -1s print quit'
   partprobe /dev/xvdo
   udevadm settle

2. Check the 'source' volume and minimize the size of original filesystem to speed up the process. We do not want to copy free disk space in the next step.
   e2fsck -f /dev/xvdm
   resize2fs -M /dev/xvdm
     - Output from resize command:
     Resizing the filesystem on /dev/xvdm to 269020 (4k) blocks.
     The filesystem on /dev/xvdm is now 269020 blocks long.

3. Duplicate 'source' to 'destination' volume.
   dd if=/dev/xvdm of=/dev/xvdo1 bs=4K count=269020
    ***NB: Ensure you use the exact value for "bs=" and "count=" from the resize2fs command***

4. Resize the filesystem on the 'destination' volume after the transfer completed.
   resize2fs /dev/xvdo1

5. Prepare 'destination' volume for the chrooted grub installation.
   mount /dev/xvdo1 /mnt
   cp -a /dev/xvdo /dev/xvdo1 /mnt/dev/
   rm -f /mnt/boot/grub/*stage*
   cp /mnt/usr/*/grub/*/*stage* /mnt/boot/grub/
   rm -f /mnt/boot/grub/device.map

6. Install grub in the chroot environment. This step will do an offline grub installation on the destination device, which is required for the HVM instance:
---------------------
cat <<EOF | chroot /mnt grub --batch
device (hd0) /dev/xvdo
root (hd0,0)
setup (hd0)
EOF
---------------------
     - Remove the temporary device from the destination volume, which was required to install grub (as above)
    rm -f /mnt/dev/xvdo /mnt/dev/xvdo1

7. Now that we have grub installed, we need to update the grub configuration.
    - Edit /mnt/boot/grub/menu.lst:
    - Change root (hd0) to root (hd0,0)
    - Add (or replace console=*) console=ttyS0 to kernel line
    - Replace root=* with root=LABEL=/ in the kernel line
    - Add xen_pv_hvm=enable to kernel line (No longer required, but I still add it)
    - Example /mnt/boot/grub/menu.lst - *ensure the filenames match the files in /mnt/boot:
---------------------
default=0
timeout=0


title CentOS 6.5
root (hd0,0)
kernel /boot/vmlinuz-2.6.32-431.11.2.el6.x86_64 o root=LABEL=/ console=ttyS0 xen_pv_hvm=enable
initrd /boot/initramfs-2.6.32-431.11.2.el6.x86_64.img
---------------------

8. Update the root (/) entry in /mnt/etc/fstab, as per the sample fstab below:
---------------------
LABEL=/       /            ext4    defaults,noatime 1 1
none          /dev/pts     devpts  gid=5,mode=620   0 0
none          /dev/shm     tmpfs   defaults         0 0
none          /proc        proc    defaults         0 0
none          /sys         sysfs   defaults         0 0
---------------------

9. Create the label on /dev/xvdo1 and unmount device.
   e2label /dev/xvdo1 /
   sync
   umount /mnt

Part 3: Finalize

1. Create a snapshot of the 'destination' volume and register the AMI.
1.1 Create snapshot:
aws ec2 create-snapshot --volume-id vol-xxxxxxxx --description "CentOS 6.5 HVM snapshot"

1.2 Register new HVM AMI (replace snap-xxxxxxxx with the snapshot id created above):
aws ec2 register-image --name "CentOS 6.5 HVM" --description "CentOS 6.5 HVM" --architecture x86_64 --root-device-name "/dev/sda1" --virtualization-type hvm --block-device-mappings '[ { "VirtualName": "ebs", "DeviceName": "/dev/sda1", "Ebs": { "SnapshotId": "snap-xxxxxxxx", "VolumeSize": 10, "DeleteOnTermination": true, "VolumeType": "standard" } } ]'

2. Launch a test HVM type instance from registered AMI.
2.1 Launch instance using AWS console
2.2 Launch instance using aws-cli:
aws ec2 run-instances --instance-type m3.medium --security-group-ids sg-xxxxxxxx --key-name ssh-key-name --image-id "ami-xxxxxxxx"

The new HVM instance should now boot successfully and will be an exact copy of the old source PV instance (if you used the correct source volume).

Once you have confirmed that everything is working, the source instance can be stopped.
Clean up by removing all temporary volumes (source and destination).
