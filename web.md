# Making the logs human-readable

## Step 1: Install a small web server
```
sudo apt install nginx -y
```
## Step 2: Create a public folder for the report
```
sudo mkdir -p /var/www/oltlogs
```
## Step 3: Configure nginx to serve it on port 8080
```
sudo nano /etc/nginx/sites-available/oltlogs
```
```
server {
    listen 8080;
    server_name _;
    root /var/www/oltlogs;
    autoindex on;
    autoindex_exact_size off;
    autoindex_localtime on;
    location / {
        try_files $uri $uri/ =404;
    }
}
```
- save and exit
```
sudo ln -s /etc/nginx/sites-available/oltlogs /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```
## Step 4: Write a script that converts raw logs into a readable table
Created `/usr/local/bin/publish_olt_log.sh`, which:
Loops through every device folder under `/var/log/remote/`
Reads recent log lines from each
Classifies each line — login success/failure, logout, config save, backup,
customer equipment online/offline/dying gasp
Pulls out timestamp, username (where reported), and source IP
Builds one combined, color-coded HTML table, labeled by device
Saves it to `/var/www/oltlogs/olt_log.html` for nginx to serve

Full script:
```
sudo tee /usr/local/bin/publish_olt_log.sh > /dev/null << 'SCRIPT_END'
#!/bin/bash
BASE="/var/log/remote"
DEST="/var/www/oltlogs/olt_log.html"
LOCK="/tmp/publish_olt_log.lock"

exec 200>"$LOCK"
flock -n 200 || exit 0

TMP=$(mktemp /var/www/oltlogs/olt_log.XXXXXX.html)

{
echo "<html><head><title>OLT Activity Log - All Sites</title>"
echo "<style>
body { font-family: Arial, sans-serif; margin: 20px; background:#f4f4f4; }
h2 { color:#222; }
table { border-collapse: collapse; width: 100%; background:#fff; }
th, td { border: 1px solid #ccc; padding: 6px 10px; font-size: 13px; text-align:left; }
th { background:#333; color:#fff; }
tr.fail { background:#ffe0e0; }
tr.success { background:#e0ffe0; }
tr.logout { background:#fff3cd; }
tr.config { background:#e0f0ff; }
tr.onu { background:#f0e0ff; }
tr.info { background:#ffffff; }
.badge-fail { color:#b30000; font-weight:bold; }
.badge-success { color:#006600; font-weight:bold; }
.badge-logout { color:#8a6d00; font-weight:bold; }
.device { font-weight:bold; color:#004080; }
</style></head><body>"
echo "<h2>OLT Activity Log - All Sites</h2>"
echo "<p>Last updated: $(date)</p>"
echo "<table>"
echo "<tr><th>Time</th><th>Device</th><th>User</th><th>Activity</th><th>Source IP</th><th>Result</th></tr>"

for dir in "$BASE"/*/; do
  device=$(basename "$dir")
  logfile="$dir/traffic.log"
  [ -f "$logfile" ] || continue

  tail -n 100 "$logfile" | tr -d '\r' | while read -r line; do
    ts=$(echo "$line" | awk '{print $1}')
    ip=$(echo "$line" | grep -oP '(?:from|session from) \K[0-9.]+' | head -1)
    user=$(echo "$line" | grep -oP "User \K[A-Za-z0-9_.-]+(?= logout| logged| authentication)" | head -1)
    [ -z "$user" ] && user=$(echo "$line" | grep -oP "User '\K[^']*" | head -1)
    [ -z "$user" ] && user="-"

    if echo "$line" | grep -qi "AUTHENTICATION_FAILED\|Failed"; then
      echo "<tr class='fail'><td>$ts</td><td class='device'>$device</td><td>$user</td><td>Login attempt</td><td>${ip:-unknown}</td><td class='badge-fail'>FAILED</td></tr>"
    elif echo "$line" | grep -qi "Succeeded"; then
      echo "<tr class='success'><td>$ts</td><td class='device'>$device</td><td>$user</td><td>Login</td><td>${ip:-unknown}</td><td class='badge-success'>SUCCESS</td></tr>"
    elif echo "$line" | grep -qi "User Logout\|logouted\|CLOSE"; then
      echo "<tr class='logout'><td>$ts</td><td class='device'>$device</td><td>$user</td><td>Logout</td><td>${ip:-unknown}</td><td class='badge-logout'>SESSION ENDED</td></tr>"
    elif echo "$line" | grep -qi "Config Save\|Config Auto Save\|Auto Backup\|Upload File"; then
      msg=$(echo "$line" | grep -oP '(?<=auditd\[0\]: ).*' | head -c 80)
      echo "<tr class='config'><td>$ts</td><td class='device'>$device</td><td>$user</td><td>$msg</td><td>${ip:-unknown}</td><td>-</td></tr>"
    elif echo "$line" | grep -qi "ONU Online\|ONU Offline\|ONU Dying Gasp"; then
      msg=$(echo "$line" | grep -oP '(?<=auditd\[0\]: ).*' | head -c 80)
      echo "<tr class='onu'><td>$ts</td><td class='device'>$device</td><td>$user</td><td>$msg</td><td>${ip:-unknown}</td><td>-</td></tr>"
    elif echo "$line" | grep -qi "SESSION request"; then
      echo "<tr class='info'><td>$ts</td><td class='device'>$device</td><td>$user</td><td>Session requested</td><td>${ip:-unknown}</td><td>-</td></tr>"
    fi
  done
done | sort -r

echo "</table></body></html>"
} > "$TMP"

mv "$TMP" "$DEST"
chmod 644 "$DEST"
SCRIPT_END
```
- execute script

      sudo chmod +x /usr/local/bin/publish_olt_log.sh



# The result
Open a browser and go to:
```
http://your_IP:8080/olt_log.html
```
One combined, color-coded table across every OLT:

      Column	Meaning
      Time	When it happened
      Device	Which OLT/site
      User	Who did it, where reported
      Activity	Login, logout, config saved, backup, customer box online/offline
      Source IP	Where the request came from
      Result	Success or failure
      🟩 Green = successful login
      🟥 Red = failed login attempt
      🟨 Yellow = logout
      🟦 Blue = configuration saved/backed up
      🟪 Purple = customer equipment online/offline/dying gasp
      Retention: 30 days, rotating daily. Webpage: refreshes every 5 minutes,
      fully automatic.
      ---
stuffing attack against the OLTs' remote management access, apparently
succeeding in some cases. Flagged to the network/security team for urgent
review, separate from this logging project.
