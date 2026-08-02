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
  Note: This means it Starts listening on port 514 for anyone trying to send a report, using either of the two common delivery methods (UDP and TCP)

# Step 3: Route incoming logs into per-device folders, filtered correctly
- So Whenever a message arrives, it looks at the real network address it actually came from (`%FROMHOST-IP%` — not a self-reported hostname field, which can be blank or spoofed), create a folder named after that
address/device if needed, and write the message into a file called `traffic.log` inside it
- But if the message came from this VM itself (`127.0.0.1`, the standard address every machine uses to refer to itself), skip it — that's local noise, and count as not something to audit."
- Scroll all the way down to the very bottom of the file, make a new line, and paste this rule:
  
      # Enhanced template to split remote logs by hostname/IP
      $template RemoteLogs,"/var/log/remote/%FROMHOST-IP%/traffic.log"
      if $fromhost-ip != '127.0.0.1' then ?RemoteLogs
      & stop
- So the whole Script looks like this:

        
            # /etc/rsyslog.conf configuration file for rsyslog
            #
            # For more information install rsyslog-doc and see
            # /usr/share/doc/rsyslog-doc/html/configuration/index.html
            #
            # Default logging rules can be found in /etc/rsyslog.d/50-default.conf
            
            
            #################
            #### MODULES ####
            #################
            
            module(load="imuxsock") # provides support for local system logging
            #module(load="immark")  # provides --MARK-- message capability
            
            # provides UDP syslog reception
            module(load="imudp")
            input(type="imudp" port="514")
            
            # provides TCP syslog reception
            module(load="imtcp")
            input(type="imtcp" port="514")
            
            # provides kernel logging support and enable non-kernel klog messages
            module(load="imklog" permitnonkernelfacility="on")
            
            ###########################
            #### GLOBAL DIRECTIVES ####
            ###########################
            
            # Filter duplicated messages
            $RepeatedMsgReduction on
            
            #
            # Set the default permissions for all log files.
            #
            $FileOwner syslog
            $FileGroup adm
            $FileCreateMode 0640
            $DirCreateMode 0755
            $Umask 0022
            $PrivDropToUser syslog
            $PrivDropToGroup syslog
            
            #
            # Where to place spool and state files
            #
            #
            $WorkDirectory /var/spool/rsyslog
            
            #
            # Include all config files in /etc/rsyslog.d/
            #
            $IncludeConfig /etc/rsyslog.d/*.conf
            
            # Enhanced template to split remote logs by hostname/IP
            $template RemoteLogs,"/var/log/remote/%FROMHOST-IP%/traffic.log"
            if $fromhost-ip != '127.0.0.1' then ?RemoteLogs
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

# Step 6: Confirm logs were actually arriving
```
sudo tcpdump -i any -n port 514 -c 20
```
Result: confirmed real UDP packets arriving from an OLT IP address
(`ip`), proving that device was already sending and the VM was
receiving.
```
ls -lt /var/log/remote/
```
Important, accurate detail: at this point, only one folder existed — `10.x.x.x`. Folders do not all appear automatically just because rsyslog is configured and listening. A device's folder only shows up once that specific device is actually:
- Configured (on its own end) to send its syslog output to this VM's IP, and Able to actually reach this VM over the network (routing/firewall rules in place).
- At this stage, only `10.x.x.x` met both conditions. The other OLT sites had not been pointed at this collector yet on their own end. Additional folders appeared later, after the network team made changes on their side — most likely one or both of:
- Configuring each additional OLT's syslog/remote-logging setting to point at vm IP
- Opening routing/firewall rules so traffic from those additional OLTs' subnets could actually reach the VM
- running `ls -lt /var/log/remote/` again some time later showed the new folders had appeared on their own, with no further changes made on this VM.
- This is expected behavior: rsyslog creates a device's folder automatically the first time it receives a message from that device — nothing needs to be pre-configured per device on the collector side.

# Step 7: Confirm retention was actually working
```
cat /etc/logrotate.d/remote-logs
```
```
/var/log/remote/*/*.log {
    daily
    rotate 30
    missingok
    notifempty
    compress
    delaycompress
    sharedscripts
    postrotate
        /usr/lib/rsyslog/rsyslog-rotate
    endscript
}
```
In plain terms: every day, the current log file is renamed and compressed, a fresh one starts, and anything older than 30 rotations is deleted — a rolling 30-day history, disk usage kept in check.
Confirmed the daily scheduler was really running, not just configured:
```
systemctl status logrotate.timer
systemctl list-timers | grep logrotate
```




