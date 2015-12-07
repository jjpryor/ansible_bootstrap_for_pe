# ansible_boot_strap_for_pe
Ansible playbooks for installing Puppet Enterprise 2015.2.3 on a three node
split install. Assumes RHEL7 on target nodes, Fedora 22 for desktop machine, and
Ansible 1.9+.

The reference material from which these playbooks are composed is the 
["Puppet Enterprise 2015.2 Docs"](http://docs.puppetlabs.com/pe/2015.2/install_system_requirements.html)



# Start here with remote account pre-reqs
git clone this repo to the a working area in your `$HOME`.

On the three nodes, perform the following steps to enable the remote user and
the remote user's public SSH key placed into the `.ssh/authorized_keys` file.

SSH into the target node:
 ```shell
 ssh root@targetnode
 ```

Now as root, add the nonprivuser:
```shell
useradd nonprivuser -u SomeLargeNumberToPreventUidCollision
```

Add `nonprivuser` to /etc/group `wheel`: `usermod -G wheel -a nonprivuser`

Change the `sudoers` file to have the wheel group able to run sudo commands
without requiring a password. *This is obviously weakens security, and should
be removed later*
```
%wheel  ALL=(ALL)       NOPASSWD: ALL
```

Now become the `nonprivuser`:
```shell
su - nonprivuser
```

As nonprivuser, make the .ssh directory, set the correct permissions, and
copy/paste the `authorized_keys` file into place, then exit the `nonprivuser`:
```shell
cd
mkdir .ssh
chmod 600 .ssh
vim .ssh/authorized_keys
<paste the public key>
chmod 600 .ssh/authorized_keys
exit
```

From your desktop, enable `ssh-agent` if your desktop environment doesn't
automatically have it running, and also add your ssh identity to the agent.

(*Quite possibly the wrong way to do this, but it works*)
```shell
exec ssh-agent bash
ssh-add ~/.ssh/your_private_key
```
Then check if ssh nonprivuser@targetnode via key & passphrase works and also
test if password-less sudo works:
```shell
ssh -i ~/.ssh/your_private_key nonprivuser@targetnode
sudo touch /etc/foobar
```

Now test ansible to see if it will work:
```
ansible -m setup pe-node.yourorg.local
```
It should connect via SSH and gather facts of the target
and display the facts in the console, something lke:
```
pe-node.yourorg.local | success >> {
    "ansible_facts": {
        "ansible_all_ipv4_addresses": [
            "111.222.323.423
        ],
        "ansible_all_ipv6_addresses": [
            "fe80::250:6bff:fe89:689f"
        ],
        "ansible_architecture": "x86_64",
        "ansible_bios_date": "04/14/2014",
        "ansible_bios_version": "6.00",
        "ansible_cmdline": {
            "BOOT_IMAGE": "/vmlinuz-3.10.0-229.el7.x86_64",
            "crashkernel": "auto",
            "nofb": true,
            "rd.lvm.lv": "sysvg/root",
            "ro": true,
            "root": "/dev/mapper/sysvg-root"
        },
```

Just about ready to run the ansible playbooks, but we need to check
if there is not enough space in the filesystem, then use `lvresize` to resize
logical volume of the partition.
Examples:
```shell
lvresize -r -L 110G /dev/mapper/optvg-opt
lvresize -r -L 45G /dev/mapper/varvg-var
lvresize -r -L 5G /dev/mapper/sysvg-tmp
```
## Split-node PE hardware & disk space requirements
For a three-node split install, we will be following the recommended specs per
the Pupppet Labs docs:
[http://docs.puppetlabs.com/pe/2015.2/install_system_requirements.html#medium-environment]


| Node | Cores | RAM | /opt/ | /var/ | EC2 | /tmp/ |
| --- | --- |  --- |  --- |  --- | --- |  --- |
|Puppet master|4|16 GB|10 GB|42 GB|m3.xlarge or m4.xlarge instance| 1G free|
|PE console|2|6 GB|10 GB|22 GB|m3.large or m4.large instance| 1G free|
|PuppetDB|4|16 GB|100 GB|42 GB|m3.xlarge or m4.xlarge instance| 1G free |

## Single node PE hardware & disk space requirements
For a single-node install, we will be following the recommended specs per
the Pupppet Labs docs:
[http://docs.puppetlabs.com/pe/2015.2/install_system_requirements.html#small-environment]


| Node | Cores | RAM | /opt/ | /var/ | EC2 | /tmp/ |
| --- | --- |  --- |  --- |  --- | --- |  --- |
|Small Monolithic Node|4|16 GB|100 GB|42 GB|m3.xlarge or m4.xlarge instance| 1G free|

# Execute the ansible playbooks
If it is all successful, then it is time to start using the ansible playbooks.
They are designed to get all the pieces in place and check to see if the disk
space requirements are met.

## Split Node Puppet Enterprise setup
Go to the `split-node-pe` directory and run the following playbooks.

```Shell
ansible-playbook 00-wget-pe-tarball-and-gpg-sig.yml
ansible-playbook 01-preqs-all-pe-nodes.yml
ansible-playbook 02-preqs-console.yml
ansible-playbook 03-preqs-master.yml
ansible-playbook 04-preqs-pdb.yml
ansible-playbook 05-bootstrap-pe-on-master.yml
```

Then on the master node, as root, extract the tarball & run the PE installer:
```shell
cd ~/pe/
tar zxf puppet-enterprise-2015.2.3-el-7-x86_64.tar.gz
cd puppet-enterprise-2015.2.3-el-7-x86_64
./puppet-enterprise-installer
```

After the install is successful, then run the last playbook:
```shell
ansible-playbook 06-postinstall-master.yml
```
---
## Single Node Puppet Enterprise setup
Go to the `single-node-pe` directory and run the following playbooks.

```shell
ansible-playbook 00-wget-pe-tarball-and-gpg-sig.yml
ansible-playbook 01-prereqs-single-master.yml
ansible-playbook 02-bootstrap-pe-on-single-master.yml
```

Then on the master node, as root, extract the tarball & run the PE installer:
```shell
cd ~/pe/
tar zxf puppet-enterprise-2015.2.3-el-7-x86_64.tar.gz
cd puppet-enterprise-2015.2.3-el-7-x86_64
./puppet-enterprise-installer
```

After the install is successful, then run the last playbook:
```shell
ansible-playbook 03-postinstall-master.yml
```

---

## Contents of split-node-pe directory
### ansible.cfg
I'm pointing ansible to use my SSH private key and I'm using my `jjpryortemp` remote account that must be manually setup on each of the three nodes.

### ansible_hosts
It should be obvious that this is where you define the hosts that these
playbooks will be interacting with. Again this is a three-node split PE
install.
```
[console]
consolenode.yourorg.local

[master]
masternode.yourorg.local

[pdb]
puppetdbnode.yourorg.local

[pe-nodes]
masternode.yourorg.local
consolenode.yourorg.local
puppetdbnode.yourorg.local
```
### 00-wget-pe-tarball-and-gpg-sig.yml
This playbook runs on the local machine. In my case it is my Fedora desktop and
the purpose is to download the Puppet Enterprise 2015.2.3 tarball for RHEL7 and
its accompanying GPG signature file, and then place these into the `./files`
directory.

### 01-preqs-all-pe-nodes.yml
Across the three nodes:
+ Performs `yum update -y`
+ Enables `$proxy_*tp` bash shell environment variables
+ Disables & stops `firewalld`
+ Enables & starts `iptables`

### 02-preqs-console.yml
+ Checks `/opt` & `/var` & `/tmp` for minimum size for Console.
+ Copies iptables for a Console node and restarts iptables

### 03-preqs-master.yml
+ Checks `/opt` & `/var` & `/tmp` for minimum size for Master.
+ Copies iptables for a *pre-PE install* Master node and restarts iptables

### 04-preqs-pdb.yml
+ Checks `/opt` & `/var` & `/tmp` for minimum size for PuppetDB.
+ Copies iptables for a PuppetDB node and restarts iptables

### 05-bootstrap-pe-on-master.yml
+ Create `/root/pe` directory, copy PE tarball & gpg signature to `pe` directory
+ Add GPG key to keyring and verify the signature.

### 06-postinstall-master.yml
+ Copy PE License key file and verify License key
+ Copies iptables for a *post-PE install* Master node and restarts iptables

---
## Contents of single-node-pe directory
### ansible.cfg
I'm pointing ansible to use my SSH private key and I'm using my `jjpryortemp` remote account that must be manually setup on each of the three nodes.

### ansible_hosts
It should be obvious that this is where you define the hosts that these
playbooks will be interacting with. Again this is a three-node split PE
install.
```
[pe-node]
pe-node.yourorg.local
```

### 00-wget-pe-tarball-and-gpg-sig.yml
This playbook runs on the local machine. In my case it is my Fedora desktop and
the purpose is to download the Puppet Enterprise 2015.2.3 tarball for RHEL7 and
its accompanying GPG signature file, and then place these into the `files/`
directory.

### 01-prereqs-single-master.yml
+ Performs `yum update -y`
+ Enables `$proxy_*tp` bash shell environment variables
+ Disables & stops `firewalld`
+ Enables & starts `iptables`
+ Checks `/opt` & `/var` & `/tmp` for minimum size for PE.
+ Copies iptables config file and restarts iptables


### 02-bootstrap-pe-on-single-master.yml
+ Create `/root/pe` directory, copy PE tarball & gpg signature to `pe` directory
+ Add GPG key to keyring and verify the signature.

### 03-postinstall-master.yml
+ Copy PE License key file and verify License key
+ Copies iptables for a *post-PE install* PE Master and restarts iptables

-----
## Contents of files directory

### files/proxy.sh
Shell environment variables for the proxy & this goes into `/etc/profile.d`

### files/wgetrc
wget's config file & this goes into `/etc/`

### files/PuppetLabsRelease-gpg.key
Puppet Labs Release GPG key from MIT keyserver

### files/iptables-console
iptables config for split installl PE Console node

### files/iptables-master-pre-install
iptables config for split installl PE Master node before the
puppet-enterprise-installer is finished

### files/iptables-pdb
iptables config for PE PuppetDB node

### files/iptables-master-post-install
iptables config for split install PE Master node after the
puppet-enterprise-installer is finished

### iptables-single-master-post-install
iptables config for single install PE Master node after the
puppet-enterprise-installer is finished

### iptables-single-master-pre-install
iptables config for single install PE Master node before the
puppet-enterprise-installer is finished

### files/license.key
License key file for PE for Your Org
