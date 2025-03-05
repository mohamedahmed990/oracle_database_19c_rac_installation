# oracle_database_19c_rac_installation

---

### **Table of Contents**

1. **Prepare Network Requirements**
2. **Operating System Configuration**
3. **Configure Users and Groups**
4. **Create Installation Directories**
5. **Configure Shared Disks Using Oracle ASM**
6. **Clone Node1 to Create Node2**
7. **Set Up `.bash_profile` for Grid and Oracle Users**
8. **Install Oracle Grid Infrastructure**
9. **Mounting ASM Disk Groups Using ASMCA**
10. **Install Oracle Database Software**
11. **Create a RAC Database**
12. **Verification and Testing**

---

## **1. Prepare Network Requirements**

### **1.1 Configure Network Connections**

**Public Network (enp0s3)**:

```bash
nmcli connection add type ethernet con-name "public" ifname enp0s3 \\
ipv4.addresses "192.168.56.105/24" \\
ipv4.gateway "192.168.56.1" \\
ipv4.dns "192.168.56.1" \\
ipv4.method "manual" \\
connection.autoconnect "yes"

```

**Private Network (enp0s8)**:

```bash
nmcli connection add type ethernet con-name "private" ifname enp0s8 \\
ipv4.addresses "192.168.10.1/24" \\
ipv4.method "manual" \\
connection.autoconnect "yes"

```

**Bridged Adapter (enp0s9):**

```bash
nmcli connection add type ethernet con-name "internet" ifname enp0s9 \\
ipv4.method "auto" \\
connection.autoconnect "yes"

```

### **1.2 List connections to verify creation**:

```bash
nmcli connection show

```

### **1.3 Manually activate any connection (optional)**:

```bash
nmcli connection up public
nmcli connection up private
nmcli connection up internet

```

### **1.4 Configure Hostnames and IPs**

Edit `/etc/hosts` :

```bash
# Public
192.168.56.105   node1.localdomain         node1
192.168.56.106   node2.localdomain         node2

# Private
192.168.10.1    node1-priv.localdomain    node1-priv
192.168.10.2    node2-priv.localdomain    node2-priv

# Virtual
192.168.56.81   node1-vip.localdomain     node1-vip
192.168.56.82   node2-vip.localdomain     node2-vip

# SCAN
192.168.56.91   node-scan.localdomain     node-scan

```

### **1.5 Verify Network Connectivity**

In the hosting PC, open a command prompt window and ping the node1 public IP address.

```bash
ping 192.168.56.105

```

## **2. Configure Users and Groups**

### **2.1 Create Required Groups**

Create the following groups :

```bash
groupadd asmadmin
groupadd asmdba
groupadd oinstall

```

### **2.2 Create Oracle Users**

Create the `oracle` and `grid` users :

```bash
useradd -u 54323 -g oinstall -G asmadmin,asmdba grid
usermod -a -g oinstall -G asmdba oracle

```

### **2.3 Set Passwords**

Set passwords for the `oracle` and `grid` users:

```bash
sudo passwd oracle
sudo passwd grid

```

---

## **3. Operating System Configuration**

### **3.1 Update the System**

```bash
sudo dnf update -y

```

### **3.2 Install Required Packages**

Install the Oracle preinstall package:

```bash
sudo dnf install -y oracle-database-preinstall-19c

```

### **3.3 Set Kernel Parameters**

Edit `/etc/sysctl.conf` and add:

```bash
kernel.sem = 250 32000 100 128
kernel.shmmni = 4096
kernel.shmall = 1073741824
kernel.shmmax = 4398046511104
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
fs.file-max = 6815744
net.ipv4.ip_local_port_range = 9000 65500

```

Apply the changes:

```bash
sudo sysctl -p

```

### **3.4 Set User Limits**

Edit `/etc/security/limits.conf` and add:

