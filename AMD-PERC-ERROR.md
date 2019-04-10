# Provoking PERC H330 block device errors on a Dell AMD EPIC host
This describes a method for provoking errors that leads to serious data corruption on block devices supported by the PERC H330 controller.

## Environment
* Dell EMC PowerEdge R7425
  - PERC H330
  - Dell THNSF8200CCSE 200Gb SATA 6Gbps MLC SFF SSD
* OS: Ubuntu 18.04 LTS server
* Kernel: 4.15.0-43-generic

## Device layout

    $ lsblk
    NAME                             MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
    sda                                8:0    0 186.3G  0 disk 
    ├─sda1                             8:1    0   512M  0 part /boot/efi
    └─sda2                             8:2    0 185.8G  0 part 
      ├─a1024--gc1--b--01--vg-root   253:0    0  23.3G  0 lvm  /
      ├─a1024--gc1--b--01--vg-swap_1 253:1    0   1.9G  0 lvm  [SWAP]
      ├─a1024--gc1--b--01--vg-var    253:2    0   9.3G  0 lvm  /var
      ├─a1024--gc1--b--01--vg-tmp    253:3    0   1.9G  0 lvm  /tmp
      └─a1024--gc1--b--01--vg-home   253:4    0 149.5G  0 lvm  /home
    sr0                               11:0    1  58.2M  0 rom  

# Observing the failures
We commonly observe the error as a failure to complete dynamic linking of binaries because their contents or libraries or i-node table entries have been corrupted. 

 * The corruption is visible in files that have not been altered through the filesystem
 * Because data and i-nodes are aggressively cached by the kernel the on-device corruption can go unnoticed for a long time. We have seen days to weeks on a busy host with 1TB RAM

 Once observed, the corruption is usually severe enough to make further analysis impossible.

# Test environment used to provoke data corruption
This is a proven method of quickly provoking the error and detecting the data corruption in our environment. 

## Detecting the corruption
First we make checksums for all files in /usr/lib and /usr/bin

    find /usr/lib /usr/bin -type f -exec md5sum {} \; | tee usr_lib_bin.md5sum

A simple bash script checks for corruption in /usr/lib by doing:

 * Validate checksums of all files in /usr/lib and /usr/bin
 * Wait 10 seconds
 * Force a flush of the kernels i-node and data caches
 * Repeat *(stops when corruption is detected)*

 The script can be given on the command line or executed from a file:

    while md5sum --quiet -c usr_lib_bin.md5sum; do 
        echo OK $(date)
        sleep 10
        sync
        sudo bash -c "echo 3 > /proc/sys/vm/drop_caches"
    done 

*The above script should be running in a separate terminal **before you do any of the following commands***

## Provoking the corruption
*Notice! Corruption can be detected at any point while performing the following commands* 

Create some test data

    mkdir src; for f in {0..75}; do cp -r /usr/share/doc src/$f; done; du -hs src

On a clean install 'src' now contains about 1GB small files.

Generate a checksum file for the files in src:

    (cd src; find . -type f -exec md5sum {} \; ) | tee src.md5sum

Put the script into the file "cra.sh":

    rm -rf error dst_*
    touch error
    while test -f error -a ! -s error; do
        echo Starting copying
        for f in {0..30}; do
            cp -r src dst_$f &
        done
        echo Waiting for copying to finish
        wait
        echo Starting checking of copies
        for f in {0..30}; do
            (cd dst_$f; md5sum --quiet -c ../src.md5sum 2>&1 ) | tee -a error &
        done
        echo Waiting for checking of copies to finish
        wait
        echo Starting deletion of copies
        for f in {0..30}; do
            rm -rf dst_$f &
        done
        echo Waiting for deletion of copies to finish
        wait
    done

Run the script:

    bash cra.sh

*Await the eminent catastrophe!*

# Command log
These are the commands used to prepare a clean Ubuntu server install for the test:

    sudo apt update
    sudo apt upgrade
    sudo apt install tmux emacs 
    sudo apt install linux-image-4.15.0-43-generic
