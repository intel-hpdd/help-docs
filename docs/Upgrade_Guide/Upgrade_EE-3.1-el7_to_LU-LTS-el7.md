# Upgrading Intel EE Lustre 3.1.1.0 to Lustre 2.10 LTS and IML 4.0

## Introduction

This document provides a description of how to upgrade an existing Lustre server file system installation from Intel EE Lustre version 3.1.1.0 running on the RHEL/CentOS 7 OS distribution to Lustre 2.10 LTS and Manager for Lustre version 4 running on RHEL/CentOS 7.

CentOS is used for the examples. RHEL users will need to refer to Red Hat for instructions on enabling the High Availability add-on needed to install Pacemaker, Corosync and related support tools.

## Risks

The process of upgrading Intel EE Lustre to a newer Lustre and Manager for Lustre version requires careful consideration and planning. There will always be some disruption to services when major maintenance works are undertaken, although this can be contained and minimized.

The reference platform used throughout the documentation has been installed and is being managed by Intel Manager for Lustre (IML), but the methods for the Lustre server components can be broadly applied to any Lustre server environment running the RHEL or CentOS OS.

## Process Overview

1. Upgrade the IML manager server
    1. Backup the Intel Manager for Lustre server (IML manager) configuration and database
    1. Install the latest version of the IML manager software for EL7
1. Upgrade the metadata and object storage server pairs. For each HA pair:
    1. Backup the server configuration for both machines
    1. Machines are upgraded one at a time
        1. Failover all resources to a single node, away from the node being upgraded
        1. Set the node to standby mode and disable the node in Pacemaker
        1. Unload the Lustre kernel modules
        1. Unload the ZFS and SPL kernel modules, if used
        1. Install the new EE software
        1. Move the Pacemaker resources from the secondary node to the upgraded node
        1. Update the secondary node
        1. Re-enable the secondary node in the Pacemaker configuration
        1. Re-balance the resources

## Upgrade Intel Manager for Lustre

The first component in the environment to upgrade is the manager for Lustre software itself. The manager server upgrade can be conducted without any impact to the Lustre file system services.

### Backup the Existing IML Configuration

1. Prior to commencing the upgrade, create a backup of the existing configuration. This will enable recovery of the original configuration in the event of a problem occurring during execution of the upgrade.

    The following shell script can be used to capture the essential configuration information that is relevant to IML itself:

    ```bash
    #!/bin/sh
    # EE Intel Manager for Lustre (IML) server backup script

    BCKNAME=bck-$HOSTNAME-`date +%Y%m%d-%H%M%S`
    BCKROOT=$HOME/$BCKNAME
    mkdir -p $BCKROOT
    tar cf - --exclude=/var/lib/chroma/repo \
    /var/lib/chroma \
    /etc/sysconfig/network \
    /etc/sysconfig/network-scripts/ifcfg-* \
    /etc/yum.conf \
    /etc/yum.repos.d \
    /etc/hosts \
    /etc/passwd \
    /etc/group \
    /etc/shadow \
    /etc/gshadow \
    /etc/sudoers \
    /etc/resolv.conf \
    /etc/nsswitch.conf \
    /etc/rsyslog.conf \
    /etc/ntp.conf \
    /etc/selinux/config \
    /etc/ssh \
    /root/.ssh \
    | (cd $BCKROOT && tar xf -)

    # IML Database
    su - postgres -c "/usr/bin/pg_dumpall --clean" | /bin/gzip > $BCKROOT/pgbackup-`date +\%Y-\%m-\%d-\%H:\%M:\%S`.sql.gz

    cd `dirname $BCKROOT`
    tar zcf $BCKROOT.tgz `basename $BCKROOT`
    ```

1. Copy the backup tarball to a safe location that is not on the server being upgraded.

**Note:** The script is not intended to provide a comprehensive backup of the entire operating system configuration. It covers the essential components pertinent to Lustre servers managed by IML that are difficult to re-create if deleted.

### Install the Manager for Lustre Upgrade

The software upgrade process requires super-user privileges to run. Login as the `root` user or use `sudo` to elevate privileges as required.

1. If upgrading from e.g. EL 7.3 to EL 7.4, run the OS upgrade first. For example:

    ```bash
    yum clean all
    yum update
    reboot
    ```

    Refer to the operating system documentation for details on the correct procedure for upgrading between minor OS releases.

1. Download the latest manager for lustre software from the project's release page:

    <https://github.com/intel-hpdd/intel-manager-for-lustre/releases>

1. Extract the Manager for Lustre bundle. For example:

    ```bash
    cd $HOME
    tar zxf iml-4.0.0.0.tar.gz
    ```

1. As root, run the installer:

    ```bash
    cd $HOME/iml-*
    ./install [-h] | [-e] [-r] [-p] [-d]
    ```

    **Note:** For an explanation of the installation options, run the install script with the `--help` flag:

    ```bash
    usage: install [-h] [--no-epel-check] [--no-repo-check] \
    [--no-platform-check] [--no-dbspace-check]

    optional arguments:
    -h, --help            show this help message and exit
    --no-epel-check, -e
    --no-repo-check, -r
    --no-platform-check, -p
    --no-dbspace-check, -d
    ```

1. The installation program will detect that there is a previous installation and will run the upgrade process.
1. When the upgrade is complete, connect to the manager for lustre service using a web browser and verify that the upgraded version has been installed and is running. IML Agents running version EE 3.1 will still be able to communicate with the new version of the manager service.