```bash
oracle   soft   nofile    1024
oracle   hard   nofile    65536
oracle   soft   nproc    16384
oracle   hard   nproc    16384
oracle   soft   stack    10240
oracle   hard   stack    32768
oracle   soft   memlock    3145728
oracle   hard   memlock    3145728
grid     soft   nofile    1024
grid     hard   nofile    65536
grid     soft   nproc    16384
grid     hard   nproc    16384
grid     soft   stack    10240
grid     hard   stack    32768
grid     soft   memlock    3145728
grid     hard   memlock    3145728

```

### **3.5 Disable the firewall**

```bash
systemctl stop firewalld
systemctl disable firewalld

```

### 3.6 Set `selinux` to permissive

```bash
cd /etc/selinux/
vi config

```

Edit the `config` file to be like the following :

```bash
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=permissive
# SELINUXTYPE= can take one of three values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted

```

---

## **4. Create Installation Directories**

Before installing Oracle Grid Infrastructure and Oracle Database, you must create the necessary directories . These directories will store the Oracle software binaries, configuration files, and inventory.



### **4.1 Create Directories for Oracle Database**

create the following directories for Oracle Database:

```bash
sudo mkdir -p /u01/app/oracle/product/19.0.0/dbhome_1

```

Set the correct ownership and permissions:

```bash
sudo chown -R oracle:oinstall /u01
sudo chmod -R 775 /u01

```
### **4.2 Create Directories for Grid Infrastructure**

create the following directories for Oracle Grid Infrastructure:

```bash
sudo mkdir -p /u01/app/19.0.0/grid
sudo mkdir -p /u01/app/grid

```

Set the correct ownership and permissions:

```bash
sudo chown -R grid:oinstall /u01/app/19.0.0/grid
sudo chown -R grid:oinstall /u01/app/grid
sudo chmod -R 775 /u01/app

```


---

## **. Configure Shared Disks Using Oracle ASM**

### 5.1 Create virtual disks steps:

1. **Shut Down Both Virtual Machines**
    - Ensure both `node1` and `node2` VMs are powered off.
2. **Create First Shared Disk (DISK1)**
    - Open **Oracle VirtualBox**.
    - Select the `node1` VM.
    - Click **Settings** → **Storage** → **SATA Controller** icon → **Add Hard Disk** button.
    - In the pop-up window, click **Create new disk**.
3. **Disk Configuration**
    - **Hard Disk File Type:** Select `VDI (VirtualBox Disk Image)`. Click **Next**.
    - **Storage on Physical Hard Disk:** Select **Fixed Size**. Click **Next**.
    - **File Location and Size:**
        - Click the folder icon and choose the parent folder of `node1`.
        - Name the disk `DISK1.vdi` and set the size to **10 GB**.
        - Click **Create**.
4. **Create Additional Disks (DISK2 and DISK3)**
    - Repeat the above steps to create `DISK2` and `DISK3`, each with **15 GB** size.
    - Ensure they are saved in the same parent folder as `DISK1`.
5. **Change Disks to Shareable**
    - In VirtualBox, go to **File** → **Virtual Media Manager (Ctrl+D)**.
    - Select `DISK1`, change its type to **Shareable**, and click **Apply**.
    - Repeat for `DISK2` and `DISK3`.
    - Close the **Virtual Media Manager** window.
6. **Attach Shared Disks to `node2`**
    - In Oracle VirtualBox, select the `node2` VM.
    - Click **Settings** → **Storage**.
    - Under **SATA Controller**, click the **Add Hard Disk** icon.
    - Click **Choose existing disk** and select the shared disk (`DISK1`).
    - Repeat for `DISK2` and `DISK3`.
    - Click **OK** to close the storage settings.

---

### **5.2 Starting `node1` and Listing Shared Disks**

1. **Start `node1`**
    - Power on `node1`.
2. **List Attached Disks**
    
    Run the following command to verify the attached shared disks:
    
    ```bash
    ls /dev/sd*
    ```
    
    You should see `sdb`, `sdc`, and `sdd` listed.
    

