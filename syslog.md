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
  