### Create Local Repositories for the Lustre Packages

The Manager for Lustre distribution does not include Lustre software packages. These need to be acquired separately from the Lustre project's download server. The following instructions will establish the manager node as a YUM repository host for the Lustre packages. The Manager for Lustre server is automatically configured as a web server, so it is a convenient location for the repository mirrors.

1. Create a temporary YUM repository definition. This will be used to assist with the initial acquisition of Lustre and related packages.

    ```bash
    cat >/tmp/lustre-repo.conf <<\__EOF
    [lustre-server]
    name=lustre-server
    baseurl=https://downloads.hpdd.intel.com/public/lustre/latest-release/el7/server
    exclude=*debuginfo*
    gpgcheck=0

    [lustre-client]
    name=lustre-client
    baseurl=https://downloads.hpdd.intel.com/public/lustre/latest-release/el7/client
    exclude=*debuginfo*
    gpgcheck=0

    [e2fsprogs-wc]
    name=e2fsprogs-wc
    baseurl=https://downloads.hpdd.intel.com/public/e2fsprogs/latest/el7
    exclude=*debuginfo*
    gpgcheck=0
    __EOF
    ```

    **Note:** The above example references the latest Lustre release available. To use a specific version, replace `latest-release` in the `[lustre-server]` and `[lustre-client]` `baseurl` variables with the version required, e.g., `lustre-2.10.1`. Always use the latest `e2fsprogs` package unless directed otherwise.

    **Note:** With the release of Lustre version 2.10.1, it is possible to use patchless kernels for Lustre servers running LDISKFS. The patchless LDISKFS server distribution does not include a Linux kernel. Instead, patchless servers will use the kernel distributed with the operating system. To use patchless kernels for the Lustre servers, replace the string `server` with `patchless-ldiskfs-server` at the end of the `[lustre-server]` `baseurl` string. For example:

    ```bash
    baseurl=https://downloads.hpdd.intel.com/public/lustre/latest-release/el7/patchless-ldiskfs-server
    ```

    Also note that the `debuginfo` packages are excluded in the example repository definitions. This is simply to cut down on the size of the download. It is usually a good idea to pull in these files as well, to assist with debugging of issues.

1. Use the `reposync` command (distributed in the `yum-utils` package) to download mirrors of the Lustre repositories to the manager server:

    ```bash
    cd /var/lib/chroma/repo
    reposync -c /tmp/lustre-repo.conf -n \
    -r lustre-server \
    -r lustre-client \
    -r e2fsprogs-wc
    ```

1. Create the repository metadata:

    ```bash
    cd /var/lib/chroma/repo
    for i in e2fsprogs-wc lustre-client lustre-server; do
    (cd $i && createrepo .)
    done
    ```

1. Create a YUM Repository Definition File

    The following script creates a file containing repository definitions for the IML Agent software and the Lustre packages downloaded in the previous section. Review the content and adjust according to the requirements of the target environment. Run the script on the upgraded Manager for Lustre host:

    ```bash
    hn=`hostname --fqdn`
    cat >/var/lib/chroma/repo/Manager-for-Lustre.repo <<__EOF
    [iml-agent]
    name=Intel Manager for Lustre Agent
    baseurl=https://$hn/repo/iml-agent/7
    enabled=0
    gpgcheck=0
    sslverify = 1
    sslcacert = /var/lib/chroma/authority.crt
    sslclientkey = /var/lib/chroma/private.pem
    sslclientcert = /var/lib/chroma/self.crt
    proxy=_none_

    [lustre-server]
    name=lustre-server
    baseurl=https://$hn/repo/lustre-server
    enabled=0
    gpgcheck=0
    sslverify = 1
    sslcacert = /var/lib/chroma/authority.crt
    sslclientkey = /var/lib/chroma/private.pem
    sslclientcert = /var/lib/chroma/self.crt
    proxy=_none_

    [lustre-client]
    name=lustre-client
    baseurl=https://$hn/repo/lustre-client
    enabled=0
    gpgcheck=0
    sslverify = 1
    sslcacert = /var/lib/chroma/authority.crt
    sslclientkey = /var/lib/chroma/private.pem
    sslclientcert = /var/lib/chroma/self.crt
    proxy=_none_

    [e2fsprogs-wc]
    name=e2fsprogs-wc
    baseurl=https://$hn/repo/e2fsprogs-wc
    enabled=0
    gpgcheck=0
    sslverify = 1
    sslcacert = /var/lib/chroma/authority.crt
    sslclientkey = /var/lib/chroma/private.pem
    sslclientcert = /var/lib/chroma/self.crt
    proxy=_none_
    __EOF
    ```

    This file is distributed to each of the Lustre servers during the upgrade process to facilitate installation of the software.

    Make sure that the `$hn` variable matches the host name that will be used by the Lustre servers to access the manager for lustre host.

## Upgrade the Lustre Servers

Lustre server upgrades can be coordinated as either an online roll-out, leveraging the failover HA mechanism to migrate services between nodes and minimize disruption, or as an offline service outage, which has the advantage of usually being faster to deploy overall, with generally lower risk.

The upgrade procedure documented here shows how to execute the upgrade while the file system is online. It assumes that the Lustre servers have been installed in pairs, where each server pair forms an independent high-availability cluster built on Pacemaker and Corosync. Intel Manager for Lustre deploys these configurations and includes its own resource agent for managing Lustre assets, called `ocf:chroma:Target`. IML can also configure STONITH agents to provide node fencing in the event of a cluster partition or loss of quorum.