---

### **5.3 Partitioning Shared Disks with `fdisk`**

1. **Partition `DISK1` (`/dev/sdb`)**
    
    Run the `fdisk` utility:
    
    ```bash
    fdisk /dev/sdb
    ```
    
    - Follow these steps:
        - Command: `n` (create new partition)
        - Command: `p` (primary partition)
        - Partition number: `1`
        - Accept defaults for the first and last cylinders by pressing **Enter** twice.
        - Command: `w` (write changes and exit).
2. **Partition `DISK2` and `DISK3`**
    - Repeat the `fdisk` steps for `/dev/sdc` and `/dev/sdd`.
3. **List the Partitions**
    
    Verify the partitions have been created by running:
    
    ```bash
    ls /dev/sd*
    ```
    

### **5.4 Install Oracle ASM Library**

Install the `oracleasm` library and tools:

```bash
sudo dnf install -y kmod-oracleasm oracleasm-support
```

### **5.5 Configure Oracle ASM**

Run the Oracle ASM configuration tool:

```bash
sudo oracleasm configure -i
```

Provide the following inputs:

- Default user to own the driver interface: `grid`
- Default group to own the driver interface: `oinstall`
- Start Oracle ASM library driver on boot: `y`
- Scan for Oracle ASM disks on boot: `y`

### **5.6 Initialize Oracle ASM**

Initialize the Oracle ASM library:

```bash
sudo oracleasm init
```

### **5.7 Create ASM Disks**

 Then, create ASM disks:

```bash
sudo oracleasm createdisk DISK1 /dev/sdb1
sudo oracleasm createdisk DISK2 /dev/sdc1
sudo oracleasm createdisk DISK3 /dev/sdd1

```

### **5.8 Verify ASM Disks**

List the ASM disks:

```bash
sudo oracleasm listdisks

```

---

## **6. Clone Node1 to Create Node2**

### **6.1. Prerequisites**

Ensure the following conditions are met before cloning:

- The source VM (node1) is powered off to prevent data corruption or inconsistencies.
- Adequate disk space is available for the clone.

### **6.2. Cloning Process**

**Step 1: Launch Oracle VirtualBox**

Open Oracle VirtualBox on your host machine.

**Step 2: Select the Source VM**

In the VirtualBox Manager, locate and select the VM you want to clone.

**Step 3: Open the Clone Dialog**

- Right-click the selected VM.
- Click **"Clone"** from the context menu.

**Step 4: Set the New VM Name**

- In the **"Name and Operating System"** section, change the name to `node2`.
- Ensure the machine folder is correct or update it if needed.

**Step 5: Configure Clone Options**

1. **Clone Type:**
    - Select **Full Clone** to create an independent copy of the VM with its own virtual hard disk.
2. **MAC Address Policy:**
    - Choose **Generate new MAC addresses for all network adapters** to avoid MAC address conflicts.
3. Click **Next** to continue.

**Step 6: Confirm and Start Cloning**

- Review the clone settings.
- Click **Clone** to begin the cloning process.
- Wait for the cloning process to complete. This may take several minutes depending on the size of the VM.

### **6.3 Update Hostname** :

- Edit `hostname` on Node2 and change the hostname to `node2`.
    
    ```bash
    hostnamectl set-hostname node2.localdomain
    
    ```
    

### 6.4 Configure Network on node2 :

Since the cloned machine has new MAC addresses, you’ll need to update or recreate the connections to reflect these changes. `nmcli` associates network connections with specific MAC addresses, so the old connections might not work with the new ones. Here’s how you can resolve the issue:

**1. Remove the Old Connections**

First, check the current connections to identify any old ones:

```bash
nmcli connection show
```

If the old connections still reference the previous MAC addresses, you can delete them:

```bash
nmcli connection delete public
nmcli connection delete private
nmcli connection delete internet

```

