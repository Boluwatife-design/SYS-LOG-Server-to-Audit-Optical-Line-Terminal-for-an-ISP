# SYS-LOG-Server-to-Audit-Optical-Line-Terminal-for-an-ISP

## How I Built OLT Log Auditing — Step by Step
This is the full journey, in order: from creating the virtual machine on Proxmox, to configuring it to receive logs from the OLTs, to viewing those logs as a clean webpage anyone can open in a browser.


# OLT Log Auditing — Explained Simply
- The problem: Our network devices (OLTs) that deliver internet service were constantly recording their own activity — who logged in, what changed, whether a customer's equipment went offline — but that information was never being
saved anywhere. It was like a security guard shouting a report into an empty room: said once, then gone forever.
- What we built
Think of it like a security camera system:
The OLTs are the front doors — every time someone tries to log in, change a setting, or something happens with a customer's equipment, the OLT writes down what happened. Our new server (the VM) is the security office — it now receives every
one of those reports automatically, organizes them by device, and keeps them safely stored. The webpage is a notice board in that office — instead of digging through raw text files, anyone can open a browser and see a clean, color-
coded table of everything that's happened.
- How to use it
Open a browser and go to:
```
http://172.16.77.30:8080/olt_log.html
```
You'll see a table with:
- Column	What it means
- Time	When it happened
- Device	Which site/OLT it came from
- User	Who did it (if the device reports a name)
- Activity	What happened — login, logout, config saved, backup taken, customer box online/offline
- Source IP	Where the request came from
- Result	Success or failure
- Colors tell the story at a glance:

  
      🟩 Green = someone logged in successfully
      🟥 Red = a login attempt failed
      🟨 Yellow = someone logged out
      🟦 Blue = a configuration was saved or backed up
      🟪 Purple = a customer's equipment went online, offline, or lost power

How long do we keep this?
- 30 days. Every day, the current log is archived and compressed, and anything older than 30 days is automatically deleted — so we always have a month of history without the storage growing forever.
Does the page update on its own?
- Yes — it refreshes automatically every 5 minutes. No one needs to run anything manually for it to stay current.