This documentation will demonstrate how to upgrade a single HA pair of Lustre servers. The process needs to be repeated for all servers in the cluster. It is possible to execute this upgrade while services are still online, with only minor disruption during critical phases. Nevertheless, where possible it is recommended that the upgrade operation is conducted during a planned maintenance window with the file system stopped.

In the procedure, "Node 1" or "first node" will be used to refer to the first server in each HA cluster to upgrade, and "Node 2" or "second node" will be used to refer to the second server in each HA cluster pair.

Upgrade one server at a time in each cluster pair, starting with Node 1, and make sure the upgrade is complete on one server before moving on to the second server in the pair.

The software upgrade process requires super-user privileges to run. Login as the `root` user or use `sudo` to elevate privileges as required to complete tasks.

**Note:** It is recommended that Lustre clients are quiesced prior to executing the upgrade. This reduces the risk of disruption to the upgrade process itself. It is not usually necessary to unmount the Lustre clients, although this is a sensible precaution.

### Prepare for the Upgrade

Upgrade one server at a time in each cluster pair, starting with Node 1, and make sure the upgrade is complete on one server before moving on to the second server in the pair.

1. As a precaution, create a backup of the existing configuration for each server. The following shell script can be used to capture the essential configuration information that is relevant to IML managed mode servers:

    ```bash
    #!/bin/sh
    BCKNAME=bck-$HOSTNAME-`date +%Y%m%d-%H%M%S` BCKROOT=$HOME/$BCKNAME
    mkdir -p $BCKROOT
    tar cf - \
    /var/lib/chroma \
    /etc/sysconfig/network \
    /etc/sysconfig/network-scripts/ifcfg-* \
    /etc/yum.conf \
    /etc/yum.repos.d \
    /etc/hosts \
    /etc/passwd \
    /etc/group \
    /etc/shadow \
    /etc/gshadow \
    /etc/sudoers \
    /etc/resolv.conf \
    /etc/nsswitch.conf \
    /etc/rsyslog.conf \
    /etc/ntp.conf \
    /etc/selinux/config \
    /etc/modprobe.d/iml_lnet_module_parameters.conf \
    /etc/corosync/corosync.conf \
    /etc/ssh \
    /root/.ssh \
    | (cd $BCKROOT && tar xf -)

    # Pacemaker Configuration:
    cibadmin --query > $BCKROOT/cluster-cfg-$HOSTNAME.xml

    cd `dirname $BCKROOT`
    tar zcf $BCKROOT.tgz `basename $BCKROOT`
    ```

    **Note:** This is not intended to be a comprehensive backup of the entire operating system configuration. It covers the essential components pertinent to Lustre servers managed by IML that are difficult to re-create if deleted.

1. Copy the backups for each server's configuration to a safe location that is not on the servers being upgraded.

### Online Upgrade

The upgrade procedure documented here shows how to execute the upgrade while the file system is online. It assumes that the Lustre servers have been installed by Intel Manager for Lustre in pairs, where each server pair forms an independent high-availability cluster built on Pacemaker and Corosync.

#### Migrate the Resources on Node 1

1. Login to node 1 and failover all resources to the standby node. The easiest way to do this is to set the server to standby mode:

    ```bash
    pcs cluster standby [<node name>]
    ```

    **Note:** if the node name is omitted from the command, only the node currently logged into will be affected.

1. Verify that the resources have moved to the second node and that the resources are running (all storage targets are mounted on the second node only):

    ```bash
    # On node 2:
    pcs resource show
    df -ht lustre
    ```

1. Disable the node that is being upgraded:

    ```bash
    pcs cluster disable [<node name>]
    ```

    **Note:** This command prevents the Corosync and Pacemaker services from starting automatically on system boot. The command is a simple and useful way to ensure that when the server is rebooted, it won't try to join the cluster and disrupt services running on the secondary node. If the node name is omitted from the command, only the node currently logged into will be affected.

#### [If Needed] Upgrade the OS on Node 1

1. Login to node 1.
1. If upgrading from e.g. EL 7.3 to EL 7.4, run the OS upgrade first. For example:

    ```bash
    yum clean all
    yum update
    reboot
    ```

    Refer to the operating system documentation for details on the correct procedure for upgrading between minor OS releases.

#### Upgrade the Manager for Lustre Agent on Node 1

1. Login to node 1.
1. Install EPEL repository support:

    ```bash
    yum -y install epel-release
    ```

1. Install the Manager for Lustre COPR Repository definition, which contains some dependencies for the IML Agent:

    ```bash
    yum-config-manager --add-repo \
    https://copr.fedorainfracloud.org/coprs/managerforlustre/manager-for-lustre/repo/epel-7/managerforlustre-manager-for-lustre-epel-7.repo
    ```

1. Install the DNF project COPR Repository definition. DNF is a package manager, and is used as a replacement for YUM in many distributions, such as Fedora. It does not replace YUM in CentOS, but Manager for Lustre does make use of some of the newer features DNF for some of its tasks:

    ```bash
    yum-config-manager --add-repo \
    https://copr.fedorainfracloud.org/coprs/ngompa/dnf-el7/repo/epel-7/ngompa-dnf-el7-epel-7.repo
    ```

1. Remove or rename the old IML YUM repository definition:

    ```bash
    mv /etc/yum.repos.d/Intel-Lustre-Agent.repo \
    $HOME/Intel-Lustre-Agent.repo.bak
    ```

