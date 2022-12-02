# This is my mashed together procedure for a Horizon App Volume Best Practice SQL Always On Cluster Installation

## **Prerequisites**:

* SQL Service Account ( Domain account ) Ex sql_svc
 + only needs sysadmin permissions in SQL, no domain or server permissions required
* Need Listener IP 
 + Ex: 192.168.1.76
* Need Windows Cluster IP
 + Ex: 192.168.1.75
* Create SQL Service cluster DNS name in Active Directory pointing to Listener IP
 + Ex: appvolumesdb -> 192.168.1.76
* Create Windows cluster DNS name in Active Directory pointing to Windows Cluster IP, ensure to check "Allow any authenticated user to update record"
 + Ex: sqlcluster -> 192.168.1.75
* Create a new OU for the SQL Servers that will be part of the cluster
 + The cluster computer account will need to have permissions to create/delete computer objects in this OU.
 + https://learn.microsoft.com/en-us/windows-server/failover-clustering/prestage-cluster-adds#grant-the-cno-permissions-to-the-ou
 + Ex: AppVolumesSql
* Create SQL Service Computer Object in Active Directory
 + Ex: sql
* Create Windows File Share for Database Replication (needs about 5MB of space), ensure the SQL Service Account has read/write access.
 + Ex: DatabaseReplication
* Firewall on all sql nodes
 + My example has 3 servers:  
  * sql1 (192.168.1.71)
  * sql2 (192.168.1.72)
  * sql3 (192.168.1.73)
 + Enable ports 1433,1496,5022
netsh advfirewall firewall add rule name = SQLPort dir = in protocol = tcp action = allow localport = 1433 remoteip = localsubnet profile = DOMAIN
netsh advfirewall firewall add rule name = SQLAvailGroupListenerPort dir = in protocol = tcp action = allow localport = 1496 remoteip = localsubnet profile = DOMAIN
netsh advfirewall firewall add rule name = SQLAvailGroupPort dir = in protocol = tcp action = allow localport = 5022 remoteip = localsubnet profile = DOMAIN
 + Create custom firewall rule to allow all traffic between nodes in the cluster
