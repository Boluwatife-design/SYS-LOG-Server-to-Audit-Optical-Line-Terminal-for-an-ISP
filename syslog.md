# Managing via SSH
- Open the standard Terminal or Command Prompt directly on your personal laptop and type:

      ssh username@IP
## Verify Your Current Network Setup

      ip a
- Check if the server can see its own gateway

      ip route
- Verify you have external internet access by running a quick ping test

      ping -c 4 google.com
- Verify if the server can see the network

      ping -c 4 'network ip'

# Step 1: Open the Rsyslog Configuration File
- Since rsyslog is installed by default on Ubuntu, You just need to modify its main configuration file. Paste this command into your SSH session:

      sudo nano /etc/rsyslog.conf

# Step 2: Enable UDP and TCP Syslog Reception
- Find the section for Modules. Check lines for UDP and TCP reception that are currently commented out with a # symbol.
- Remove the # from the front of those lines to activate them, so the section should look like this:

      # provides UDP syslog reception
      module(load="imudp")
      input(type="imudp" port="514")
      
      # provides TCP syslog reception
      module(load="imtcp")
      input(type="imtcp" port="514")

# Step 3: Create a Custom Template for Foreign Logs
- If you don't set up a template, all incoming logs from firewalls, switches, and other servers will completely flood your VM's local /var/log/syslog file, making it a nightmare to read.
- Tell rsyslog to automatically parse incoming messages and sort them cleanly into separate folders named after the device's IP or hostname.
- Scroll all the way down to the very bottom of the file, make a new line, and paste this rule:
  
      # Custom template to split remote logs by hostname/IP
      $template RemoteLogs,"/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log"
      *.* ?RemoteLogs
      & stop

# Step 4: Save, Exit, and Restart the Daemon
- Press Ctrl + O then Enter to save your work.
- Press Ctrl + X to exit the nano editor.
- Restart the rsyslog service to apply your new configurations:

      sudo systemctl restart rsyslog

# Step 5: Verify the Syslog Ports are Listening
- Let's make sure the Ubuntu firewall layer and the rsyslog daemon are actively keeping port 514 open. Run this command:

      ss -tulpn | grep 514

<img width="380" height="110" alt="image" src="https://github.com/user-attachments/assets/35954124-a084-4fda-995e-fcb19686e1b8" />

- Ensure log directory ownership (make sure it has the authority to write files to the custom path that was built)

      sudo chown -R syslog:adm /var/log
- Tell your terminal to monitor the remote directory in real-time

      watch ls -R /var/log/remote/
- Open a second Command Prompt tab on your laptop, SSH back in, and trigger a test log using the logger utility pointing to your server's own network interface:

      logger -n 'your_sys_log_IP' -T -p local0.notice "Test Log From Oltsyslog Wire"
- Look back at your first terminal window running the watch command. If the template rules are working properly, you should see a brand-new folder, containing a clean text log file