**2. Create New Connections (with the new MAC addresses)**

Since your cloned machine has the following interface names:

- `enp0s3` for Public
- `enp0s8` for Private
- `enp0s9` for Internet

You can recreate the connections as follows:

**Public Connection (`enp0s3`):**

```bash
nmcli connection add type ethernet con-name "public" ifname enp0s3 \\
ipv4.addresses "192.168.56.106/24" \\
ipv4.gateway "192.168.56.1" \\
ipv4.dns "192.168.56.1" \\
ipv4.method "manual" \\
connection.autoconnect "yes"

```

**Private Connection (`enp0s8`):**

```bash
nmcli connection add type ethernet con-name "private" ifname enp0s8 \\
ipv4.addresses "192.168.10.2/24" \\
ipv4.method "manual" \\
connection.autoconnect "yes"

```

**Internet Connection (`enp0s9`):**

```bash
nmcli connection add type ethernet con-name "internet" ifname enp0s9 \\
ipv4.method "auto" \\
connection.autoconnect "yes"

```

**3. Restart NetworkManager**

After creating new connections, restart the network service:

```bash
sudo systemctl restart NetworkManager

```

**4. Verify and Activate the Connections**

List the connections to verify they are properly configured:

```bash
nmcli connection show

```

Manually activate the connections (if needed):

```bash
nmcli connection up public
nmcli connection up private
nmcli connection up internet

```

This should ensure the cloned machine uses new MAC addresses and IP configurations without conflicts. Let me know if you run into any issues!

### 6.5 Verify the Connection between the Two Nodes:

Login to every machine as root and make sure that they can ping each other :

```bash
ping -c 3 node1
ping -c 3 node1.localdomain
ping -c 3 node1-priv
ping -c 3 node1-priv.localdomain
ping -c 3 node2
ping -c 3 node2.localdomain
ping -c 3 node2-priv
ping -c 3 node2-priv.localdomain

```

---



## **7. Set Up `.bash_profile` for Grid and Oracle Users**

### **7.1 `.bash_profile` for the `grid` User**

The `grid` user is responsible for managing Oracle Grid Infrastructure (GI), including ASM and the clusterware. The `.bash_profile` must be configured on **both Node1 and Node2**.

### **On Node1:**

```bash
su - grid
mv ~/.bash_profile ~/.bash_profile_bkp
vi ~/.bash_profile

```

Add the following lines:

```bash
# Oracle Grid Infrastructure Environment
export ORACLE_HOME=/u01/app/19.0.0/grid
export ORACLE_SID=+ASM1
export ORACLE_BASE=/u01/app/grid
ORACLE_TERM=xterm; export ORACLE_TERM
TNS_ADMIN=$ORACLE_HOME/network/admin; export TNS_ADMIN
PATH=.:${PATH}:$ORACLE_HOME/bin
PATH=${PATH}:/usr/bin:/bin:/usr/local/bin
export PATH
export TEMP=/tmp
export TMPDIR=/tmp
umask 022

```

Save and exit, then apply the changes:

```bash
source ~/.bash_profile

```

### **On Node2:**

```bash
su - grid
mv ~/.bash_profile ~/.bash_profile_bkp
vi ~/.bash_profile

```

Add the following lines:

```bash
# Oracle Grid Infrastructure Environment
export ORACLE_HOME=/u01/app/19.0.0/grid
export ORACLE_SID=+ASM2
export ORACLE_BASE=/u01/app/grid
ORACLE_TERM=xterm; export ORACLE_TERM
TNS_ADMIN=$ORACLE_HOME/network/admin; export TNS_ADMIN
PATH=.:${PATH}:$ORACLE_HOME/bin
PATH=${PATH}:/usr/bin:/bin:/usr/local/bin
export PATH
export TEMP=/tmp
export TMPDIR=/tmp
umask 022

```

Save and exit, then apply the changes:

```bash
source ~/.bash_profile

```

