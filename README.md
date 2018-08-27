
# Disco Deployment & Administration

"Disco" is the name of the automated media ingestion system at [RXMusic (PCMusic)](http://www.rxmusic.com).

Disco consists of a cluster of Ubuntu servers built ontop of the **Python-Erlang MapReduce** [Disco Project](http://discoproject.org/). The music streaming industry currently consists of a growing catalog of over 24 milltion unique titles. On average, Disco processed and ingested one new title every 4 seconds.

Deploying new slave hosts to the **Disco cluster** and all cluster wide adminstration is accomplished using [Ansible](https://docs.ansible.com/) and a few custom scripts built on-top of [parallel-ssh](http://manpages.ubuntu.com/manpages/xenial/man1/parallel-ssh.1.html) and [parallel-rsync](http://manpages.ubuntu.com/manpages/bionic/man1/parallel-rsync.1.html).

#### Host Terminology
* **localhost** - your development machine
* **master** - the **Disco Master** host *pcm-ub10m10*
* **slaves** - the active *slaves* already in the Disco cluster
* **slaves_new** - the new slave hosts to setup and add to the Disco cluster

The **localhost**, **master**, and **slaves** should be mostly identical:
* All will be **Ubuntu 14.04 LTS** systems
* All will have matching users local users \@disco and \@pcmit
    * The *home* folders will be mostly synced in respect to all **production** related folders and files
* Since *development* is done on **localhost**, this host will contain additional development and build tools that **master** does not require.
* When new and updated files are ready for **production deployment**, these production files are rsynced to from the **localhost** to all the **master** and **slaves** hosts.
    * The **master** host will also contain additional deployment and adminstration files that are not required for **slaves**. Again, these files be rsynced from **localhost**.

More details will about production deployment will be provided in the section [TBD]().

#### User IDs
The local user \@disco must have user-id **1002** on all Disco hosts. When configuring `fstab` mount details, permissions will be configured with `gid=1002`.

All **PCMEDIA** domain network access from the Disco hosts are done using the credentials for **PCMEDIA\autoproc1**. Only local user \@disco

## Deploying New Slave Hosts

These instructions assume the **localhost** and **master** are already setup. We will be rsyncing folders and files from **master** to the **slaves_new** hosts.

### Pre-setup of Ansible Project Files

Please ensure the **localhost** contains the latest **Disco ansible project** files. Do a git pull or clone as necessary.

To clone the project:
```
disco@locahost:~/ansible/$>
$>git clone https://your-username@bitbucket.org/pcmtech/disco-deployment.git disco
```

To pull the latest project files:
```
disco@localhost:~/ansible/disco$>
$>git pull origin master
```

The host **master** must also contain the latest ansible project files. Once the project files are up-to-date on **localhost**, you should be able to do this to update **master**:
```
disco@locahost:~/ansible/disco$>
$>pcmsync_hosts master .
```

### Ubuntu 14.04 LTS Installation
During Ubuntu 14.04 LTS install on **slaves_new** hosts:
1. Create the sudo admin user as: **pcmit**

#### Install Packages
1.OpenSSH

Once Ubuntu has been install, we will be using **ansible** to complete the rest of the host setup.

To help explain the instructions, we will assume we are adding the following two new slaves to the Disco cluster:
* pcm-ub10n13
* pcm-ub10n14

### Prepare Inventory Files

#### Update hosts
Edit the **production** inventory `hosts` file:
```
disco@localhost:~/ansible/disco$>
$>cd inventories/production/
$>vi hosts
```

Add new slave hostnames to the group `[slaves_new]`:
```
[slaves_new]
pcm-ub10n13
pcm-ub10n14
```

The `hosts` file should now look similar to this (from the top):
```
[master]
pcm-ub10m10

[slaves]
pcm-ub10n11
pcm-ub10n12
pcm-ub10n13
pcm-ub10n14

[master_slaves:children]
master
slaves

[slaves_new]
pcm-ub10n13
pcm-ub10n14

[slaves_all:children]
slaves
slaves_new
```

#### Add hosts to hosts_vars
Create a **hosts_vars** subfolder for each **slaves_new** host:
```
$>mkdir hosts_vars/pcm-ub10n13
$>touch hosts_vars/pcm-ub10n13/defaults.yml
$>mkdir hosts_vars/pcm-ub10n14
$>touch hosts_vars/pcm-ub10n14/defaults.yml
```
Add variables for each new host to their respective `defaults.yml` file. 

This following variables are required:
```
# The max number of disco workers allowed for the host.
# For a host doing media processing:
#
#   disco_max_workers = number of CPUs
#
# You can do a lscpu on the host to get its CPU info.
#
disco_max_workers: 40 
```
Use an already existing **slaves**' `defaults.yml` for a complete reference of what other variables are required.

### Ansible Site Playbook

The ansible site playbook is file ``site.yml``. 

All plays are executed by calling the site playbook with the desired *tags* and any required user-password switches.

Plays that require root access (sudo) must be executed by \@pcmit.

All other plays will involve \@disco's home folders and files. These plays must be executed by \@disco.

In the instructions that follow, please also pay attention to the host and user you are running the plays from. Some plays need to be run from **master**. Some plays need to be run from **localhost**.

Make sure you know the passwords for \@disco and \@pcmit.

#### Tips:  
* The `-k` switch will ask you for the `remote_user` password
    * Once passwordless SSH logins is created for \@pcmit and \@disco, the `remote_user` password will no longer be required
* The `--ask-become-pass` switch will prompt you for the `sudo` user password. The `sudo` user is \@pcmit.
    * All commands that require \@pcmit as the `remote_user` will require this switch.
* If you run into errors executing these commands, use switch `-vvv` to see all verbose details.

#### slaves_new:init_host:pcmit
```
pcmit@localhost:~/ansible/disco$>
$>ansible-playbook site.yml -t slaves_new:init_host:pcmit -k --ask-become-pass
```
Note the `-k` and `--ask-become-pass` switches. This command will prompt you twice for \@pcmit's password:
* Once for the `remote_user` 
* Once for the `sudo` privileges

When this completes, the **slaves_new** hosts will reboot. 

Passwordless SSH login is now enabled for \@pcmit on all **slaves_new** hosts.

#### slaves_new:init_host:disco
After reboot, we will run the init plays associated with user \@disco.

```
disco@localhost:~/ansiable/disco$>
$>ansible-playbook site.yml -t slaves_new:init_host:disco -k
```
This command will prompt you for \@disco's password.

When it completes, passwordless SSH login will be enabled for \@disco on all **slaves_new** hosts.

### Sync User Home

We will now rsync the home folders of \@pcmit and \@disco from the **master** host machine to  **slaves_new** hosts.

#### slaves_new:sync_user:pcmit
```
pcmit@master:~/ansible/disco$>
$>ansiable-playbook site.yml -t slaves_new:sync_user:pcmit --ask-become-pass
```
#### slaves_new:sync_user:disco
```
disco@master:~/ansible/disco$>
$>ansiable-playbook site.yml -t slaves_new:sync_user:disco
```
This command will take some time to complete.
### Setup fstab
We will now setup **/etc/fstab** on **slaves_new** hosts. This will add mounting instructions to allow the new slave hosts to access the required network shares.

#### slaves_new:sync_fstab:pcmit
```
pcmit@master:~/ansible/disco$>
$>ansible-playbook site.yml -t slaves_new:sync_fstab:pcmit
```

## Mount Media
We can now mount all required network shares on **slaves_new** hosts. Network shares are mounted to `/mnt/pcmedia`.

#### slaves_new:mount_media:pcmit
```
$>ansible-playbook site.yml -t slaves_new:mount_media:pcmit --ask-become-pass
```

## Disco Media Folder Links

We can now create the media folders for user \@disco at `~/media`. The subfolders will be links to the mounted folders at `/mnt/pcmedia`.

#### slaves_new:sync_user:disco:media
```
disco@master:~/ansible/disco$>
$>ansible-playbook site.yml -t slaves_new:sync_media:disco
```

### Quick Verification
As a quick verification, login into one of the **slaves_new** host.
```
disco@localhost:~$>ssh pcm-ub10n14
...
disco@pcm-ub10n14:~$>
```

Check \@disco's media folder:
```
$>ls media/pcmedia/nx3500/PCMServer
HRMedia      Modules      WebArtFiles   zIssueFiles
Art          LowResMedia  nginx         UI       WebImages    Backups      HiResAlbumArt Media        Stations     WebApps      WebMedia
```
Feel free to check the other subfolders.

Check that **python** works:
```
$>python
Python 2.7.15 |Anaconda 2.5.0 (64-bit)| (default, May  1 2018, 23:32:55) 
[GCC 7.2.0] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import numpy
>>> import PCMusic, PCMusicApps, pcmemdia
>>>
```
If you did not encounter any errors running any of these ansible commands and if don't see any errors during this spotcheck, then the deployment has very likely succeeded for all the **slaves_new** hosts. 

## Add New Slaves to Disco Cluster
After successful deployment, we can now add the **slaves_new** to the **Disco cluster**.

#### Update hosts
Edit the **production** inventory `hosts` file by moving the hostnames under **[slaves_new]** to **[slaves]**:
```
disco@localhost:~/ansible/disco$>
$>cd inventories/production/
$>vi hosts
```
Using are example hosts, the `hosts` file will look like this
```
[master]
pcm-ub10m10

[slaves]
pcm-ub10n11
pcm-ub10n12
pcm-ub10n13
pcm-ub10n14

[master_slaves:children]
master
slaves

[slaves_new]

[slaves_all:children]
slaves
slaves_new

```
Now deploy the updated `hosts` file to the **master** host:
```
disco@localhost:~/ansible/disco$>
$>pcmsync_hosts master .
```

### Update Disco Config

Now we update the Disco configuration files on **master** with the new hosts.
```
disco@localhost:~/ansible/disco$>
$>ansible-playbook site.yml -t master:admin_disco:config
```
#### Verification
Browse to the [Disco Dashboard](http://pcm-ub10m10:8989/index.html).


You should see blocks for the newly added hosts. It may take a few seconds for Disco to display *max worker* and *available storage space* details for each newly added hosts.

Congrations! The newly added hosts are now apart of the Disco cluster.

#### Localhost Config
We also want to update the `parellel-ssh` and `parallel-rsync` hosts config files on **localhost** so that we can use these tools to perform tasks on the Disco cluster.
```
disco@localhost:~/ansible/disco$>
$>ansible-playbook site.yml -t localhost:admin_disco:config 
```
The following config files will have been updated from:
* `~/.disco_hosts_slaves`
* `~/.disco_hosts_master_slaves`

To verify
```
$>cat ~/.disco_hosts_slaves
pcm-ub10n11
pcm-ub10n12
pcm-ub10n13
pcm-ub10n14
$>cat ~/.disco_hosts_master_slaves
pcm-ub10m10
pcm-ub10n11
pcm-ub10n12
pcm-ub10n13
pcm-ub10n14
```

### Update Master Application Config

Recall that there are **Disco master applications** for the following kinds of jobs:
* **Daily Media Processing and Ingestion**
    * v4_audio
        * Master responsible for processing audio titles
    * v4_video
        * Master responsible for processing video titles

* **Daily Feed Shelf-Choreography**
    * v4_shelf_choreo
        * Master responsible for processing all titles

* **Daily Media Features Extraction**
    * v4u_audio
        * Master responsible for processing audio titles
    * v4u_video
        * Master responsible for processing video titles

You will need to update the corresponding Python configuration module for the **Disco masters applications** that can now send jobs to any one of the new **slaves**.

For example, say the requirement is as follows:
* `pcm-ub10n13` will be added to **v4_audio** slaves
    * currently **v4_audio** slaves includes only `pcm-ub10n11`
* `pcm-ub10n14` will be added to **v4_video** slaves
    * currently **v4_video** slaves includes only `pcm-ub10n12`
  
You will need to update the Python module: `PCMusicApps.Client.MediaProcessing.Disco.MasterLabelRepos`

Under the configuration values for key: `NS.disco_worker_hosts`

You may meed to update the `forceon_pat` value to make sure it will match the all required hosts.

So in our example, the following settings will work:
```
NS.disco_worker_hosts : {
    'v4_audio': {
        'forceon_path': 'pcm-ub10n1[1,3]+$',
        ...
    },
    'v4_video': {
        'forceon_path' 'pcm-ub10n1[2,4]+$',
        ..
    }
}
```
We see that **v4_audio** jobs will be delegated to the slave hosts `pcm-ub10n11` and `pcm-ub10n13`.


We see that **v4_video** jobs will be delegated to the slave hosts `pcm-ub10n11` and `pcm-ub10n13`.

