# SYS-LOG-Server-to-Audit-Optical-Line-Terminal-for-an-ISP

## How I Built OLT Log Auditing — Step by Step
This is the full journey, in order: from creating the virtual machine on Proxmox, to configuring it to receive logs from the OLTs, to viewing those logs as a clean webpage anyone can open in a browser.


# PART 1 — Creating the VM on Proxmox

(Standard process for what was done before this configuration work began.)

- Logged into the Proxmox web interface (the hypervisor — software that lets one physical server host many virtual machines).
- Clicked Create VM, gave it a name (oltsyslog).
- Attached an Ubuntu Server ISO and installed it.
- Leave system with default settings
- Assigned CPU, RAM, and disk — a syslog collector doesn't need much; 2 vCPU, 2–4GB RAM, and 20–30GB disk is comfortable for months of compressed logs.
- Connected it to the network bridge that could reach the OLTs.
<img width="1080" height="871" alt="image" src="https://github.com/user-attachments/assets/95c32375-bb68-464d-867a-8ff9123c231b" />


       1. OS Tab
      
      -ISO Image: Click the dropdown and select your Ubuntu Server or Debian ISO. (If it's not already uploaded, you'll need to upload it to your local storage first).
         Type: Linux
         Version: Keep the default selection.
         Click Next.
      
      2. System Tab
      
      Leave everything at their defaults here. Proxmox defaults work flawlessly for standard Linux VMs.
      Click Next.
      
      3. Disks Tab
      
      - Storage: Select the storage where your other VMs live (likely `local-lvm` or whatever network storage you use).
      - Disk Size (GiB): Set this to 30 or 40 GiB. Since OLT logs are plain text, this gives you plenty of runway before you ever have to worry about log rotation filling it up.
      - Click Next.
      
      4. CPU Tab
      
      - Cores: Set this to 1 or 2. Syslog processing is incredibly lightweight.
      - Click Next.
      
      5. Memory Tab
      
      - Memory (MiB): Set this to your desired RAM.
       Click Next.
      
      6. Network Tab
      
      - Bridge: Keep it on `vmbr0` (or whichever bridge matches your main management network).
      - VLAN Tag: If your Proxmox host port is trunked and your network team told you to put the syslog server on a specific management VLAN (e.g., VLAN 50), enter that number here. If you use untagged access ports, leave it blank.
       Click Next.
      
      7. Confirm Tab
      
      - Review the summary.
      - Check the box that says "Start after created" if you want it to boot right up, then click Finish.
      
      Once the VM spins up, you can click on 122 (SYSLOG) in your left-hand menu, open the Console, and go through the standard OS installation wizard.
      At the end of this: a blank Ubuntu server, ready to configure, but nothing set up yet to receive or store logs.

# Install the Ubuntu Server operating system

- In the left-hand panel of your Proxmox dashboard, click on your newly created VM.
- If it hasn't started yet, click the Start button in the top right corner.
- Click on the Console tab right below the Proxmox summary grid, the Ubuntu installation screen boot up.
- Language & Keyboard: Select your preferred layout (usually English).
  <img width="1912" height="1102" alt="image" src="https://github.com/user-attachments/assets/77d94291-3d78-4dc1-8a3a-9a0ac6ee2a40" />

- Type of Install: Choose Ubuntu Server
- Network Connections: By default, it will grab a random IP via DHCP.

## Crucial Step: If you have a static IP, select your network interface (eth0 or ens18), press Enter, choose Edit IPv4, change it from DHCP to Manual, and enter your static IP details.
<img width="1291" height="806" alt="image" src="https://github.com/user-attachments/assets/0f687de5-0d00-44a5-901e-1bcdd5c9b61c" />

- If you don't have the static IP yet, just leave it on DHCP for now. You can easily change it to static in the command line later using:

                       Log into your Ubuntu server.
                       Open the network configuration file using this command:
                              sudo nano /etc/netplan/50-cloud-init.yaml
                       Update the addresses (your IP/CIDR), routes (your gateway), and nameservers fields with the exact info they gave you.
                       Save the file and apply the changes instantly with:
                              sudo netplan apply
  
- Storage Configuration: Leave it on Use an entire disk (the 32GB virtual disk you created). Uncheck the box for "Set up this disk as an LVM logical volume" to keep things simple, then select Done.
  
- Profile Setup: Enter your name, choose a server name (e.g., syslog-server), and create a secure username and password. Remember these credentials!
  <img width="1275" height="698" alt="image" src="https://github.com/user-attachments/assets/9b7e2e59-5bdb-408a-8804-490315386350" />

- SSH Setup: Check the box to Install OpenSSH server. This is important so you can SSH into this box from your own laptop later instead of using the Proxmox browser console.(Leave [X] Allow password authentication over      SSH checked as it is)
  <img width="1275" height="785" alt="image" src="https://github.com/user-attachments/assets/ad6d8a3c-9e56-48b9-84d1-0dc14cd2d198" />

- Featured Server Snaps: Don't select anything here. Just hit Done and reboot the virtual machine.
  <img width="1280" height="808" alt="image" src="https://github.com/user-attachments/assets/0e91f406-2918-4b0d-b0bd-07488bfaa49f" />

- During the reboot process, the screen might freeze or show a message like "Please remove the installation medium, then press ENTER" Just Click Enter.
  <img width="1295" height="802" alt="image" src="https://github.com/user-attachments/assets/b724630f-ae2f-4d56-8b5a-9b0700822214" />

  
  ### If you have an IP

- Open the Network Menu
- Ensure [ ens18 eth - ▶ ] is highlighted in white.
- Press Enter. A small dropdown menu will appear.
- Use the down arrow key to select Edit IPv4 and press Enter.
- Switch to Manual Mode
- On the "IPv4 Method" line, press Enter and change it from Automatic (DHCP) to Manual.
- Press Enter. You will see a box with empty lines for Subnet, Address, Gateway, and Name servers.
- Enter your Details (Type in your network settings using the layout below)

(Note: In the Subnet line, Ubuntu requires CIDR notation—meaning the network range followed by /24 or /22, but adjust if your network team gave you a different subnet mask like 255.255.255.0):
- Subnet: 10.80.50.0/24 (Change the 10.80.50 part to match the first three octets of your specific IP).
- Address: [Your_Allocated_Static_IP]
- Gateway: Input your network gateway IP
- Name servers: 1.1.1.1, 8.8.8.8 (You can separate multiple DNS servers with a comma).
- Leave the Proxy address field completely blank (In most corporate environments, servers don't need a proxy to access the internet unless the network team explicitly forces all traffic through an explicit proxy appliance)
- Save and press Enter.

### The main network configuration screen will reappear, and you should now see your static IP listed next to ens18.

- For the storage (Disable set up this disk as an LVM group to keeps the partition layout standard to make it easier to expand the disk layout if OLT logs grow fast)
<img width="1189" height="722" alt="image" src="https://github.com/user-attachments/assets/a5b113c9-5a6f-4b5d-a9a1-87cfcad48975" />
<img width="1251" height="786" alt="image" src="https://github.com/user-attachments/assets/8c6ba355-d7ea-46f9-86db-4924edb1d984" />



