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
      - Disk Size (GiB): Set this to **30 or 40 GiB**. Since OLT logs are plain text, this gives you plenty of runway before you ever have to worry about log rotation filling it up.
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
      
      Once the VM spins up, you can click on **122 (SYSLOG)** in your left-hand menu, open the **Console**, and go through the standard OS installation wizard. Let me know when you get to the login prompt!
      At the end of Part 1: a blank Ubuntu server, ready to configure, but nothing set up yet to receive or store logs.
