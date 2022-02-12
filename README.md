# Islander. The analogue of Docker.

## Features

- Namespaces: `Network` `Mount` `UTS` `PID` `User`
- Cgroups: `blkio` `memory` `cpu` `devices` `net_cls`
- Container Management: `ps` `delete` `detach` `islenodes`
- Data Management: `volumes` `btrfs` `tmpfs` `S3 buckets`
- Daemon: `Daemon Server` `Daemon Client`
- Logger: `Logger Server` `Logs`
- Clouds: `terraform scripts`

## Description

**Islander** is a container engine, analog of Docker. Our container is called **isle** (pronunciation -- [il]), which is based on Linux namespaces and cgroup v1.

- Full demo of remote volumes with AWS, Azure and GCP you can find here: [Remote Volumes Demo](https://youtu.be/IjaCSn4vsO0)
- Full demo of namespaces can be found here: [Islander Namespaces Demo](https://www.youtube.com/watch?v=JzkbJ1fnCM0&ab_channel=%D0%AF%D1%80%D0%BE%D1%81%D0%BB%D0%B0%D0%B2%D0%9C%D0%BE%D1%80%D0%BE%D0%B7%D0%B5%D0%B2%D0%B8%D1%87)
- Project presentations can be found in `./docs`


### Description of limit options

For more details we recommend to look in section 3 of RedHat documentation about cgroups "Subsystems and Tunable Parameters" [here](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/resource_management_guide/ch-subsystems_and_tunable_parameters).

| Option [default]     | Description |
| --------------- | ----------- |
|   --memory-in-bytes [500M]   | Memory limit (format: `<number>`[`<unit>`]). Number is a positive integer. Unit can be one of b, k, m, or g. Minimum is 4M. |
|   --cpu-shares [100]   | CPU shares (relative weight). For example, tasks in two cgroups that have cpu.shares set to 100 will receive equal CPU time, but tasks in a cgroup that has cpu.shares set to 200 receive twice the CPU time of tasks in a cgroup where cpu.shares is set to 100. The value specified in the cpu.shares file must be 2 or higher. |
|   --cpu-period [100_000]   | Limit the CPU CFS (Completely Fair Scheduler) period.  If tasks in a cgroup should be able to access a single CPU for 0.2 seconds out of every 1 second, set cpu.cfs_quota_us to 200000 and cpu.cfs_period_us to 1000000. |
|   --cpu-quota [1000_000]   | Limit the CPU CFS (Completely Fair Scheduler) quota. |
|   --device-read-bps [500M]   | Limit read rate from the host filesystem (format: `<number>`[`<unit>`]). Number is a positive integer. Unit can be one of kb, mb, or gb. |
|   --device-write-bps [100M]   | Limit write rate the host filesystem (format: `<number>`[`<unit>]`). Number is a positive integer. Unit can be one of kb, mb, or gb. |


## Compile Project

1) Set up environment
```shell
# 1. Create all required folders and install rootfs:
make configure

# 2. Compile all the subprojects at once:
make build
```

2) Create files with credentials in case you want to use remote volumes
* how to create ~/islander/remote-volumes/gcp_secrets.json described [here](https://cloud.google.com/iam/docs/creating-managing-service-account-keys)

* to create ~/islander/remote-volumes/s3_secrets.txt use the next pattern:
```text
<access_key_id>:<secret_access_key>
```

* to create ~/islander/remote-volumes/az_secrets.txt use the next pattern:
```text
AZURE_STORAGE_ACCOUNT=myaccountname
AZURE_STORAGE_ACCESS_KEY=myaccountkey
```

For more details look at [this README, Usage/Mounting section](https://github.com/Azure/azure-storage-fuse)


## Container management

### Attach to detached container

Manual approach for debugging:
```shell
# run logger_server to save containers output
sudo ./logger_server

# start container in detached mode;
# binary file path is based on ubuntu-root-fs as root dir
sudo ./islander_engine ./project_bin/log_time_sample -d

# attach to process and redirect stdout and stderr in special tty (change tty for your needs)
sudo gdb -p 70235 -x process_attach
```

Actually, idea how to create attach is correct and defined above. However, we have no enough time to finish it. The base structure of the solution defined in `./attach` directory. 

**Idea of the solution** is the next:
* implement detached mode (already done).

* create logger server to save stdout and stderr of the container to external file outside of the container. Note that we need to create a logger server to save output into an external file, since we use mount namespace for our container and it can not see anything outside of ubuntu-rootfs, which we mounted as a defaul fs. If you want to get any file on your host fs, you can not do it from the container. Hence, we created a logger instance, which get stdout and stderr of the container via pipe and save in target files. This step is already implemented.

* use gdb to attach to container via its PID and redirect its stdout and stderr into your tty with dup2 syscall or just redirect back into 1 and 2 file descriptors, since in detached container stdout and stderr write to descriptors of logger pipe. For more detailes explore [the next page](https://www.baeldung.com/linux/redirect-output-of-running-process)
    * for this step you also need to know numbers of file descriptors, in which stdout and stderr of containers were redirected, to redirect these numbers to 1 and 2, or just into fd of target tty. For this you can use communication between logger server and target container.


* with above logic you can attach and see stdout and stderr of a previously detached container. To return again to detached container you need to use the same idea with gdb, but partly reverted. However, to implement attach also for stdin, including dynamic stdin, as in bash, you need to develop the same logic, which we created for client-server communication for dynamic commands like bash or gcc. Deep dive into our code of `./islander-client` and `./islander-server` and also watch our final presentation to understand the main architecture of workflow with dynamic commands.


## Manage data

### Usage of remote volumes

#### S3 buckets
```shell
# create s3 bucket
./remote-vlm-manager create aws os-project-bucket2022

# mount s3 bucket manually
s3fs os-project-bucket2022 ../ubuntu-rootfs/s3_bucket/ -o passwd_file=/home/denys_herasymuk/islander/remote-volumes/cloud_secrets/s3_secrets.txt

# mount s3 bucket with islander_engine
sudo ./islander_engine /bin/bash --mount-aws src os-project-bucket2022 dst ../ubuntu-rootfs/s3_bucket/

# delete s3 bucket
./remote-vlm-manager delete aws os-project-bucket2022
```


#### Azure storage containers
```shell
# create storage container
./remote-vlm-manager create az os-project-bucket2022

# to mount storage container manually use next commands;
# before this step, container should be already created in your storage account
export AZURE_STORAGE_ACCOUNT=mystorage
export AZURE_STORAGE_ACCESS_KEY=mycontainer
blobfuse ./test --container-name=os-project-bucket2022 --tmp-path=/home/denys_herasymuk/islander/remote-volumes/az_storage/blobfusetmp -o allow_other

# set feature of mounting Azure storage container
sudo ./islander_engine /bin/bash --mount-az src os-project-bucket2022 dst ../ubuntu-rootfs/s3_bucket/

# delete storage container
./remote-vlm-manager delete az os-project-bucket2022
```


#### GCP buckets
```shell
# create storage container
./remote-vlm-manager create gcp os-project-bucket2022

# to mount bucket manually use next commands;
# before this step, bucket should be already created in your GCP account
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/secrets.json
gcsfuse my_bucket_name /path/to/mountpoint

# set feature of mounting GCP storage bucket
sudo ./islander_engine /bin/bash --mount-gcp src os-project-bucket2022 dst ../ubuntu-rootfs/s3_bucket/

# delete storage container
./remote-vlm-manager delete az os-project-bucket2022
```


### Test data management

```shell
# example of mount feature usage
sudo ./islander_engine /bin/bash --mount src ../tests/test_mount/ dst ../ubuntu-rootfs/test_mnt/ --mount src /dev/ dst ../ubuntu-rootfs/host_dev/

# example of volume feature usage
sudo ./islander_engine /bin/bash --volume src test_mnt1 dst ../ubuntu-rootfs/test_mnt/ --volume src host_dev dst ../ubuntu-rootfs/host_dev/

# check results in ~/islander/volumes
sudo sh -c "cd test_mnt1/; ls"
sudo sh -c "cat ./test_mnt1/f2.txt"

# example of tmpfs feature usage
sudo ./islander_engine /bin/bash --device-write-bps 10485760000 --memory-in-bytes 1000M --tmpfs dst ../ubuntu-rootfs/test_tmpfs size 2G nr_inodes 1k --mount src /dev/ dst ../ubuntu-rootfs/host_dev/
```

```shell
## Data Management
# example of volume feature usage
sudo ./islander_engine /bin/bash --volume src test_mnt1 dst ../ubuntu-rootfs/test_mnt/ --volume src host_dev dst ../ubuntu-rootfs/host_dev/

./ps

# check results in ~/islander/volumes
sudo sh -c "cd test_mnt1/; ls"
sudo sh -c "cat ./test_mnt1/f2.txt"


# example of tmpfs feature usage
sudo ./islander_engine /bin/bash --device-write-bps 10485760000 --memory-in-bytes 1000M --tmpfs dst ../ubuntu-rootfs/test_tmpfs size 2G nr_inodes 1k --mount src /dev/ dst ../ubuntu-rootfs/host_dev/


# filter in htop

# create 1G temp file
dd if=/host_dev/zero of=./writetest bs=256k count=4000 conv=fdatasync

htop

df -h


# start wireshark

## Client-server interaction
sudo ./islander-server --port 5020

./client --bin id --memory-in-bytes 1000m --port 5020
./client --bin ls --memory-in-bytes 1000m --port 5020
./client --bin hostname --memory-in-bytes 1000m --port 5020
./client --bin pwd --memory-in-bytes 1000m --port 5020

# show wireshark

```


### tmpfs usage
```shell
sudo mount /dev/nvme0n1p5 ~/islander/volumes/

# filter in htop

sudo mount -t tmpfs -o size=2G,nr_inodes=1k,mode=777 tmpfs ./reports

df -h

# create 1G temp file
dd if=/host_dev/zero of=./writetest bs=256k count=4000 conv=fdatasync

htop

df -h

sudo umount -R ./reports
```

### Btrfs usage

```shell
sudo mkfs.btrfs -L data /dev/nvme0n1p5 -f

sudo mount /dev/nvme0n1p5 ~/islander/volumes/

sudo btrfs subvolume create /var/lib/islander/volumes/test_volume

sudo btrfs subvolume list /var/lib/islander/volumes

sudo btrfs subvolume show /var/lib/islander/volumes/test_volume

sudo mount /dev/nvme0n1p5 -o subvol=test_volume ./test_volumes/

sudo umount ./test_volumes/
```

### Namespaces Usage
```shell
# Check User Namespace
id

# Check UTS Namespace
hostname <some-host-name>
hostname

# Check Mount Namespace
ls
cd ..
ls

# Check PID Namespace
echo $$

# Check Network Namespace
ip link list
ping 10.1.1.1
```

### PS Usage
```shell
shell-main # sudo ./islander_engine /bin/bash

shell-seconady # ./ps
```


## Limit management

### Type of commands

```shell
sudo ./namespaces /bin/bash

# in this case default limits will be used
sudo ./namespaces /bin/bash --memory-in-bytes 1G --cpu-quota 100000 --device-write-bps 10485760
```


### Test examples

Different rootfs tar.gz are located in isle/files dir, can be useful for testing.

**Check --memory-in-bytes and --cpu-quota flags**
```shell
isle/build$ sudo ./namespaces /bin/bash --memory-in-bytes 1G --cpu-quota 100000 --device-write-bps 10485760

PID: 30106


=========== /bin/bash ============
/ # stress --cpu 6
stress: info: [2] dispatching hogs: 6 cpu, 0 io, 0 vm, 0 hdd
^C
/ # stress --vm 8
stress: info: [9] dispatching hogs: 0 cpu, 0 io, 8 vm, 0 hdd
^C
/ # stress --vm 8
stress: info: [18] dispatching hogs: 0 cpu, 0 io, 8 vm, 0 hdd
^C
/ # stress --vm 2 --vm-bytes 1G
stress: info: [32] dispatching hogs: 0 cpu, 0 io, 2 vm, 0 hdd
^C
/ # exit
```


```shell
isle/build$ sudo ./namespaces /bin/bash --memory-in-bytes 4G --cpu-quota 1000000 --device-write-bps 10485760

PID: 30106


=========== /bin/bash ============
/ # stress --cpu 6
stress: info: [2] dispatching hogs: 6 cpu, 0 io, 0 vm, 0 hdd
^C
/ # stress --vm 8
stress: info: [9] dispatching hogs: 0 cpu, 0 io, 8 vm, 0 hdd
^C
/ # stress --vm 8
stress: info: [18] dispatching hogs: 0 cpu, 0 io, 8 vm, 0 hdd
^C
/ # stress --vm 4 --vm-bytes 1G
stress: info: [27] dispatching hogs: 0 cpu, 0 io, 4 vm, 0 hdd
^C
/ # stress --vm 2 --vm-bytes 1G
stress: info: [32] dispatching hogs: 0 cpu, 0 io, 2 vm, 0 hdd
^C
/ # exit
```


**Check --device-read-bps flags**


* Run this command to test --device-read-bps
```shell
sudo ./namespaces /bin/bash --device-read-bps 20975760
```

* Mount **/dev/** directory to our namespace with:
```shell
mkdir ./ubuntu-rootfs/host_dev

sudo nsenter -t <PID> mount --bind /dev/ ./ubuntu-rootfs/host_dev/

# before exit from our isle run this command on host to make safe exit
sudo nsenter -t <PID> umount -R ./ubuntu-rootfs/host_dev/
```

* Run dd command in our isle

```shell
dd iflag=direct if=/tmp/readtest of=/host_dev/null bs=64K count=1600
```



**Check --device-write-bps flags**

* Run this command to test --device-write-bps
```shell
sudo ./namespaces /bin/bash --device-write-bps 20975760
```

* Mount **/dev/** directory to our namespace with:
```shell
mkdir ./ubuntu-rootfs/host_dev

sudo nsenter -t <PID> mount --bind /dev/ ./ubuntu-rootfs/host_dev/

sudo nsenter -t <PID> umount -R ./ubuntu-rootfs/host_dev/
```

* Run dd command in our isle

```shell
dd if=/host_dev/zero of=/tmp/writetest bs=64k count=1600 conv=fdatasync && rm /tmp/writetest
```


### Useful commands:

* `docker run -ti --rm containerstack/alpine-stress sh`
* `docker export ec72296fbdde | gzip > ubuntu-rootfs.tar.gz` or
 `docker export 0d95c058d6ea > alpine-stress.tar.gz`
* to send SIGKILL to process use:
```shell
# get PID
ps ax | grep isla
sudo kill -s SIGKILL <PID>
```
* to check limits ou can use `glances` Linux tool
* load computer memory
```shell
stress --vm 8 --vm-bytes 1G
```

* Run dd command in host
```shell
dd if=/dev/zero of=./ubuntu-rootfs/tmp/writetest bs=64k count=3200 && rm ./ubuntu-rootfs/tmp/writetest
```

* block io measure
```shell
(base) root@denys-herasymuk-Strix-15-GL503GE:/# time bash -c "dd if=/home/writetest of=/home/writetest2 bs=64k count=3200 && rm /home/writetest2"
3200+0 records in
3200+0 records out
real	0m 0.18s
user	0m 0.00s
sys	0m 0.17s
```

## References

- Network Namespace. [josephmuia.ca](https://josephmuia.ca/2018-05-16-net-namespaces-veth-nat/)
- Linux Namespaces. [habr.com](https://habr.com/ru/company/selectel/blog/303190/)