---

### **7.2 `.bash_profile` for the `oracle` User**

The `oracle` user is responsible for managing the Oracle Database. The `.bash_profile` must also be configured on **both Node1 and Node2**.

### **On Node1:**

```bash
su - oracle
mv ~/.bash_profile ~/.bash_profile_bkp
vi ~/.bash_profile

```

Add the following lines:

```bash
# Oracle Database Environment
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export ORACLE_SID=rac1
export ORACLE_BASE=/u01/app/oracle
ORACLE_TERM=xterm; export ORACLE_TERM
NLS_DATE_FORMAT="DD-MON-YYYY HH24:MI:SS"; export NLS_DATE_FORMAT
TNS_ADMIN=$ORACLE_HOME/network/admin; export TNS_ADMIN
PATH=.:${PATH}:$ORACLE_HOME/bin
PATH=${PATH}:/usr/bin:/bin:/usr/local/bin
export PATH
LD_LIBRARY_PATH=$ORACLE_HOME/lib
LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:$ORACLE_HOME/oracm/lib
LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:/lib:/usr/lib:/usr/local/lib
export LD_LIBRARY_PATH
THREADS_FLAG=native; export THREADS_FLAG
export TEMP=/tmp
export TMPDIR=/tmp
export EDITOR=vi
umask 022

```

Save and exit, then apply the changes:

```bash
source ~/.bash_profile
```

### **On Node2:**

```bash
su - oracle
mv ~/.bash_profile ~/.bash_profile_bkp
vi ~/.bash_profile
```

Add the following lines:

```bash
# Oracle Database Environment
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export ORACLE_SID=rac2
export ORACLE_BASE=/u01/app/oracle
ORACLE_TERM=xterm; export ORACLE_TERM
NLS_DATE_FORMAT="DD-MON-YYYY HH24:MI:SS"; export NLS_DATE_FORMAT
TNS_ADMIN=$ORACLE_HOME/network/admin; export TNS_ADMIN
PATH=.:${PATH}:$ORACLE_HOME/bin
PATH=${PATH}:/usr/bin:/bin:/usr/local/bin
export PATH
LD_LIBRARY_PATH=$ORACLE_HOME/lib
LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:$ORACLE_HOME/oracm/lib
LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:/lib:/usr/lib:/usr/local/lib
export LD_LIBRARY_PATH
THREADS_FLAG=native; export THREADS_FLAG
export TEMP=/tmp
export TMPDIR=/tmp
export EDITOR=vi
umask 022

```

Save and exit, then apply the changes:

```bash
source ~/.bash_profile

```

---

## **8. Install Oracle Grid Infrastructure**

### 8.1 Copy Grid Infrastructure Software to your machine using winscp (node1)

as grid user , copy grid software to your grid  home (`/home/grid`) and ensure it’s exist :

```bash
ls /home/grid/LINUX.X64_193000_grid_home.zip
/home/grid/LINUX.X64_193000_grid_home.zip

```

### 8.2 Extract Grid Software in the $ORACLE_HOME of the grid user :

```bash
unzip LINUX.X64_193000_grid_home.zip -d $ORACLE_HOME/

```

### 8.3 Create SSH Passwordless connection to grid user

```bash
cd $ORACLE_HOME/deinstall

```

then you can run the following script to create ssh passwordless :

```bash
./sshUserSetup.sh -user grid -hosts "node1 node2" -noPromptPassphrase -confirm -advanced

```

also create ssh passwordless for oracle user :

```bash
./sshUserSetup.sh -user oracle -hosts "node1 node2" -noPromptPassphrase -confirm -advanced
```

### 8.4 Perform Prerequisite Actions:

In `node1` and `node2`, disable avahi-daemon. This daemon should not be running with Oracle clusterware services.

```bash
systemctl disable avahi-daemon.socket avahi-daemon.service
systemctl mask avahi-daemon.socket avahi-daemon.service
systemctl stop avahi-daemon.socket avahi-daemon.service
```

