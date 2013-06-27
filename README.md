Flock / Rapid Infrastructure Prototype Engine
=================================================

*The fact is, Adelmo's death has caused much spiritual unease among my flock.*

![Flock](http://24.media.tumblr.com/tumblr_lzinfntu2G1qj8pa7o1_500.gif)

## Install for OS X
Go home:

    cd

For the Flock boot you need `ngnix` and `dnsmasq`:

    brew install nginx dnsmasq python libyaml

Edit your `.profile` or `.bash_profile` and set PATH:

    PATH=/usr/local/bin:$PATH

Install python packages:

    pip pyyaml jinja2 paramiko

For the Flock provision you need Ansible:

    git clone git://github.com/ansible/ansible.git

For the Flock you need Flock:

    git clone git://github.com/hornos/flock.git

Install VirtualBox extension pack.

### Setup
Edit your `.profile` or `.bash_profile`:

    source $HOME/flock/flockrc

Or run

    source $HOME/flock/flockrc

Mind that `flock` always works relative to the current directory:

    pushd flock

Generate SSH keys:

    flock init

Keys and certificates are in the `keys` directory.

## Network Install
Download [syslinux 4.X](https://www.kernel.org/pub/linux/utils/boot/syslinux/) and the following files to `space/boot`:

    core/pxelinux.0
    com32/mboot/mboot.c32

Get install images eg. for Debian (mind the trailing slash!):

    pushd space/boot
    rsync -avP ftp.us.debian.org::debian/dists/wheezy/main/installer-amd64/current/images/netboot/ ./wheezy
    popd

Debian-based systems should be installed with NAT.

Or get the kickass Debian killer CentOS (mind the trailing slash!):

    pushd space/boot
    rsync -avP rsync.hrz.tu-chemnitz.de::ftp/pub/linux/centos/6.4/os/x86_64/isolinux/ ./centos64
    popd

Space Jockey (`jockey`) is a simple Cobbler replacement. You need a simple inventory file like this (`space/hosts`):

    boot_server=10.1.1.254
    dhcp_range="10.1.1.1,10.1.1.128,255.255.255.0,6h"
    interface=eth1

The boot server listens on `boot_server` IP and Debian-based systems use the `interface` interface to reach the internet (NAT or bridged or 2nd physical network card). DNSmasq gives IPs from the `dhcp_range`.

### Core Servers
The following network topology is used:

    Network   VBox Net IPv4 Addr  Mask DHCP
    system    vboxnet0 10.1.1.254 16   off
    external  NAT/Bridged

If you use VirtualBox for modelling you can create the servers by:

    for i in 1 2 3 ; do flock-vbox create core-0$i; done

Set the boot device to the 1st network interface:

    for i in 1 2 3 ; do flock-vbox boot core-0$i net; done

Kickstart the cores with CentOS:

    for i in 1 2 3 ; do jockey kick centos64 @core-0$i 10.1.1.$i core-0$i; done

Start the kickstart servers:

    jockey http
    jockey masq

and the machines:

    for i in 1 2 3 ; do flock-vbox start core-0$i; done

restart after installation is ready:

    for i in 1 2 3 ; do flock-vbox off core-0$i; done
    for i in 1 2 3 ; do flock-vbox boot core-0$i disk; done
    for i in 1 2 3 ; do flock-vbox start core-0$i; done

In case of real servers you have to play with ipmi.

## Provision with Ansible
You need an inventory file with your hosts, edit `hosts`

    [core]
    core-01 ansible_ssh_host=10.1.1.1
    core-02 ansible_ssh_host=10.1.1.2
    core-03 ansible_ssh_host=10.1.1.3

Test the 1st server:

    jockey password @core-01
    flock ping root@core-01

### Boostrap
Bootstrap the System Operator:

    for i in 1 2 3 ; do jockey password @core-0$i; flock play root@core-0$i bootstrap; done

and ping the by `sysop`:

    flock ping @@core

### Ground State
You need a simple network topology (`networks.yml`):

    interfaces:
      bmc: eth0
      system: eth0
      external: eth1
      dhcp: eth1
    networks:
      bmc: 10.0.0.0
      system: 10.1.0.0
      compute: 10.1.1.0
      vpn: 10.9.0.0
    masks:
      system: 16
      compute: 24
      home: 24
      vpn: 24
    dhcp_masks:
      system: 255.255.0.0
      compute: 255.255.255.0
    broadcasts:
      system: 10.1.255.255
    sysops:
      - 10.1.1.254
      - 10.1.1.253
    master: core-01

Currently, we have role-like playbooks. First, secure the installation:

    flock play @@core secure

Due to selinux you have to reboot now:

    flock reboot @@core

Now, reach the ground state:

    flock play @@core ground
    flock reboot @@core

Mind that the system network is not protected, due to performance reasons core servers can reach each other wihtout any serious authentication. Ground monitoring is done by Ganglia in multicast mode.

Check the cluster state by or on `http://10.1.1.1/ganglia`:

    gstat -a

For the awesome PCP download `ftp://oss.sgi.com/projects/pcp/download/mac/` and install `pcp` and `pcp-gui`. Get realtime statistics:

    pmstat -h 10.1.1.1 -h 10.1.1.2 -h 10.1.1.3
    /Applications/pmchart.app/Contents/MacOS/pmchart -h 10.1.1.1 -c Overview 

### Globus CA
Install the certificate utilities and Globus on your mac:

    make globus_simple_ca globus_gsi_cert_utils

There is a hash mismatch between OpenSSL 0.9 and 1.X. Install newer OpenSSL on your mac. You can use the [NCE module/package manager](https://github.com/NIIF/nce). Load the Globus and the new OpenSSL environment:

    module load globus openssl

The Grid needs a PKI, which protects access and the communication. You can create as many CA as you like. It is advised to make many short-term flat CAs. Edit grid scripts as well as templates in `share/globus_simple_ca` if you want to change key parameters. Create a Core CA:

    flock-ca create coreca 365 sysop@localhost

The new CA is created under the `ca/coreca` directory. The CA certificate is installed under `ca/grid-security` to make requests easy. If you compile Globus with the old OpenSSL (system default) you have to use old-style subject hash. Create old CA hash by:

    flock-ca oldhash

Edit `coreca/grid-ca-ssl.conf` and add the following line under `policy` in `CA_default` section, this enables extension copy on sign and let alt names go.

    copy_extensions = copy

Request & sign host certificates:

    for i in 1 2 3 ; do flock-ca host coreca core-0$i; done
    for i in 1 2 3 ; do flock-ca sign coreca core-0$i; done

Certs, private keys and requests are in `ca/coreca/grid-security`. There is also a `ca/<CAHASH>` directory link for each CA. You have to use the `<CAHASH>` in the playbooks. Get the `<CAHASH>`:

    flock-ca cahash coreca

Edit `roles/globus/vars/globus.yml` and set the default CA hash.

Create and sign the sysop certificate:

    flock-ca user coreca sysop "System Operator"
    flock-ca sign coreca sysop

In order to use `sysop` as a default grid user you have to copy cert and key into the `keys` directory:

    flock-ca keys coreca sysop

Create a pkcs12 version if you need for the browser (this command works in the `keys` directory):

    flock-ca p12 sysop

Test your user certificate (you might have to create the old hash):

    flock-ca verify coreca sysop

Enter Grid state:

    flock play @@core grid

Check `ssl.conf` for a strong [PFS](http://vincent.bernat.im/en/blog/2011-ssl-perfect-forward-secrecy.html) cipher setting.

### Clustering
Create common authentication key (`keys/authkey`):

    dd if=/dev/urandom of=keys/authkey bs=128 count=1

Enter Cluster state:

    flock play @@core cluster

The following tools are installed under `/root/bin`:

    ring
    totem
    quorum

### Database
#### MariaDB with Galera
Install database:

    flock play @@core database

Login to the master node and secure the installation:

    mysql_secure_installation

### Storage
#### Gluster
Change to the latest mainline kernel:

    flock play @@core roles/system/kernel
    flock reboot @@core

Install Gluster and setup a 3-node FS cluster:

    flock play @@core roles/cluster/gluster

Login to the master node and bootstrap the cluster:

    /root/gluster_bootstrap

Finally, mount the common directory:

    flock play @@core roles/cluster/glusterfs
    flock play @@core roles/cluster/gtop

Monitor the cluster:

    /root/bin/gtop

### Scheduler
#### Slurm
Generate the Munge auth key:

    dd if=/dev/random bs=1 count=1024 > keys/munge.key

Install Slurm:

    flock play @@core scheduler

Login to the master node and test the queue:

    srun -N 3 hostname

### VPN
Download Easy RSA CA:

    git clone git://github.com/OpenVPN/easy-rsa.git ca/easy-rsa

Create a VPN CA (`vpnca`):

    flock-vpn create

Create server certificates:

    for i in 1 2 3 ; do flock-vpn server vpnca core-0$i; done

Create sysop client certificate:

    flock-vpn client vpnca sysop

Create DH and TA parameters:

    flock-vpn param vpnca

You need the following files:

Filename | Needed By | Purpose | Secret
--- | --- | --- | ---
ca.crt | server & clients | Root CA cert | NO
ca.key | sysop | Root CA key | YES
ta.key | server & clients | HMAC | YES
dh{n}.pem | server | DH parameters | NO
server.crt | server | Server Cert | NO
server.key | server | Server Key | YES
client.crt | client | Client Cert | NO
client.key | client | Client Key | YES

Install OpenVPN servers:

    flock play @@core roles/vpn/openvpn

Install Tunnelblick on your mac and link:

    pushd $HOME
    ln -s 'Library/Application Support/Tunnelblick/Configurations' .openvpn
    popd

Install the sysop certificate for Tunnelblick:

    flock-vpn blick vpnca sysop

#### Ting
You can determine server IPs by Ting. Register a free [pusher.com](https://github.com/NIIF/nce) account. Generate a key:

    openssl rand -base64 32 > keys/ting.key

Create `keys/ting.yml` with the following content:

    ting:
      key: ...
      secret: ...
      app_id: ...

Enable the role:

    flock play @@core roles/vpn/ting

Install Ting on your mac:

    pushd $HOME
    git clone git://github.com/hornos/ting.git

Edit the `ting.yml` and fill in your app details.

The ting service will pong back the machines external IP ans SSH host fingerprints. On your local machine start the monitor:

    pushd $HOME/ting
    ./ting -k ../flock/keys/ting.key -m client

Ping them

    ./ting -k ../flock/keys/ting.key ping

Hosts are collected in `ting/hosts` as json files. Create an Tunnelblick client configuration based on Ting pongs:

    pushd $HOME/flock
    flock-vpn ting core? core

Now connect with Tunnelblick.

### Warewulf
Warewulf is a badass HPC cluster kit. If you have a standalone master disable galera wsrep:

    flock play @@core roles/database/mariadb_master.yml

Put on the mask:

    flock play @@core warewulf

Rerun clones:

    flock play @@core roles/warewulf/ansible.yml --tag clone --extra-vars='master=core'

Enable health check (TODO):

    flock play @@core roles/warewulf/healthcheck

Start NFS server:

    flock play @@core roles/warewulf/nfsserver

Create compute VMs:

     for i in 1 2 3; do flock-vbox create cn-0$i;done

Login to the master node and make a child:

    pushd /common/warewulf/chroots
    ./cloneos centos-6

Install basic packages (NTP, Munge, Slurm):

    ./clonepackages centos-6

Generate cluster keys:

    ssh-keygen -b2048 -N "" -f playbooks/keys/cluster

Configure the clone directory (TODO one playbook):

    ./clonesetup centos-6

TODO: LDAP & storage

Enable services (TODO: from playbook):

    chroot centos-6 chkconfig munge on
    chroot centos-6 chkconfig ntpd on

TODO: update/create warewulf packages
FIX
FIX Edit /usr/bin/wwvnfs exclude_files part
FIX

    change foreach my $file (glob("$tmpdir/$line")) {
    to foreach my $file (glob("$line")) {

(re)Make the image:

    ./cloneimage centos-6

Bootstrap the kernel:

    ./clonekernel list centos-6
    ./clonekernel centos-6 2.6.32-358.11.1.el6.x86_64

Mainline kernel:

    ./cloneyum centos-6 --enablerepo=elrepo-kernel install kernel-ml
    ./clonekernel list centos-6
    ./clonekernel centos-6 3.9.7-1.el6.elrepo.x86_64
    wwsh provision set cn-01 -b 3.9.7-1.el6.elrepo.x86_64

UPX (TODO):

    ./clonemini centos-6
    ./cloneimage centos-6-mini
    wwsh provision set cn-01 -v centos-6-mini

Provision:

    ./clonescan centos-6/2.6.32-358.11.1.el6.x86_64 compute/cn-0[1-3]

Start the VMs.

Reconfigure the scheduler, `slurmconf` copies pro/epi scripts as well:

    ./slurmconf cn-0[1-3]

Parameter       | srun option | When Run  | Run by | As User
--- | --- | --- | ---
PrologSlurmCtld |             | job start | slurmctld | SlurmUser
Prolog          |             | job start | slurmd    | SlurmdUser
TaskProlog (batch) |          | script start | slurmstepd | User
SrunProlog      | --prolog    | step start | srun     | User
TaskProlog      |             | step start | slurmstepd | User
                | --task-prolog | step start | slurmstepd | User
                | --task-epilog | step finish | slurmstepd | User
TaskEpilog      |             | step finish | slurmstepd | User
SrunEpilog      | --epilog    | step finish | srun  | User
TaskEpilog (batch) |          | script finish | slurmstepd | User
Epilog          |             | job finish | slurmd | SlurmdUser
EpilogSlurmCtld |             | job finish | slurmctld | SlurmUser

Mind the time if needed:

    ./cloneservice compute ntpd restart

Try them out (TODO):

    srun -N 2 -D /common/scratch hostname

Batch test:

    cd /common/scratch
    Create slurm.sh:
    #!/bin/sh 
    #SBATCH -o StdOut
    #SBATCH -e StdErr
    srun hostname

    sbatch -N 2 slurm.sh

MPI test on the host:

    git clone git://github.com/NIIF/nce.git /common/software/nce
    module use /common/software/nce/modulefiles/
    export NCE_ROOT=/common/software/nce
    module load nce/global

Compile the latest OpenMPI:

    yum install gcc gcc-c++ make gcc-gfortran environment-modules
    mkdir -p ${NCE_PACKAGES}/openmpi/1.6.4
    ./configure --prefix=${NCE_PACKAGES}/openmpi/1.6.4
    make && make install
    cd examples
    make
    cd ..
    cp -R examples /common/scratch/
    cd /common/scratch/examples
    module load openmpi/1.6.4

Performance tests (TODO):

    ftp://ftp.mcs.anl.gov/pub/mpi/tools/perftest.tar.gz

[Soft RoCE](http://www.systemfabricworks.com/downloads/roce) (TODO):

    yum --enablerepo=elrepo-kernel install kernel-ml-devel kernel-ml-headers rpm-build
    yum install libibverbs-devel libibverbs-utils libibverbs
    yum install librdmacm-devel librdmacm-utils librdmacm

Downlod rxe package:

    wget http://198.171.48.62/pub/OFED-1.5.2-rxe.tgz

Edit `/etc/exports`, export fs by `exportfs -a`. Edit Push out a new `autofs.common` config:

    ./autofsconf compute

Service restart:

    ./cloneservice compute gmond restart

TODO
* Cgroup
* Wake-on-lan with compute suspend
* Kernel 3.10 full nohz cpuset scheduler numa shit ptp
* error: /usr/sbin/nhc: exited with status 0x0100

### Message Queue

### Hadoop
#### CDH4

### Communication Center
Install MariaDB master:

    flock play @@com roles/database/mariadb --extra-vars "master=com"
    flock play @@com roles/database/mariadb_master --extra-vars "master=com"

Log in and secure:

    mysql_secure_installation

Install openfire:

    flock play @@com roles/com/openfire --extra-vars \"schema=yes master=com\"

Setup the wizard:

    jdbc:mysql://127.0.0.1:3306/openfire?rewriteBatchedStatements=true

Clustering:

SSL:

## Kernel
### Kdump
Remote kdump
http://blog.kreyolys.com/2011/03/17/no-panic-its-just-a-kernel-panic/

## Home Server

## XenServer 6.2
Mount (link) the ISO under `boot/xs62/repo` into jockey's root directory.

    flock-vbox create xs62 RedHat_64 2 2048
    jockey kick xs62 @xs62 10.1.1.30 xs62

Controller:

    export ANSIBLE_HOSTS=xen
    flock play root@xc bootstrap
    flock play @@xc secure
    flock reboot @@xc
    flock play @@xc ground
    flock reboot @@xc

    flock play @@xc roles/database/mariadb --extra-vars "master=xc"
    flock play @@xc roles/database/mariadb_master --extra-vars "master=xc"
    > mysql_secure_installation
    flock play @@xc roles/database/memcache.yml
    flock play @@xc roles/mq/rabbitmq.yml
    flock play @@xc roles/monitor/icinga.yml --extra-vars \"schema=yes master=xc\"

Create admin token:

    openssl rand -hex 10 > keys/admin_token