1. Install the IML Agent repository definition:

    ```bash
    curl -o /etc/yum.repos.d/Manager-for-Lustre.repo \
    --cacert /var/lib/chroma/authority.crt \
    --cert /var/lib/chroma/self.crt \
    --key /var/lib/chroma/private.pem \
    https://<admin server>/repo/Manager-for-Lustre.repo
    ```

    Replace `<admin server>` in the `https` URL with the appropriate manager for lustre hostname (normally the fully-qualified domain name).

1. Upgrade the IML Agent and Diagnostics packages

    ```bash
    yum -y --enablerepo=iml-agent install chroma-\*
    ```

    **Note:** following warning during install / update is benign and can be ignored:

    ```bash
    ...
      /var/tmp/rpm-tmp.MV5LGi: line 5: /selinux/enforce: No such file or directory
      []

      {
        "result": null,
        "success": true
      }
    ...
    ```

#### Upgrade the Lustre Server Software on Node 1

1. Stop ZED and unload the ZFS and SPL modules:

    ```bash
    systemctl stop zed
    rmmod zfs zcommon znvpair spl
    ```

1. Remove the Lustre, ZFS and SPL packages, as they will be replaced with upgraded packages. The upgrade process will fail if these packages are not preemptively removed:

    ```bash
    yum erase zfs-dkms spl-dkms \
    lustre \
    lustre-modules \
    lustre-osd-ldiskfs \
    lustre-osd-zfs \
    lustre-osd-ldiskfs-mount \
    lustre-osd-zfs-mount \
    libzpool2 \
    libzfs2
    ```

1. Clean up the YUM cache to remove any residual information on old repositories and package versions:

    ```bash
    yum clean all
    ```

    **Note:** If an error similar to the following is observed when installing packages at any point in the process, re-run the `yum clean all` command, then re-try the install:

    ```bash
    libzfs2-0.6.5.7-1.el7.x86_64.r FAILED
    http://download.zfsonlinux.org/epel/7.3/x86_64/libzfs2-0.6.5.7-1.el7.x86_64.rpm: [Errno 14] HTTP Error 404 - Not Found
    Trying other mirror.
    To address this issue please refer to the below knowledge base article

    https://access.redhat.com/articles/1320623

    If above article doesn't help to resolve this issue please create a bug on https://bugs.centos.org/

    ...

    spl-dkms-0.6.5.7-1.el7.noarch. FAILED
    http://download.zfsonlinux.org/epel/7.3/x86_64/spl-dkms-0.6.5.7-1.el7.noarch.rpm: [Errno 14] HTTP Error 404 - Not Found
    Trying other mirror.
    libzpool2-0.6.5.7-1.el7.x86_64 FAILED
    http://download.zfsonlinux.org/epel/7.3/x86_64/libzpool2-0.6.5.7-1.el7.x86_64.rpm: [Errno 14] HTTP Error 404 - Not Found
    Trying other mirror.
    zfs-dkms-0.6.5.7-1.el7.noarch. FAILED
    http://download.zfsonlinux.org/epel/7.3/x86_64/zfs-dkms-0.6.5.7-1.el7.noarch.rpm: [Errno 14] HTTP Error 404 - Not Found
    Trying other mirror.

    Error downloading packages:
    spl-dkms-0.6.5.7-1.el7.noarch: [Errno 256] No more mirrors to try.
    libzpool2-0.6.5.7-1.el7.x86_64: [Errno 256] No more mirrors to try.
    zfs-dkms-0.6.5.7-1.el7.noarch: [Errno 256] No more mirrors to try.
    libzfs2-0.6.5.7-1.el7.x86_64: [Errno 256] No more mirrors to try.
    ```