In `node1` and `node2` as root, disable the chronyd service and rename its configuration file:

```bash
systemctl disable chronyd
mv /etc/chrony.conf /etc/chrony.conf.bak
```

**As `root`, install the cvuqdisk in node1**

The package cvuqdisk must be installed before installing the Clusterware software

```bash
cd /u01/app/19.0.0/grid/cv/rpm/ 
CVUQDISK_GRP=oinstall; export CVUQDISK_GRP 
rpm -iv cvuqdisk-1.0.10-1.rpm
```

then, copy the cvuqdisk package to node2  

```bash
scp cvuqdisk-1.0.10-1.rpm root@node2:/root
```

now in `node2`, install it using `root` 

```bash
rpm -iv cvuqdisk-1.0.10-1.rpm
```

### **8.5 Run the Grid Installer**

On Node1, run the Grid Infrastructure installer:

```bash
cd $ORACLE_HOME
./gridSetup.sh
```

### **Step-by-Step Installation Wizard**

**1. Installation Option**

- **Option:** Install and Configure Oracle Grid Infrastructure for a Cluster
- **Cluster Type:** Configure a Standard Cluster
- **Installation Type:** Advanced Installation
- **Product Language:** English
- **Grid Plug and Play Cluster Name:** rac
- **SCAN Name:** node-scan
- **SCAN Port:** 1521
- **GNS Configuration:** Unmark Configure GNS

**2. Cluster Node Information**

1. Click on the **Add** button.
2. **Public Hostname:** node2.localdomain
3. **Virtual Hostname:** node2-vip.localdomain
4. Click **OK**.
5. Click the **SSH Connectivity** button.
6. Enter the OS Password (grid).
7. Click **Setup** and then **Test**.
    - **Note:** If the setup is successful but the test fails, restart the nodes and try again.
8. Click **Next**.

**3. Network Interface Configuration**

- **enp0s3:** Select Public
- **enp0s8:** Select Private and ASM
- **enp0s9:** Select Do Not Use

**4. Storage Option**

- **Storage Option:** Use Standard ASM for Storage

**5. Create ASM Disk Group**

1. Click **Change Discovery Path**.
2. Enter Disk Discovery Path: `/dev/oracleasm/disks*`.
3. Click **OK**.
4. **Disk Group Name:** CRS
5. **Redundancy:** External
6. **Allocation Unit Size:** 1MB
7. Mark **DISK1**.
8. Click **Next**.

**6. ASM Password**

- **Password Option:** Select "Use same password for these accounts"
- **Specify Password:** oracle
- **Confirm Password:** oracle

**7. Failure Isolation**

- **Option:** Select "Do not use Intelligent Platform Management Interface (IPMI)"

**8. Management Options**

- Click **Next**.

**9. Operating System Groups**

- **OSASM Group:** asmadmin
- **OSDBA for ASM Group:** asmdba
- **OSOPER for ASM Group:** Leave blank

**10. Installation Location**

- **Oracle Base:** `/u01/app/grid`
- **Software Location:** `/u01/app/19.0.0/grid`
    - Note: These values are taken from the OS environment variables.

**11. Create Inventory**

- **Inventory Directory:** `/u01/app/oraInventory`
- **Root Script Execution:** Mark "Automatically run configuration scripts".
- Enter the root password (****).

**12. Prerequisite Checks**

- The verification process may take some time.
- The following warnings can be safely ignored:
    - Physical Memory
    - Swap Size
    - Device Checks for ASM
    - Task resolve.conf Integrity
- **Note:** If other warnings appear, check their details, resolve the issue, and click **Check Again**.
- If necessary, select **Ignore All** and click **Next**.

**13. Summary**

- Click **Install**.

**14. Progress and Installation Notes**

- Around 80% progress, a message may appear. Click **Yes**.
- While the script runs, the progress bar may seem stuck at 84%. This is not a hanging status; wait for it to finish.

