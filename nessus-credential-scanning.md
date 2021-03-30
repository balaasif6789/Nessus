# Nessus - Credential scanning



One of the following conditions have not been met:

1. The **Windows Management Instrumentation \(WMI\)** service must be **enabled** on the target. For more information, see [https://technet.microsoft.com/en-us/library/cc180684.aspx](https://technet.microsoft.com/en-us/library/cc180684.aspx)
2. The **Remote Registry** service must be **enabled** on the target.
3. **File & Printer Sharing** must be **enabled** in the target's network configuration.
4. An **SMB account** must be used that has **local administrator rights** on the target. **Note:** A domain account can be used as long as that account is a local administrator on the devices being scanned.
5. **TCP ports 139 and 445** must be **open** between the Nessus Scanner and the target.
6. Ensure that there are no security policies are in place that blocks access to these services. This includes:
   * Windows Security Policies
   * Antivirus or Endpoint Security rules
   * IPS/IDS
7. The **default administrative shares** must be **enabled**.
   * These shares include:
     * IPC$
     * ADMIN$
     * C$
   * The setting that controls this is **AutoShareServer** which must be set to **1**.
   * Windows 10 has the ADMIN$ disabled by default.
   * For all other OS's, these shares are enabled by default and can cause other issues if disabled. For more information, see [http://support.microsoft.com/kb/842715/en-us](http://support.microsoft.com/kb/842715/en-us)

Resolution

### Testing from a Windows Host

Run all commands from an elevated Command prompt or PowerShell on a host in the same network as the target. Make sure this is not done on the target itself. If possible, use the scanner.

**Test WMI**

Test to see if WMI is working properly by[Verifying SMB via wbemtest](https://community.tenable.com/s/article/Verifying-SMB-via-wbemtest) 

**Anonymous IPC$ login test**

Test the IPC$ share without a username by using the following command. This command is similar to how Nessus checks the share.  
**Note:** Change **&lt;Target\_IP&gt;** to the target's IP address.

```text
net use \\<Target_IP>\ipc$ "" /user:""
```

For example:

![User-added image](https://community.tenable.com/servlet/rtaImage?eid=ka63a000000LFBp&feoid=00N60000003Ci0t&refid=0EMf2000000g6OI)If this returns "Failed to connect to the IPC$ share anonymously." then the following should be verified:

* Ensure SMB is set up correctly
* Double-check firewall settings

**SMB Log on Test**

This is how Nessus tests the credentials to make sure it has access to the system.

Run the following commands from an elevated command prompt.  
**Note:** Replace **&lt;username&gt;** and **&lt;password&gt;** with the credentials the scan is using. Also, change **&lt;Target\_IP&gt;** to the target's IP address.

```text
net use \\<Target_IP>\ipc$ /user:<username> <password>
net use \\<Target_IP>\admin$ /user:<username> <password>
```

![User-added image](https://community.tenable.com/servlet/rtaImage?eid=ka63a000000LFBp&feoid=00N60000003Ci0t&refid=0EMf2000000g6QE)These commands should return "The command completed successfully." If it does not, then:

* Check the credentials.
* Check the account has sufficient privileges.

**Remote Registry Test**

Run the following command to check if the remote registry is running.  
**Note:** Change **&lt;Target\_IP&gt;** to the target's IP address.

```text
reg query \\x.x.x.x\hklm
```

![User-added image](https://community.tenable.com/servlet/rtaImage?eid=ka63a000000LFBp&feoid=00N60000003Ci0t&refid=0EMf2000000g6ON)If this returns registry keys, the service is running and accessible.  If this returns "ERROR: The network path was not found." then the service is not running and must be enabled.

### Testing from a Linux Host

The program **smbclient** can be used as an alternative method of testing if the Nessus scanner is running on a Linux system that is scanning the Windows-based host. To install **smbclient**, run the following command as root:

```text
yum install samba-client
```

To test the IPC$ share without a username by using the following command. This command is similar to how Nessus checks the share.  
**Note:** Change **&lt;Target\_IP&gt;** to the target's IP address.

```text
smbclient ///x.x.x.x/IPC$ -U Administrator password
```

* If this returns "smb: \&gt;", then the credentials and permissions work.
* If this returns "session setup failed: NT\_STATUS\_LOGON\_FAILURE", then:
  * Check the credentials.
  * Check the account has sufficient privileges.

### ​​Still Having Issues

If you continue to have authentication issues after completing this process, open a case with Technical Support providing the following information:

* A detailed description of what has already been tried
* A Nessus DB. For more information, see [Collecting nessus.db Scan Results from Tenable Products](https://community.tenable.com/s/article/Collecting-nessus-db-Scan-Results-from-Tenable-Products)

