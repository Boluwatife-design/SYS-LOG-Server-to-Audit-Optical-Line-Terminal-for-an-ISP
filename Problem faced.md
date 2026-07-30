# Problem 1: "Too many open files"
- Cause: an earlier/incorrect config created a brand-new file for every single log message instead of appending to one file per device — tens of thousands of tiny files piled up.
- Fix: corrected the template to key only on the device and archived the old debris:
```
mkdir -p /var/log/remote/10.80.50.6/old_debris
find /var/log/remote/10.80.50.6/ -maxdepth 1 -regextype posix-extended \
  -regex '.*/[0-9]+\.log' -exec mv {} /var/log/remote/10.80.50.6/old_debris/ \;
```
# Problem 2: The VM's own local activity mixing into OLT logs
- Cause: the first working version of the routing rule used `%HOSTNAME%` with no filter, so the catch-all rule was also capturing the VM's own local system messages (SSH logins to the VM, cron, etc.).
- Fix: switched to `%FROMHOST-IP%` (the real source address, not a self-reported field) and added the `127.0.0.1` exclusion shown in syslog.md,

# Problem 3: Fix corrupted output
- Cause: traced back to the terminal/console used to paste the script mangling characters on long pastes — not a logic bug.
- Fix: Used a proper SSH client instead of the Proxmox browser console, so paste worked reliably Wrote the script as a single atomic block using `tee` with a heredoc, instead of pasting line-by-line into `nano` (which was silently dropping/
mangling characters on long pastes) Added file locking (`flock`) and unique temp files (`mktemp`) so two runs could never collide and corrupt each other's output

# Problem 3: Fix a 403 Forbidden error
- Cause: the generated file was owned by `root` with permissions too strict for the web server to read.
- Fix: added to the end of the script:
```
chmod 644 "$DEST"