**15. Installation Completion**

- At the end of the installation, you may receive the following error:
    - "Oracle Cluster Verification Utility failed."
- The log file will contain details of the error. The following causes can be safely ignored:
    - Sufficient physical memory is not available on the node.
    - Sufficient swap size is not available on the node.
    - Group of device `/dev/oracleasm/disks/DISK1` did not match the expected group.
    - Attempt to get udev information from the node.

### 8.5 Verify the clusterware resources and services:

In node1, check the status of the running clusterware resources. The state of all the resources should be ONLINE:

```bash
crsctl status resource -t

```

Ensure that all the cluster services are up and running in all the cluster nodes:

```bash
crsctl check cluster -all

```

---

## **9. Mounting ASM Disk Groups Using ASMCA**

1. **Login as Grid User:**
    - In the Oracle VirtualBox or VMware window, log in to `node1` as the `grid` user.
2. **Open Terminal:**
    - Open a terminal window and start the `asmca` utility:
        
        ```bash
        asmca
        
        ```
        
3. **Create DATA Disk Group:**
    - Click on the **Create** button.
    - In the **Disk Group Name** field, enter `DATA`.
    - For **Redundancy**, select "External".
    - Mark `DISK2`.
    - Click **OK**.
    - Wait for a few seconds until a success message appears confirming the disk group creation.
4. **Create FRA Disk Group:**
    - Repeat the process for the FRA disk group:
        - **Disk Group Name:** FRA
        - **Redundancy:** External
        - Mark the relevant disk.
    - Click **OK**.
5. **Verify Disk Groups:**
    - Ensure that all the disk groups are mounted and accessible from both nodes.
    - The `asmca` window should display both `DATA` and `FRA` as mounted.
    - Click **Exit**.

---

## **10. Install Oracle Database Software**

perform the following steps as `oracle` user to install oracle database software 

### 10.1 Copy the oracle database software to oracle home ( `/home/oracle`) using winscp

### 10.2 unzip the software in the `$ORACLE_HOME` :

```bash
unzip LINUX.X64_193000_db_home.zip -d $ORACLE_HOME
```

### 10.3 Creating `sqlnet.ora` file

 create the `sqlnet.ora` file  

```bash
mkdir -p $ORACLE_HOME/network/admin 
vi $ORACLE_HOME/network/admin/sqlnet.ora 
```

then , add the following code in it

```bash
DIAG_ADR_ENABLED=ON 
NAMES.DIRECTORY_PATH= (TNSNAMES, EZCONNECT)
```

### **10.4 Run the Database Installer**

On Node1, run the Database installer:

```bash
cd $ORACLE_HOME
./runInstaller

```

### **Step-by-Step Installation Wizard**

1. **Configure Security Updates:**
    - Uncheck the option "I wish to receive security updates...".
    - Click **Next**, then click **Yes**.
2. **Installation Option:**
    - Select "Install database software only".
    - Click **Next**.
3. **Grid Installation Options:**
    - Select "Oracle Real Application Clusters database installation".
    - Click **Next**.
4. **Nodes Selection:**
    - Ensure both nodes are selected.
    - Click **SSH Connectivity**.
    - Enter the `oracle` password.
    - Click **Setup**, and after the SSH connectivity setup is finished, click **Test**.
    - Click **Next**.
5. **Product Language:**
    - Ensure English is selected.
    - Click **Next**.
6. **Database Edition:**
    - Select "Enterprise Edition".
    - Click **Next**.
7. **Installation Location:**
    - **Oracle Base:** `/u01/app/oracle`
    - **Software Location:** `/u01/app/oracle/product/19.0.0/db_home/`
    - Note: These values are taken from the OS environment variables.
8. **Operating System Groups:**
    - Select the `dba` group for all options except `OSOPER` (leave it blank).
    - Click **Next**.