1. For systems that have **both** LDISKFS and ZFS OSDs:

    **Note:** This is the configuration that IML installs for all managed-mode Lustre storage clusters. Use this configuration for the highest level of compatibility with Manager for Lustre.

    1. Install the Lustre `e2fsprogs` distribution:

        ```bash
        yum --nogpgcheck --disablerepo=* --enablerepo=e2fsprogs-wc \
        install e2fsprogs e2fsprogs-devel
        ```

    1. If the installed `kernel-tools` and  `kernel-tools-libs` packages are at a higher revision than the patched kernel packages in the Lustre server repository, they will need to be removed:

        ```bash
        yum erase kernel-tools kernel-tools-libs
        ```

    1. Install the Lustre-patched kernel packages. Ensure that the Lustre repository is picked for the kernel packages, by disabling the OS repos:

        ```bash
        yum --nogpgcheck --disablerepo=base,extras,updates \
        --enablerepo=lustre-server install \
        kernel \
        kernel-devel \
        kernel-headers \
        kernel-tools \
        kernel-tools-libs \
        kernel-tools-libs-devel
        ```

    1. Install additional development packages. These are needed to enable support for some of the newer features in Lustre – if certain packages are not detected by Lustre's configure script when its DKMS package is installed, the features will not be enabled when Lustre is compiled. Notable in this list are `krb5-devel` and `libselinux-devel`, needed for Kerberos and SELinux support, respectively.

        ```bash
        yum install asciidoc audit-libs-devel automake bc \
        binutils-devel bison device-mapper-devel elfutils-devel \
        elfutils-libelf-devel expect flex gcc gcc-c++ git \
        glib2 glib2-devel hmaccalc keyutils-libs-devel krb5-devel ksh \
        libattr-devel libblkid-devel libselinux-devel libtool \
        libuuid-devel libyaml-devel lsscsi make ncurses-devel \
        net-snmp-devel net-tools newt-devel numactl-devel \
        parted patchutils pciutils-devel perl-ExtUtils-Embed \
        pesign python-devel redhat-rpm-config rpm-build systemd-devel \
        tcl tcl-devel tk tk-devel wget xmlto yum-utils zlib-devel
        ```

    1. Ensure that a persistent hostid has been generated on the machine. If necessary, generate a persistent hostid (needed to help protect zpools against simultaneous imports on multiple servers). For example:

        ```bash
        hid=`[ -f /etc/hostid ] && od -An -tx /etc/hostid|sed 's/ //g'`
        [ "$hid" = `hostid` ] || genhostid
        ```

    1. Reboot the node.

        ```bash
        reboot
        ```

    1. Install Lustre, and the LDISKFS and ZFS `kmod` packages:

        ```bash
        yum --nogpgcheck --enablerepo=lustre-server install \
        kmod-lustre-osd-ldiskfs \
        lustre-dkms \
        lustre-osd-ldiskfs-mount \
        lustre-osd-zfs-mount \
        lustre \
        lustre-resource-agents \
        zfs
        ```

    1. Verify that the DKMS kernel modules for Lustre, SPL and ZFS have installed correctly:

        ```bash
        dkms status
        ```

        All packages should have the status `installed`. For example:

        ```bash
        # dkms status
        lustre, 2.10.1, 3.10.0-693.2.2.el7_lustre.x86_64, x86_64: installed
        spl, 0.7.1, 3.10.0-693.2.2.el7_lustre.x86_64, x86_64: installed
        zfs, 0.7.1, 3.10.0-693.2.2.el7_lustre.x86_64, x86_64: installed
        ```

    1. Load the Lustre and ZFS kernel modules to verify that the software has installed correctly:

        ```bash
        modprobe -v zfs
        modprobe -v lustre
        ```

1. For systems that use LDISKFS OSDs:
    1. Upgrade the Lustre `e2fsprogs` distribution:

        ```bash
        yum --nogpgcheck \
        --disablerepo=* --enablerepo=e2fsprogs-wc install e2fsprogs
        ```

    1. If the installed `kernel-tools` and  `kernel-tools-libs` packages are at a higher revision than the patched kernel packages in the Lustre server repository, they will need to be removed:

        ```bash
        yum erase kernel-tools kernel-tools-libs
        ```

    1. Install the Lustre-patched kernel packages. Ensure that the Lustre repository is picked for the kernel packages, by disabling the OS repos:

        ```bash
        yum --nogpgcheck --disablerepo=base,extras,updates \
        --enablerepo=lustre-server install \
        kernel \
        kernel-devel \
        kernel-headers \
        kernel-tools \
        kernel-tools-libs \
        kernel-tools-libs-devel
        ```

    1. Reboot the node.

        ```bash
        reboot
        ```

    1. Install the LDISKFS `kmod` and other Lustre packages:

        ```bash
        yum --nogpgcheck --enablerepo=lustre-server install \
        kmod-lustre \
        kmod-lustre-osd-ldiskfs \
        lustre-osd-ldiskfs-mount \
        lustre \
        lustre-resource-agents
        ```

    1. Load the Lustre kernel modules to verify that the software has installed correctly:

        ```bash
        modprobe -v lustre
        ```

