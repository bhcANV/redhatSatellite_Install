# RedHat Satellite Installation & Setup

## Table of Contents 
- [Important Notes](#notes)
- [Installation](#guide)
- [Connecting to Satellite](#connect)

<a id=notes></a>
## Important Notes: 
- Root access needed for pre-installation setup. Enter root via `sudo -s` after each login.
- Login to Azure VM with the following requirements:
    - OS: RedHat Enterprise Linux 8
    - 4 CPUs
    - 32 GB RAM
    - 1 TB Drive / Disk Space 
- This guide assumes installation from Satellite ISO disc image
- [RedHat Satellite Installation from ISO disc image](https://access.redhat.com/documentation/en-us/red_hat_satellite/6.0/html/installation_guide/sect-red_hat_satellite-installation_guide-installing_rednbsphat_satellite_with_an_iso_image#:~:text=Installing%20Red%20Hat%20Satellite%20with%20an%20ISO%20Image,installer%20script%20in%20the%20mounted%20directory%3A%20%23%20.%2Finstall_packages)
    - This is different from the [guide](#guide) below but may be more beneficial depending on your needs
- [General Information about Red Hat Satellite](https://access.redhat.com/products/red-hat-satellite/)
- [RedHat Satellite Technical Documentation](https://access.redhat.com/documentation/en-us/red_hat_satellite/6.13/html/satellite_overview_concepts_and_deployment_considerations/index)
- [RedHat Satellite Technical Overview Course (RH053)](https://www.youtube.com/playlist?list=PLbMP1JcGBmSEnmwbVGvtX-URDxmgOYOGd)

<a id=guide></a>
## Installation 

1. Mounting Resources
    1. After logging into the VM use `df -h` to check disk partitions and allocations
    2. The output should look something like this:
    ```
    Filesystem      Size  Used Avail Use% Mounted on
    devtmpfs         16G     0   16G   0% /dev
    tmpfs            16G     0   16G   0% /dev/shm
    tmpfs            16G   17M   16G   1% /run
    tmpfs            16G     0   16G   0% /sys/fs/cgroup
    /dev/sdb2        30G  1.5G   29G   5% /
    /dev/sda1       667G   32K  633G   1% /mnt/resource
    tmpfs           3.2G     0  3.2G   0% /run/user/1000
    ```
    We care about this line: `/dev/sda1       667G   32K  633G   1% /mnt/resource`. The filesystem name 
    `/dev/sda1` may be different in your system.

    3. In the command line enter the following commands:
    ```
    mkdir /var/lib/pulp
    umount /mnt/resource 
    mount /dev/sda1
    mount /dev/sda1 /var/lib/pulp
    ```

    4. Next enter in command line:
    ```
    systemctl daemon-reload
    systemctl list-units | grep pulp 
    ```
    Which should have the following output:
    ```
    var-lib-pulp.mount                                                                                                     
                                            loaded active mounted   /var/lib/pulp
    ```

    5. If no issues reboot and relogin: `systemctl reboot`

2. Edit waagent.conf
    1. Open waagent.conf : `vi /etc/waagent.conf`
    2. Find the following lines: 
    ```
    # Mount point for the resource disk
    ResourceDisk.MountPoint=/mnt/resource
    ```
    Replace `/mnt/resource` with `/var/lib/pulp`. Then save and close.

    3. Ensure that the change took place with grep: 
    ```
    cat /etc/waagent.conf | grep /var/lib/pulp
    ```

    4. If no issues reboot and relogin: `systemctl reboot`

3. Add Host and Update
    1. Ensure that firewalld and cockpit are not enabled. The following command should 
    not return any output. 
    ```
    systemctl list-units | (grep firewalld || grep cockpit)
    ```
    2. Go to the Azure Portal for this VM
    3. Get the Private IP Address. Ex xx.xxx.xxx.xx
    4. Get the DNS Name:
        1. If not configured, click on `Not configured`
        2. Enter a DNS name and save. Ex. `rhelSatVM1`
        3. Get full DNS Name. -> `rhelSatVM1.usgovvirginia.cloudapp.usgovcloudapi.net`
    5. Get the hostname via: `hostname`
    5. Add this DNS name and IP to hosts
    ```
    echo "xx.xxx.xxx.xx rhelSatVM1.usgovvirginia.cloudapp.usgovcloudapi.net your_hostname" >> /etc/hosts
    ```
    6. Check that the hosts have been updated with the new addition via `cat /etc/hosts`
    7. Update the VM
    ```
    dnf check-update
    dnf update -y
    ```
    8. If no issues reboot and relogin: `systemctl reboot`

4. Remove RHUI Packages & Register
    1. Remove RHUI packages:
    ```
    sudo yum --disablerepo='*' remove 'rhui-azure-rhel8'
    ```
    or search for packages to delete and enter manually in terminal 
    ```
    rpm -aq | grep rhui
    yum remove package_name
    ```
    and confirm deletion at prompt. 

    2. Using your RedHat User credentials register your subscription:
    ```
    subscription-manager register
    ```

5. Satellite Package Install
    1. Create a second mnt directory: `mkdir /mnt2`
    2. Run the following command: 
    ```
    mount -o loop,ro -t auto Satellite-6.13.3-rhel-8-x86_64.dvd.iso /mnt2
    ```
    3. Go into new mnt directory and attempt install
    ```
    cd /mnt2
    ./install_packages
    ```
    4. Attempt full install:
    ```
    satellite-installer --scenario satellite --foreman-proxy-dns-managed=false --foreman-proxy-dhcp-managed=false --foreman-initial-admin-username your_admin_username
    ```
    5. Store generated credentials somewhere for later


<a id=connect></a>
## Connecting to Satellite

1. Secondary VM
    1. In order to access the installed, running Satellite server through the web you need to 
    use a VM that can search the web. 
        - Note: Azure Windows Server 2019 Datacenter VM is good
    2. This VM needs to be be on the same Virtual Network and Network Security Group as VM where
    RedHat Satellite is installed.
        - Note: Ensure that the Network Security Group is configured to allow RDP connections.
    3. Connect to this VM via RDP

2. Open Microsoft Edge or browser on VM
3. Enter the private IP of the VM in the search bar.
```
hhtps://10.x.x.x 
```
4. Ignore security warnings
5. You should be routed to the RedHat Satellite Login portal.
    1. Sign in using the generated credentials from the install.
6. You are now logged in to your RedHat Satellite Server and can manage your organization!
7. [Get Started with Manifests](https://www.redhat.com/en/blog/how-create-and-use-red-hat-satellite-manifest)