9. **Prerequisite Checks:**
    - The following warnings can be safely ignored:
        - Swap Size
        - Task `resolv.conf` Integrity
        - Single Client Access Name (SCAN)
    - If other warnings appear, check their details, resolve the issues, and click **Check Again**.
    - If necessary, click **Ignore All** and click **Next**.
10. **Summary:**
    - Click **Install**.
11. **Execute Configuration Scripts:**
    - When prompted, execute the root script on both nodes:
        - Run the script on the first node.
        - After it completes, run the script on the second node.
    - Click **Close**.

---

## **11. Create a RAC Database**

### **11.1 Run DBCA**

On Node1, run the Database Configuration Assistant (DBCA):

```bash
dbca
```

### **Step-by-Step DBCA Wizard**

1. **Database Operation:**
    - Select **Create Database**.
    - Click **Next**.
2. **Creation Mode:**
    - Select **Advanced Mode**.
    - Click **Next**.
3. **Database Template:**
    - Ensure the **Database Type** is **Oracle Real Application Clusters (RAC)**.
    - Change the **Configuration Type** to **Admin Managed**.
    - Select the **General Purpose or Transaction Processing** option.
    - Click **Next**.
4. **Database Identification:**
    - **Global Database Name:** `rac.localdomain`
    - **SID Prefix:** `rac`
    - Uncheck the "Create As Container Database" option.
    - Click **Next**.
5. **Database Placement:**
    - Move the node `node2` from the **Available** list to the **Selected** list.
    - Click **Next**.
6. **Management Options:**
    - Uncheck the **"Run Cluster Verification Utility (CVU) Checks Periodically"** checkbox.
    - Check the **"Configure Enterprise Manager (EM) Database Express"** checkbox.
    - Click **Next**.
7. **Database Credentials:**
    - Set passwords for the users (e.g., `SYS`, `SYSTEM`).
    - Click **Next**.
8. **Storage Locations:**
    - **Database Files Storage Type:** Select `ASM`.
    - **Common Location for All Database Files:** `+DATA`
    - Check the **"Use Oracle-Managed Files"** checkbox.
    - **Recovery Files Storage Type:** `ASM`
    - Check the **"Specify Fast Recovery Area"** checkbox.
    - **Fast Recovery Area:** `+FRA`
    - **Fast Recovery Size:** `12 GB`
    - Ensure the **"Enable Archiving"** checkbox is **unmarked**.
    - Click **Next**.
9. **Database Options:**
    - Check the **"Sample Schemas"** checkbox.
    - Click **Next**.
10. **Initialization Parameters:**
    - **Memory:** Move the slider to nearly 50%.
        - **SGA Size:** `1500 MB`
        - **PGA Size:** `512 MB`
    - Click the **Sizing** tab:
        - **Processes:** `500`
    - Click the **Character Sets** tab:
        - Select **"Use Unicode (AL32UTF8)"**.
    - Click **Next**.
11. **Creation Options:**
    - Ensure the **"Create Database"** option is selected.
    - Click **Next**.
12. **Prerequisite Checks:**
    - Ignore the following warnings:
        - **Single Client Access Name (SCAN)**
    - Click **Ignore All**.
    - Click **Next**.
13. **Summary:**
    - Review the details.
    - Click **Finish**.
14. **Database Configuration Assistant:**
    - Wait for the database creation to complete.
    - Click **Close** when done.

---

## **12. Verification and Testing**

Make sure that the SCAN hostname replies to ping command.

```bash
ping -c 3 node-scan

```

Issue the following commands and examine their output

```bash
srvctl status database -d rac
srvctl config database -d rac

```

Identify the database instance names that are currently running on `node1` from the OS command shell:

```bash
ps -ef | grep -i pmon

```

Make sure the tnsnames.ora file has been configured for connecting to rac database. This has automatically been done by the dbca utility:

```bash
cat $TNS_ADMIN/tnsnames.ora

```

---
