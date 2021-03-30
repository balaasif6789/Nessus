# Nessus - Extracting passwords from memory



There have been a number of articles in the past regarding how information security controls can create vulnerabilities themselves, including a good one from Microsoft, “[How InfoSec Security Controls Create Vulnerability](https://web.archive.org/web/20170115191215/https://blogs.technet.microsoft.com/johnla/2016/02/20/how-infosec-security-controls-create-vulnerability/)”. Network vulnerability scanners, like Tenable’s Nessus, are a great example of this issue, especially when they are configured to perform credentialed scans throughout your organization. Credentialed scans require privileged accounts to be comprehensive so it is more common than not that organizations end up using “Domain Admin” types of accounts.

If you don’t properly secure your Nessus system and an attacker is able to gain privileged access to it, the saved credentials within Nessus can easily be recovered. Nessus protects these saved credentials by storing them in an encrypted SQLite database named “policies.db” and there are different “policies.db” files for each Nessus user. They are encrypted with a randomly generated key located in a file named “master.key” \(also an encrypted SQLite database\). You also cannot view the stored password\(s\) through the web interface, however, usernames can be viewed.

These are the typical locations of the “master.key” file:

* Windows: C:\ProgramData\Tenable\Nessus\nessus
* Linux: /opt/nessus/var/nessus

These are the typical locations of the “policies.db” file\(s\):

* Windows: C:\ProgramData\Tenable\Nessus\nessus\users\&lt;username&gt;
* Linux: /opt/nessus/var/nessus/users/&lt;username&gt;

![](https://web.archive.org/web/20170115191215im_/https://www.appsecconsulting.com/documents/uploads/tlee1.jpg)

_Figure 1 – Nessus web interface view of stored Windows credentials in a policy._

We could go attempt to reverse engineer the process of how Nessus decrypts the “master.key” and “policies.db” files \(perhaps I will save this for a follow-up article\), however, that could take some time and there is an easier technique that Digital Forensics & Incident Response \(DFIR\) folks use that we can leverage by dumping the memory of the Nessus process and searching it for clear text strings.

#### Steps for a Windows-based Nessus system:

1. Copy **procdump.exe** and **strings.exe** from Microsoft’s [Sysinternals Suite](https://web.archive.org/web/20170115191215/https://technet.microsoft.com/en-us/sysinternals/bb842062.aspx) onto the Nessus system through your shell or other means.
2. Log into the Nessus web interface and start a scan against any system using the profile\(s\) that have the saved credentials
   * Note: If you have privileged access to the Nessus system, but don’t have credentials to log into the web interface, you can use “C:\Program Files\Tenable\Nessus\nessuscli.exe adduser \[username\]” to create yourself a new Nessus admin user.
3. Get the PID of the nessusd.exe process with the following command:
   * **tasklist \| findstr nessusd**

![](https://web.archive.org/web/20170115191215im_/https://www.appsecconsulting.com/documents/uploads/tlee2.jpg)

_Figure 2 – The nessusd.exe process has a PID of 3700._

1. Run the following command to dump the memory of the nessusd.exe process, replacing “\[pid\]” with the PID of the nessusd.exe process.
   * Note: You must run ProcDump as an administrator-level account. Additionally, ProcDump saves the memory dump file in the same directory that it is run in with a default name of: &lt;processname&gt;\_&lt;datetime&gt;.dmp.
   * **procdump.exe -accepteula -ma \[pid\]**

![](https://web.archive.org/web/20170115191215im_/https://www.appsecconsulting.com/documents/uploads/tlee3.jpg)

_Figure 3 – Running ProcDump on the nessusd.exe process with a PID of 3700._

1. Then run the following command to extract the strings from the newly created memory dump file, replacing “\[memdump\_file\]” with the name of the memory dump file created by ProcDump and save the strings to a file named “nessusd-strings.txt”:
   * **strings.exe -accepteula \[memdump\_file\] &gt; nessusd-strings.txt**
2. Finally, run the following commands to search through the “nessus-strings.txt” file and find the saved credentials, even saved SSH keys.
   * Note: The search terms are case-sensitive. Ensure you enter them as shown here. Additionally, there may be random stray characters appended to the username and/or password entries since this is pulled from memory.
   * **findstr /C:"Login configurations" nessusd-strings.txt**
   * **findstr /C:"SSH settings" nessusd-strings.txt**

![](https://web.archive.org/web/20170115191215im_/https://www.appsecconsulting.com/documents/uploads/tlee4.jpg)

_Figure 4 – Windows SMB credentials in the nessusd-strings.txt file._

![](https://web.archive.org/web/20170115191215im_/https://www.appsecconsulting.com/documents/uploads/tlee5.jpg)

_Figure 5 – Stored SSH key \(redacted\) in nessusd-strings.txt_

#### Steps for a Linux-based Nessus system:

1. Install the “gcore” application on the Nessus system if it isn’t already installed.
2. Log into the Nessus web interface and start a scan against any system using the profile\(s\) that have the saved credentials.
   * Note: If you have privileged access to the Nessus system, but don’t have credentials to log into the web interface, you can use “/opt/nessus/sbin/nessuscli adduser \[username\]” to create yourself a new Nessus admin user.
3. Get the PID of the nessusd process with the following command:
   * **\# ps aux \| grep nessusd**

![](https://web.archive.org/web/20170115191215im_/https://www.appsecconsulting.com/documents/uploads/tlee6.jpg)

_Figure 6 – The nessusd process has a PID of 3921._

1. Run the following command to dump the memory of the nessusd process, replacing “\[/path/to/file\]” with the location and filename of where you want to save your dump file and “\[pid\]” with the PID of the nessusd process.
   * **\# gcore -o \[/path/to/file\] \[pid\]**

![](https://web.archive.org/web/20170115191215im_/https://www.appsecconsulting.com/documents/uploads/tlee7.jpg)

_Figure 7 – Memory dump of the nessusd process saved to /root/nessusd.dmp.3921._

1. Then run the following command to extract the strings from the newly created memory dump file, replacing “\[memdump\_file\]” with the name of the memory dump file created by gcore and save the strings to a file named “nessusd-strings.txt”:
   * **\# strings \[memdump\_file\] &gt; nessusd-strings.txt**
2. Finally, run the following commands to search through the “nessus-strings.txt” file and find the saved credentials, even saved SSH keys.
   * Note: The search terms are case-sensitive. Ensure you enter them as shown here. Additionally, there may be random characters appended to the username and/or password entries since this is pulled from memory. In this instance, there is an additional character at the end of both the username and password.
   * **\# grep "Login configurations" nessusd-strings.txt**
   * **\# grep "SSH settings” nessusd-strings.txt**

![](https://web.archive.org/web/20170115191215im_/https://www.appsecconsulting.com/documents/uploads/tlee8.jpg)

_Figure 8 – Windows SMB credentials in the nessusd.dmp.3921 file._

Alternatively, you can copy the memory dump file off of the Nessus system to your attack box to perform steps 5 and 6 and to make it easier to process. You could also just copy the “policies.db” and “master.key” files to your own local Nessus instance and perform all the steps locally.

#### Mitigation

The following are a few general tips to help protect your Nessus \(or other InfoSec control\) system:

* Utilize secure network segmentation to isolate the system to only allow the required destination ports and source IPs for authorized users.
* Patch, patch, patch! Keep the OS and any installed software up to date.
* Ensure all user accounts utilize long passphrases.
* Ensure all user accounts used on the system are different from the user’s normal account.
* Utilize multi-factor authentication.
* Harden the system utilizing industry best practices \(e.g. [CIS Secure Configuration Benchmarks](https://web.archive.org/web/20170115191215/https://benchmarks.cisecurity.org/downloads/latest/)\).
* Log at the system and network-level and alert on abnormal usage.

