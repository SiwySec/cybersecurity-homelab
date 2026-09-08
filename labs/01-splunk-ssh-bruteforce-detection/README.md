SSH Brute Force Detection with Splunk

#ssh-brute-force-detection-with-splunk

This project shows how I use Splunk to detect a SSH brute force attack in my home lab. I generate the attack myself with Kali Linux, then find it in Splunk using a search query, and turn that query into an alert.

Goal

#goal

The goal is to detect multiple failed SSH login attempts from the same IP address in a short time window, which is a common sign of a brute force attack.

Lab Setup

#lab-setup

Component	Role
Kali Linux VM	Attacker machine, runs the brute force attempt
Target VM (Ubuntu)	Victim machine with SSH enabled
Splunk SIEM	Collects and analyzes the logs
Splunk Universal Forwarder	Installed on the target VM, sends /var/log/auth.log to Splunk
Step 1: Generate the Attack

#step-1-generate-the-attack

From the Kali VM, I used hydra to attempt several SSH logins against the target with a wordlist of common passwords:

hydra -l testuser -P /usr/share/wordlists/rockyou.txt ssh://192.168.5.X

This creates a burst of failed login attempts in the target's auth.log, which gets forwarded to Splunk.

Step 2: Find the Attack in Splunk

#step-2-find-the-attack-in-splunk

In Splunk, I searched the forwarded logs for failed password events:

spl
index=linux_logs sourcetype=linux_secure "Failed password"
| stats count by src_ip
| where count > 5

This query counts failed login attempts grouped by source IP, and only shows IPs with more than 5 failures. The Kali VM's IP shows up clearly with a high count.

Step 3: Turn It into an Alert

#step-3-turn-it-into-an-alert

I saved the search as an alert that runs every 5 minutes and triggers when the condition is met.

Setting	Value
Search	same query as above
Schedule	Every 5 minutes
Trigger condition	Number of results > 0
Action	Add to Triggered Alerts list

Screenshot: screenshots/alert-config.png

Step 4: Result

#step-4-result

Screenshot: screenshots/splunk-search-results.png

Screenshot: screenshots/triggered-alert.png

The alert fired a few minutes after I started the hydra attack, showing the Kali VM's IP with over 100 failed attempts in a 5 minute window.

Lessons Learned

#lessons-learned

The default linux_secure sourcetype in Splunk already parses SSH auth logs well, no extra field extraction was needed.
A count threshold of 5 is very low for a real environment and would cause false positives from normal typos. In a real SOC this would need tuning, or a time-based rate instead of a raw count.
Next step is to extend this into a proper correlation search that also checks if a login eventually succeeds after many failures, which is a stronger sign of a compromised account.


