Got it! Since your VMs are stored in **`D:\virtual_machines`** on Windows, here’s a **simplified, step-by-step guide** to run them remotely on Linux Mint while keeping the files on Windows.  

---

### **Method: SSH + VirtualBox Headless Mode (Using Windows Storage)**  
✅ **What this does:**  
- Linux Mint **uses its CPU/RAM** to run the VM.  
- The **VM files stay on Windows** (`D:\virtual_machines`).  
- You control everything via **SSH** from Windows.  

---

### **Step 1: Share the VM Folder on Windows**  
1. **Right-click** `D:\virtual_machines` → **Properties** → **Sharing** → **Advanced Sharing**.  
2. Check **Share this folder**, set **Permissions** to **Full Control** for your user.  
3. Note your Windows **IP address** (`ipconfig` in CMD).  

---

### **Step 2: Mount the Windows Folder on Linux Mint**  
On Linux Mint, open a terminal and run:  
```bash
sudo mkdir -p /mnt/win_vms  # Create mount folder
sudo mount -t cifs //WINDOWS_IP/virtual_machines /mnt/win_vms -o username=WIN_USER,password=WIN_PASS,uid=$USER
```  
Replace:  
- `WINDOWS_IP` = Your Windows PC’s local IP (e.g., `192.168.1.100`).  
- `WIN_USER` = Your Windows username.  
- `WIN_PASS` = Your Windows password.  

*(If you get errors, try adding `,vers=2.0` or `,vers=3.0` to the command.)*  

---

### **Step 3: Install VirtualBox on Linux Mint**  
```bash
sudo apt update && sudo apt install virtualbox -y
```

---

### **Step 4: Register & Start the VM**  
1. **List available VMs** (check `.vbox` files):  
   ```bash
   ls /mnt/win_vms
   ```
2. **Register the VM** (replace `Your_VM` with your VM folder name):  
   ```bash
   VBoxManage registervm "/mnt/win_vms/Your_VM/Your_VM.vbox"
   ```
3. **Start the VM in headless mode** (runs in background):  
   ```bash
   VBoxHeadless --startvm "Your_VM_Name" --vrde on
   ```
   - `Your_VM_Name` = Name shown in `VBoxManage list vms`.  
   - `--vrde on` allows Remote Desktop access (optional).  

---

### **Step 5: Access the VM from Windows**  
#### **Option A: Remote Desktop (RDP) for Windows/Linux VMs**  
- Open **Remote Desktop Connection** (`mstsc`) on Windows.  
- Connect to: `linux_mint_ip:3389`.  

#### **Option B: SSH (For Linux VMs)**  
1. Find the VM’s IP:  
   ```bash
   VBoxManage guestproperty get "Your_VM_Name" /VirtualBox/GuestInfo/Net/0/V4/IP
   ```
2. SSH into it:  
   ```bash
   ssh username@VM_IP
   ```

---

### **Troubleshooting**  
🔹 **Permission denied?**  
   - Add `,gid=$USER` to the `mount` command.  
   - Or run: `sudo chmod -R 777 /mnt/win_vms` (temporarily).  

🔹 **VM not starting?**  
   - Check logs: `tail -f ~/.config/VirtualBox/VBox.log`.  
   - Ensure VirtualBox version matches on both systems.  

🔹 **Auto-mount on reboot?**  
   Add this to `/etc/fstab`:  
   ```
   //WINDOWS_IP/virtual_machines  /mnt/win_vms  cifs  username=WIN_USER,password=WIN_PASS,uid=1000  0  0
   ```

---

### **Why This Works**  
- **No file duplication** – VMs stay on Windows.  
- **Linux handles execution** – Better performance if Linux has more RAM/CPU.  
- **Full remote control** – Manage via SSH or RDP.  

Let me know if you’d like help with any specific step! 🚀