1. For systems with ZFS-based OSDs:
    1. Follow the instructions from the ZFS on Linux project to [install the ZFS YUM repository definition](https://github.com/zfsonlinux/zfs/wiki/RHEL-%26-CentOS). Use the DKMS package repository (the default).

        For example:

        ```bash
        yum -y install \
        http://download.zfsonlinux.org/epel/zfs-release.el7_4.noarch.rpm
        ```

    1. Clean the YUM cache:

        ```bash
        yum clean all
        ```

    1. Install the kernel packages that match the latest supported version for the Lustre release:

        ```bash
        yum install \
        kernel \
        kernel-devel \
        kernel-headers \
        kernel-tools \
        kernel-tools-libs \
        kernel-tools-libs-devel
        ```

        It may be necessary to specify the kernel package version number in order to ensure that a kernel that is compatible with Lustre is installed. For example, Lustre 2.10.0 has support for RHEL kernel 3.10.0-514.21.1.el7:

        ```bash
        yum install \
        kernel-3.10.0-514.21.1.el7 \
        kernel-devel-3.10.0-514.21.1.el7 \
        kernel-headers-3.10.0-514.21.1.el7 \
        kernel-tools-3.10.0-514.21.1.el7 \
        kernel-tools-libs-3.10.0-514.21.1.el7 \
        kernel-tools-libs-devel-3.10.0-514.21.1.el7
        ```

        Refer to the [Lustre Changelog](http://wiki.lustre.org/Category:Changelog) for the list of supported kernels.

    1. Install additional development packages. These are needed to enable support for some of the newer features in Lustre – if certain packages are not detected by Lustre's configure script when its DKMS package is installed, the features will not be enabled when Lustre is compiled. Notable in this list are `krb5-devel` and `libselinux-devel`, needed for Kerberos and SELinux support, respectively.

        ```bash
        yum install asciidoc audit-libs-devel automake bc \
        binutils-devel bison device-mapper-devel elfutils-devel \
        elfutils-libelf-devel expect flex gcc gcc-c++ git \
        glib2 glib2-devel hmaccalc keyutils-libs-devel krb5-devel ksh \
        libattr-devel libblkid-devel libselinux-devel libtool \
        libuuid-devel libyaml-devel lsscsi make ncurses-devel \
        net-snmp-devel net-tools newt-devel numactl-devel \
        parted patchutils pciutils-devel perl-ExtUtils-Embed \
        pesign python-devel redhat-rpm-config rpm-build systemd-devel \
        tcl tcl-devel tk tk-devel wget xmlto yum-utils zlib-devel
        ```

    1. Ensure that a persistent hostid has been generated on the machine. If necessary, generate a persistent hostid (needed to help protect zpools against simultaneous imports on multiple servers). For example:

        ```bash
        hid=`[ -f /etc/hostid ] && od -An -tx /etc/hostid|sed 's/ //g'`
        [ "$hid" = `hostid` ] || genhostid
        ```

    1. Reboot the node.

        ```bash
        reboot
        ```

    1. Install the packages for Lustre and ZFS:

        ```bash
        yum --nogpgcheck --enablerepo=lustre-server install \
        lustre-dkms \
        lustre-osd-zfs-mount \
        lustre \
        lustre-resource-agents \
        zfs
        ```

    1. Load the Lustre and ZFS kernel modules to verify that the software has installed correctly:

        ```bash
        modprobe -v zfs
        modprobe -v lustre
        ```

#### Start the Cluster Framework on Node 1

1. Login to node 1 and start the cluster framework as follows:

    ```bash
    pcs cluster start
    ```

1. Take node 1 out of standby mode:

    ```bash
    pcs cluster unstandby [<node 1>]
    ```

    **Note:** If the node name is omitted from the command, the currently logged in node will be removed from standby mode.

1. Verify that the node is active:

    ```bash
    pcs status
    ```

    Node 1 should now be in the online state.

#### Migrate the Lustre Services to Node 1

1. Verify that Pacemaker is able to identify node 1 as a valid target for hosting its preferred resources:

    ```bash
    pcs resource relocate show
    ```

    The command output should list the set of resources that would be relocated to node 1 according to the location constraints applied to each resource.

1. Nominate a single cluster resource and move it back to node 1 to validate the upgrade node configuration:

    ```bash
    pcs resource relocate run <resource name>
    ```

1. Verify that the resource has started successfully on node 1:

    ```bash
    pcs status

    df -ht lustre
    ```

1. Once the first resource has been successfully started on node 1, migrate all of the remaining resources away from node 2. The easiest way to do this is to set the node 2 to standby mode, using the following command:

    ```bash
    pcs cluster standby <node 2>
    ```

    **Note:** if the node name is omitted from the command, the node currently logged into will be affected. To be completely safe, log into node 2 to run the command. When specifying the node, ensure that the node name exactly matches the name used in the Pacemaker configuration. If Pacemaker uses the FQDN of a host, then the FQDN must be used in the `pcs` commands.

1. Verify that the resources have moved to node 1 and that the resources are running (all storage targets are mounted on the second node only):

    ```bash
    # On node 1:
    pcs resource show
    df -ht lustre
    ```

1. Disable node 2 in the cluster framework:

    ```bash
    pcs cluster disable <node 2>
    ```

Node 1 upgrade is complete.

#### [If Needed] Upgrade the OS on Node 2

1. Login to node 2.
1. If upgrading from e.g. EL 7.3 to EL 7.4, run the OS upgrade first. For example:

    ```bash
    yum clean all
    yum update
    reboot
    ```

    Refer to the operating system documentation for details on the correct procedure for upgrading between minor OS releases.

#### Upgrade the Manager for Lustre Agent on Node 2

1. Login to node 2.
1. Install EPEL repository support:

    ```bash
    yum -y install epel-release
    ```

1. Install the Manager for Lustre COPR Repository definition, which contains some dependencies for the IML Agent:

    ```bash
    yum-config-manager --add-repo \
    https://copr.fedorainfracloud.org/coprs/managerforlustre/manager-for-lustre/repo/epel-7/managerforlustre-manager-for-lustre-epel-7.repo
    ```

1. Install the DNF project COPR Repository definition. DNF is a package manager, and is used as a replacement for YUM in many distributions, such as Fedora. It does not replace YUM in CentOS, but Manager for Lustre does make use of some of the newer features DNF for some of its tasks:

    ```bash
    yum-config-manager --add-repo \
    https://copr.fedorainfracloud.org/coprs/ngompa/dnf-el7/repo/epel-7/ngompa-dnf-el7-epel-7.repo
    ```

1. Remove or rename the old IML YUM repository definition:

    ```bash
    mv /etc/yum.repos.d/Intel-Lustre-Agent.repo \
    $HOME/Intel-Lustre-Agent.repo.bak
    ```

1. Install the IML Agent repository definition:

    ```bash
    curl -o /etc/yum.repos.d/Manager-for-Lustre.repo \
    --cacert /var/lib/chroma/authority.crt \
    --cert /var/lib/chroma/self.crt \
    --key /var/lib/chroma/private.pem \
    https://<admin server>/repo/Manager-for-Lustre.repo
    ```

    Replace `<admin server>` in the `https` URL with the appropriate manager for lustre hostname (normally the fully-qualified domain name).

1. Upgrade the IML Agent and Diagnostics packages

    ```bash
    yum -y --enablerepo=iml-agent install chroma-\*
    ```

    **Note:** following warning during install / update is benign and can be ignored:

    ```bash
    ...
      /var/tmp/rpm-tmp.MV5LGi: line 5: /selinux/enforce: No such file or directory
      []

      {
        "result": null,
        "success": true
      }
    ...
    ```

#### Upgrade the Lustre Server Software on Node 2

1. Stop ZED and unload the ZFS and SPL modules:

    ```bash
    systemctl stop zed
    rmmod zfs zcommon znvpair spl
    ```

1. Remove the Lustre, ZFS and SPL packages, as they will be replaced with upgraded packages. The upgrade process will fail if these packages are not preemptively removed.

    ```bash
    yum erase zfs-dkms spl-dkms \
    lustre \
    lustre-modules \
    lustre-osd-ldiskfs \
    lustre-osd-zfs \
    lustre-osd-ldiskfs-mount \
    lustre-osd-zfs-mount \
    libzpool2 \
    libzfs2
    ```

1. Clean up the YUM cache to remove any residual information on old repositories and package versions:

    ```bash
    yum clean all
    ```

1. For systems that have **both** LDISKFS and ZFS OSDs:

    **Note:** This is the configuration that IML installs for all managed-mode Lustre storage clusters. Use this configuration for the highest level of compatibility with Manager for Lustre.

    1. Install the Lustre `e2fsprogs` distribution:

        ```bash
        yum --nogpgcheck --disablerepo=* --enablerepo=e2fsprogs-wc \
        install e2fsprogs e2fsprogs-devel
        ```

    1. If the installed `kernel-tools` and  `kernel-tools-libs` packages are at a higher revision than the patched kernel packages in the Lustre server repository, they will need to be removed:

        ```bash
        yum erase kernel-tools kernel-tools-libs
        ```

    1. Install the Lustre-patched kernel packages. Ensure that the Lustre repository is picked for the kernel packages, by disabling the OS repos:

        ```bash
        yum --nogpgcheck --disablerepo=base,extras,updates \
        --enablerepo=lustre-server install \
        kernel \
        kernel-devel \
        kernel-headers \
        kernel-tools \
        kernel-tools-libs \
        kernel-tools-libs-devel
        ```

    1. Install additional development packages. These are needed to enable support for some of the newer features in Lustre – if certain packages are not detected by Lustre's configure script when its DKMS package is installed, the features will not be enabled when Lustre is compiled. Notable in this list are `krb5-devel` and `libselinux-devel`, needed for Kerberos and SELinux support, respectively.

        ```bash
        yum install asciidoc audit-libs-devel automake bc \
        binutils-devel bison device-mapper-devel elfutils-devel \
        elfutils-libelf-devel expect flex gcc gcc-c++ git glib2 \
        glib2-devel hmaccalc keyutils-libs-devel krb5-devel ksh \
        libattr-devel libblkid-devel libselinux-devel libtool \
        libuuid-devel libyaml-devel lsscsi make ncurses-devel \
        net-snmp-devel net-tools newt-devel numactl-devel \
        parted patchutils pciutils-devel perl-ExtUtils-Embed \
        pesign python-devel redhat-rpm-config rpm-build systemd-devel \
        tcl tcl-devel tk tk-devel wget xmlto yum-utils zlib-devel
        ```

    1. Ensure that a persistent hostid has been generated on the machine. If necessary, generate a persistent hostid (needed to help protect zpools against simultaneous imports on multiple servers). For example:

        ```bash
        hid=`[ -f /etc/hostid ] && od -An -tx /etc/hostid|sed 's/ //g'`
        [ "$hid" = `hostid` ] || genhostid
        ```

    1. Reboot the node.

        ```bash
        reboot
        ```

    1. Install Lustre, and the LDISKFS and ZFS `kmod` packages:

        ```bash
        yum --nogpgcheck --enablerepo=lustre-server install \
        kmod-lustre-osd-ldiskfs \
        lustre-dkms \
        lustre-osd-ldiskfs-mount \
        lustre-osd-zfs-mount \
        lustre \
        lustre-resource-agents \
        zfs
        ```

    1. Verify that the DKMS kernel modules for Lustre, SPL and ZFS have installed correctly:

        ```bash
        dkms status
        ```

        All packages should have the status `installed`. For example:

        ```bash
        # dkms status
        lustre, 2.10.1, 3.10.0-693.2.2.el7_lustre.x86_64, x86_64: installed
        spl, 0.7.1, 3.10.0-693.2.2.el7_lustre.x86_64, x86_64: installed
        zfs, 0.7.1, 3.10.0-693.2.2.el7_lustre.x86_64, x86_64: installed
        ```

    1. Load the Lustre and ZFS kernel modules to verify that the software has installed correctly:

        ```bash
        modprobe -v zfs
        modprobe -v lustre
        ```

1. For systems that use LDISKFS OSDs:
    1. Upgrade the Lustre `e2fsprogs` distribution:

        ```bash
        yum --nogpgcheck \
        --disablerepo=* --enablerepo=e2fsprogs-wc install e2fsprogs
        ```

    1. If the installed `kernel-tools` and  `kernel-tools-libs` packages are at a higher revision than the patched kernel packages in the Lustre server repository, they will need to be removed:

        ```bash
        yum erase kernel-tools kernel-tools-libs
        ```

    1. Install the Lustre-patched kernel packages. Ensure that the Lustre repository is picked for the kernel packages, by disabling the OS repos:

        ```bash
        yum --nogpgcheck --disablerepo=base,extras,updates \
        --enablerepo=lustre-serverinstall \
        kernel \
        kernel-devel \
        kernel-headers \
        kernel-tools \
        kernel-tools-libs \
        kernel-tools-libs-devel
        ```

    1. Reboot the node.

        ```bash
        reboot
        ```

    1. Install the LDISKFS `kmod` and other Lustre packages:

        ```bash
        yum --nogpgcheck --enablerepo=lustre-server install \
        kmod-lustre \
        kmod-lustre-osd-ldiskfs \
        lustre-osd-ldiskfs-mount \
        lustre \
        lustre-resource-agents
        ```

    1. Load the Lustre kernel modules to verify that the software has installed correctly:

        ```bash
        modprobe -v lustre
        ```

1. For systems with ZFS-based OSDs:
    1. Follow the instructions from the ZFS on Linux project to [install the ZFS YUM repository definition](https://github.com/zfsonlinux/zfs/wiki/RHEL-%26-CentOS). Use the DKMS package repository (the default).

        For example:

        ```bash
        yum -y install \
        http://download.zfsonlinux.org/epel/zfs-release.el7_4.noarch.rpm
        ```

    1. Clean the YUM cache:

        ```bash
        yum clean all
        ```

    1. Install the kernel packages that match the latest supported version for the Lustre release:

        ```bash
        yum install \
        kernel \
        kernel-devel \
        kernel-headers \
        kernel-tools \
        kernel-tools-libs \
        kernel-tools-libs-devel
        ```

        It may be necessary to specify the kernel package version number in order to ensure that a kernel that is compatible with Lustre is installed. For example, Lustre 2.10.0 has support for RHEL kernel 3.10.0-514.21.1.el7:

        ```bash
        yum install \
        kernel-3.10.0-514.21.1.el7 \
        kernel-devel-3.10.0-514.21.1.el7 \
        kernel-headers-3.10.0-514.21.1.el7 \
        kernel-tools-3.10.0-514.21.1.el7 \
        kernel-tools-libs-3.10.0-514.21.1.el7 \
        kernel-tools-libs-devel-3.10.0-514.21.1.el7
        ```

        Refer to the [Lustre Changelog](http://wiki.lustre.org/Category:Changelog) for the list of supported kernels.

    1. Install additional development packages. These are needed to enable support for some of the newer features in Lustre – if certain packages are not detected by Lustre's configure script when its DKMS package is installed, the features will not be enabled when Lustre is compiled. Notable in this list are `krb5-devel` and `libselinux-devel`, needed for Kerberos and SELinux support, respectively.

        ```bash
        yum install asciidoc audit-libs-devel automake bc \
        binutils-devel bison device-mapper-devel elfutils-devel \
        elfutils-libelf-devel expect flex gcc gcc-c++ git \
        glib2 glib2-devel hmaccalc keyutils-libs-devel krb5-devel ksh \
        libattr-devel libblkid-devel libselinux-devel libtool \
        libuuid-devel libyaml-devel lsscsi make ncurses-devel \
        net-snmp-devel net-tools newt-devel numactl-devel \
        parted patchutils pciutils-devel perl-ExtUtils-Embed \
        pesign python-devel redhat-rpm-config rpm-build systemd-devel \
        tcl tcl-devel tk tk-devel wget xmlto yum-utils zlib-devel
        ```

    1. Ensure that a persistent hostid has been generated on the machine. If necessary, generate a persistent hostid (needed to help protect zpools against simultaneous imports on multiple servers). For example:

        ```bash
        hid=`[ -f /etc/hostid ] && od -An -tx /etc/hostid|sed 's/ //g'`
        [ "$hid" = `hostid` ] || genhostid
        ```

    1. Reboot the node.

        ```bash
        reboot
        ```

    1. Install the packages for Lustre and ZFS:

        ```bash
        yum --nogpgcheck --enablerepo=lustre-server install \
        lustre-dkms \
        lustre-osd-zfs-mount \
        lustre \
        lustre-resource-agents \
        zfs
        ```

    1. Load the Lustre and ZFS kernel modules to verify that the software has installed correctly:

        ```bash
        modprobe -v zfs
        modprobe -v lustre
        ```

#### Start the Cluster Framework on Node 2

1. Login to node 2 and start the cluster framework as follows:

    ```bash
    pcs cluster start
    ```

1. Take node 2 out of standby mode:

    ```bash
    pcs cluster unstandby [<node 2>]
    ```

    **Note:** If the node name is omitted from the command, the currently logged in node will be removed from standby mode.

1. Verify that the node is active:

    ```bash
    pcs status
    ```

    Node 2 should now be in the online state.

### Rebalance the Distribution of Lustre Resources Across all Cluster Nodes

1. Verify that Pacemaker is able to identify node 2 as a valid target for hosting its preferred resources:

    ```bash
    pcs resource relocate show
    ```

    The command output should list the set of resources that would be relocated to node 2 according to the location constraints applied to each resource.

1. Nominate a single cluster resource and move it back to node 2 to validate the upgrade node configuration:

    ```bash
    pcs resource relocate run <resource name>
    ```

1. Verify that the resource has started successfully on node 2:

    ```bash
    pcs status

    df -ht lustre
    ```

1. Relocate the remaining cluster resources to their preferred nodes:

    ```bash
    pcs resource relocate run
    ```

    **Note:** When the `relocate run` command is invoked without any resources in the argument list, all of the resources are evaluated and moved to their preferred nodes.

Node 2 upgrade is complete

Repeat this procedure on all Lustre server HA cluster pairs for the file system.