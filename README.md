deliverance-macos-cis
Hardening a macOS 26 Tahoe endpoint from 62% to 100% on the CIS Apple macOS 26.0 Tahoe Benchmark v1.0.0, measured and verified with Wazuh Security Configuration Assessment (SCA).

By @alex_mh · TrifoldPathSec · October 2026

Passed	Failed	Not applicable	Score
Before	40	24	2	62%
After	66	0	0	100%
Lab setup
flowchart LR
    subgraph Host["MacBook Pro (macOS 26 Tahoe)"]
        A["Wazuh agent v4.14.8<br/>SCA module"]
        subgraph UTM["UTM hypervisor"]
            S["Ubuntu 26.04.1 VM<br/>Wazuh server + dashboard"]
        end
    end
    A -- "SCA scan results" --> S
    S -- "Dashboard: score + failed checks" --> U["Analyst"]
Endpoint: physical MacBook Pro running macOS 26 Tahoe with the Wazuh agent.
SIEM: Wazuh server and dashboard on an Ubuntu VM in UTM, on the same machine.
Policy: CIS_Apple_macOS_26.0_Tahoe_Benchmark_v1.0.0, 66 checks.
Method
Filter the SCA policy view to result=failed and expand each check to read its exact rule.
Apply the CIS remediation on the endpoint, then verify it with the same command the check runs.
Restart the agent (sudo /Library/Ossec/bin/wazuh-control restart), or reboot where a setting needs it, to trigger a fresh scan.
Refresh the dashboard and repeat until nothing fails.
Score progression across four scans: 62% → 70% → 95% → 100%.

Remediated controls
25 controls were remediated: 23 of the original failures (one more passed as a side effect of early fixes), plus 41041 and 41043, which moved from Not applicable to Failed once auditing was turned on.

Network and services
ID	Control	Change	Why it matters
41006	Firewall stealth mode	socketfilterfw --setglobalstate on --setstealthmode on	The Mac stops answering probes such as ping, so it is harder to find on a network
41044	Bonjour advertising disabled	NoMulticastAdvertisements = true (mDNSResponder)	Stops the Mac broadcasting its services to everyone on the local network
41046	NFS server disabled	launchctl disable system/com.apple.nfsd	Removes an unused file-sharing service from the attack surface
Data protection and privacy
ID	Control	Change	Why it matters
41028	FileVault enabled	fdesetup enable + dontAllowFDEDisable	Encrypts the disk, so a lost or stolen laptop does not expose data
41030	Help Apple Improve Search off	Search Queries Data Sharing Status = 2	Search queries are not sent to Apple
41039	On-device dictation only	forceOnDeviceOnlyDictation = true	Spoken input is processed locally, not in the cloud
41031	Power Nap off	pmset -a powernap 0	The Mac does not wake to sync and run network activity while asleep
Authentication and access
ID	Control	Change	Why it matters
41050	Lockout threshold	5 attempts, 15-minute auto-unlock	Stops online password guessing
41051	Minimum length	15 characters	Length is the strongest defence against cracking
41052–41055	Complexity	Letters, numbers, symbols, mixed case	Widens the search space for brute force
41056	Password age	Expires every 365 days	Limits how long a leaked password stays useful
41057	Password history	Last 24 passwords cannot be reused	Prevents cycling back to a known password
41058	sudo timeout 0	Defaults timestamp_timeout=0	A stolen terminal session cannot reuse cached sudo rights
41060	root disabled	dsenableroot -d; root shell /usr/bin/false	Removes an all-powerful account that attackers target
Login window
ID	Control	Change	Why it matters
41033	Custom login message	LoginwindowText	Shows an authorized-use notice before sign-in
41034	Name and password fields	SHOWFULLNAME = true	Does not reveal valid usernames at the login screen
41062	Policy banner	/Library/Security/PolicyBanner.txt	A legal banner that must be acknowledged
Logging and auditing
ID	Control	Change	Why it matters
41040	Security auditing on	Load com.apple.auditd	Records security events for investigation
41041	Audit flags	flags:-fm,ad,-ex,aa,-fr,lo,-fw	Captures logins, admin actions and failed file access
41043	Audit retention	expire-after:60d OR 5G	Keeps evidence long enough to investigate an incident
41042	install.log retention	ttl=365, no size cap	Keeps a year of software install history
41065	sudo logging	Defaults log_allowed	Every privileged command is logged
Lessons learned
Read the check, not just the title. 41060 kept failing after dsenableroot -d because the rule also requires root's shell to be /bin/false. Expanding the check in Wazuh showed the exact conditions.
Some commands fail quietly. nfsd disable printed an error about a missing /etc/exports and left the service enabled. launchctl disable system/com.apple.nfsd worked. Always verify after you remediate.
Fixes can surface new findings. Turning on auditing (41040) moved two checks from Not applicable to Failed, because the audit config they inspect now existed.
The scanner runs as root. Preference checks read the root context, so settings had to be written system-wide (/Library/Preferences), not just for the logged-in user.
Trade-offs
Control	Usability cost	When I would relax it
sudo timeout 0	A password prompt on every sudo	On a dedicated admin workstation with short screen-lock times
15-character complexity	Harder to remember passwords	With a password manager plus MFA, length alone (no complexity) follows NIST 800-63B
365-day expiry	Forced changes	NIST 800-63B advises against periodic expiry without evidence of compromise
Limitations
Not enforced like MDM. dontAllowFDEDisable and forceOnDeviceOnlyDictation are normally pushed by a configuration profile. Writing them locally passes the scan, but a user with admin rights could change them back.
macOS updates can reset settings, for example in /etc/asl and audit_control. Re-scan after every major update.
Rollback
Change	Undo
Password policy	sudo pwpolicy -clearaccountpolicies
sudo settings	sudo rm /etc/sudoers.d/10_cis
Login banner	sudo rm /Library/Security/PolicyBanner.txt
Name and password login	sudo defaults delete /Library/Preferences/com.apple.loginwindow SHOWFULLNAME
install.log retention	Restore /etc/asl/com.apple.install.bak
Audit settings	Restore /etc/security/audit_control.bak
Next steps
 Package all remediations as one idempotent script
 Build a .mobileconfig configuration profile so the settings are enforced, as an MDM would
 Add Wazuh file-integrity monitoring on /etc/sudoers.d/, /etc/security/audit_control and /Library/Preferences, and show an alert firing when a setting is reverted
 Map each control to NIST 800-53 and CIS Controls v8
Part of the TrifoldPathSec lab series. Lab-only: test on a machine you own before applying any of this to production.