netsh advfirewall firewall add rule name = SQLServer1 dir = in remoteip = 192.168.1.71 profile = DOMAIN action = allow
netsh advfirewall firewall add rule name = SQLServer2 dir = in remoteip = 192.168.1.72 profile = DOMAIN action = allow
netsh advfirewall firewall add rule name = SQLServer3 dir = in remoteip = 192.168.1.73 profile = DOMAIN action = allow
* Add Windows Features
 + Failover Clustering
 + .NET 3.5 ( don't need HTTP Activation or Non-HTTP Activation )

## **Create Cluster**
+ Failover Cluster Manager
 * -> Validate Configuration
   -> Add Servers
   -> Run all tests -> Next -> Next -> View Report (If all looks good) -> Finish
 * -> This takes us to the Create Cluster Wizard
  + Enter Cluster DNS name
  + Next -> Finish
 * Configure Quorum
  + Right click on the cluster -> More Acitons -> Configure Cluster Quoruom Settings...
   * Select the quorum witness
   * Configure a file share witness
   * Browse -> Browse -> Type in a File Server hostname that you have admin writes on.  
     - We're creating the share via the wizard so we don't have a chicken and the egg permission issue
      - Click "New Shared Folder"
       * Share name Ex ClusterWitness
       * Local path -> Browse -> select "c$" and then click on "Make New Folder".  Call it something simliar to the share name.  
      - Click off of the new folder and then back onto the folder.  Click OK
       - Permissions - Use custom permissions
        - grant full access to your cluster computer account.  It should match the name of your "Windows cluster DNS name" with a dollar sign at the end.
         - Ex sqlcluster$

**Install SQL**
 - Configure Firewall - cmd or powershell:
   - netsh advfirewall firewall add rule name = SQLPort dir = in protocol = tcp action = allow localport = 1433 remoteip = localsubnet profile = DOMAIN
   - netsh advfirewall firewall add rule name = SQLAvailGroupListenerPort dir = in protocol = tcp action = allow localport = 1496 remoteip = localsubnet profile = DOMAIN
   - netsh advfirewall firewall add rule name = SQLAvailGroupPort dir = in protocol = tcp action = allow localport = 5022 remoteip = localsubnet profile = DOMAIN
 - Launch Installer and install SQL Management Tools on the first node (Requires a reboot...)
 - Install New SQL Stand-Alone Install on all nodes
   - Features:
     - Database Engine Services
     - SQL Server Replication
     - Client Tools Connectivity
   - Set both SQL Server Agent and SQL Server Database Engine to use the SQL Service Account and set both to Automatic startup
   - Databse Engine Configuration
     - Use Mixed Mode Auth
       - Set the SA password
       - Add either the current logged in user, or add an AD group that should have full access
     - Data Directories - use at a minimum another disk for logs and another disk for databases
     - Ensure TempDB "Number of files" matches the number of vcpus
     - Ensure MaxDOP "Maximum degree of parallelism" matches the number of vcpus
     - Memory - Up to user.  By default, SQL will use all RAM available.  If you want to guarntee some portion of memory to the OS, switch to Recommended and either use their "defaults" or edit the Max to be something reasonable.  If you switch to Recommended, make sure to check the "Click here to accept" box.


**Launch "Sql Server 2019 Configuration Manager"**
  - SQL Server Services -> SQL Server (MSSQLSERVER)
    - Right click -> Properties -> Always on Availability Groups - Check "Enable Always on Availability Groups"
    - Click Apply and go back to the "Log On" tab.  Click Restart
  - SQL Server Network Configuration -> Protocols for MSSQLSERVER
    - Enable Named Pipes
    - Restart SQL Server (MSSQLSERVER) and SQL Server Agent (MSSQLSERVER)


**Launch "Microsoft SQL Server Management Studio 18"**
  - Connect to each SQL Server
  - Expand Security and right click on Logins.  Choose "New Login..."
    - Click Search and find the SQL Service Account
    - Click on "Server Roles" and add sysadmin
  - Minimize the Security folder back to default so we can see more of the servers at once
  - On SQL1:
    - Right click on Databases and choose "New Database".
      - Name AppVolumes
      - For LogicalName AppVolumes:
        - Initial Size should be 3
        - Autogrowth should be By 1MB, Unlimited
      - For LogicalName AppVolumes_log:
        - Initial Size should be 1
        - Autogrowth should be By 10percent, Unlimited
    - Before adding new DB to Availability Group, run a backup in the query window:
      - "Backup database appvolumes to disk = 'c:\users\zac\Downloads\appvolumes_bak'"
    - Right click on Availability Groups -> New Availability Group
      - The name should match the "SQL Service Computer Object" that you created in Active Directory
      - Ensure Status has "Meets prerequisites".  If it does not, click the status and see what's wrong
      - "Specify Replicas"
        - "Replicas"
          - Click "Add Replica..." and add the other SQL nodes
            - Change all of their "Availability Mode" to "Synchronous commit"
            - Select Automatic Failover on all
            - For the databases with an "Initial Role" of "Secondary", set "Readable Secondary" to Yes
        - "Endpoints"
          - Ensure each has Port Number set to 5022
        - "Listener"
          - # this URL will be utilized as the hostname for the cluster SQL Connection String
          - Listener DNS Name should match "SQL Service cluster DNS name" from above Ex appvolumesdb
      - "Select Data Synchronization" 
        - Full database and log backup
          - Specify the path to "Windows File Share for Database Replication" you created above.  Ex DatabaseReplication
      - "Validation"
        - Ensure we have green checkmarks on all tests
          - If not, go back and fix what it shows you is wrong
    - After the Availability Group is created, in SQL Managment Studio, right click on the "(Primary)" Availability Group and choose "Show Dashboard"
      - Wait until all nodes are in the Synchronized "Synchonization State"
    - Test Failover
      - Right click on the current "(Primary)" in the Availability Groups and choose Failover...
