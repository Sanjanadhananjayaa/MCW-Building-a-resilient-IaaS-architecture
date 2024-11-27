## Exercise 1: Enable High Availability for the Contoso application

Duration: 60 minutes

The Contoso application has been deployed to the <inject key="Region" enableCopy="false" /> region. This initial deployment does not have any redundancy - it uses a single web VM, a single database VM, and a single domain controller VM.

In this exercise, you will convert this deployment into a highly-availability architecture by adding redundancy to each tier.

### Task 1: Deploy HA resources

In this task, you will deploy additional web, database and domain controller VMs.

A template will be used to save time. You will configure each tier in subsequent exercises in this lab.

1.  Copy the Link given below and paste it in in a browser insite the VM to launch the template deployment for the additional infrastructure components that will be used to enable high availability for the Contoso application. Log in to the Azure portal using your subscription credentials.
   
    [![Button to deploy the Contoso High Availability resource template to Azure.](https://aka.ms/deploytoazurebutton "Deploy the Contoso HA resources to Azure")](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FCloudLabs-MCW%2FMCW-Building-a-resilient-IaaS-architecture%2Fprod%2FHands-on%20lab%2FResources%2Ftemplates%2Fcontoso-iaas-ha.json)

2.  Complete the Custom deployment blade as follows:

    - Resource Group: **ContosoRG1 (1)** (existing)
    - Location: This should be same as the region of your *Contoso-RG1* resource group (2).

    Select **Review + Create (3)** and then **Create** to deploy resources.

    ![The custom deployment screen with ContosoRG1 as the resource group.](images/deploy02.png "Custom deployment")

3.  While you wait for the HA resources to deploy, take some time review the template contents. You can review the template by navigating to the **ContosoRG1** resource group, selecting **Deployments** in the resource group left-nav, and selecting any of the deployments, followed by **template**.

    ![Screenshot of the Azure portal showing the HA template contents.](images/Deployment02.png "Screenshot of the Azure portal showing the HA template contents.")

    Note that the template contains five child templates, containing the various resources required for:

    -   The ADVM2 virtual machine, which will be a second Domain Controller.
    -   The SQLVM2 virtual machine, which will be a second SQL Server.
    -   The WebVM2 virtual machine, which will be a second web server.
    -   Two load balancers, one for the web tier, and one for the SQL tier.
    -   The virtual network with the proper DNS configuration in place.

4.  You can check the HA resource deployment status by navigating to the **ContosoRG1** resource group, selecting **Deployments** in the resource group left-nav, and checking the status of the deployments. Make sure the deployment status is **Succeeded** for all templates before proceeding to the next task.

    ![Screenshot of the Azure portal showing the template deployment status 'Succeeded' for each template.](images/E1T1S4.png "Screenshot of the Azure portal showing the template deployment status Succeeded for each template")


### Task 2: Configure HA for the Domain Controller tier

In this task, you will reboot all the virtual machines to ensure they receive the updated DNS settings.

When using a domain controller VM in Azure, other VMs in the virtual network must be configured to use the domain controller as their DNS server. This is achieved by setting the DNS settings in the virtual network. These settings are then picked up by the VMs when they reboot or renew their DHCP lease.

The initial deployment included a first domain controller VM, **ADVM1**, with static private IP address **10.0.3.100**. The initial deployment also configured this IP address in the VNet DNS settings.

The HA resources template has added a second domain controller **ADVM2**, with static private IP address **10.0.3.101**. This server has already been promoted to be a domain controller using a CustomScriptExtension (you can review this script if you like, you'll find it linked from the ADVM2 deployment template). The template also updated the DNS setting on the virtual network to include the IP address of the second domain controller.

In this task, you will reboot all the servers to ensure they have the latest DNS settings.

1. Restart the **ADVM1** and **ADVM2** virtual machines in the **ContosoRG1** resource group so they pick up the new DNS server settings.
   
    ![Restart the VM.](images/restart1.png "Custom deployment")

2. Wait a minute or two for the domain controller VMs to fully boot, then restart the **WebVM1**, **WebVM2**, **SQLVM1** and **SQLVM2** virtual machines, so they also pick up the new DNS server settings.


### Task 3: Configure HA for the SQL Server tier

In this task, you will build a Windows Failover Cluster and configure SQL Always On Availability Groups to create a high-availability database tier.


1. In the Azure portal's left navigation, select **+ Create a resource**, then search for and select **Storage account**, followed by **Create**.

   ![Screenshot of the Azure portal showing the create storage account navigation.](images/storage-create.png "Create storage account blade")
   
2. Complete the **Create storage account** form using the following details:

    - **Resource group**: Use existing / ContosoRG1
    - **Storage account name:** contososqlwitness<inject key="DeploymentID" />
    - **Location**: Any location in your area that is **NOT** your Primary or Secondary site, for example **West US 2**
    - **Primary Service** : Azure Files.
    - **Performance**: Standard
    - **Replication**: Zone-redundant storage (ZRS).

    ![Fields in the Create storage account blade are set to the previously defined settings.](images/ha-storage1.png "Create storage account blade")

3.  Switch to the **Advanced** tab. Check the **Allow enabling anonymous access on individual containers** checkbox, change the **Minimum TLS version** to **Version 1.0** and Select **Review + create** and **Create**.

    ![The 'Advanced' tab of the Create storage account blade shows the minimum TLS version as 1.2](images/ha-tls1.png)

    > **Note:** To promote use of the latest and most secure standards, by default Azure storage accounts require TLS version 1.2. This storage account will be used as a Cloud Witness for our SQL Server cluster. SQL Server requires TLS version 1.0 for the Cloud Witness.

4.  Once the storage account is created, navigate to the storage account blade. Select **Access keys** under **Security + networking**. Select **Show keys** then copy the **storage account name** and the **first access key** and paste them into your text editor of choice - you will need these values later.

    ![In the Storage account Access keys section, the Storage account name and key1 are called out.](images/ha-storagekey1.png "Storage account section")

5.  Return to the Azure portal and navigate to the **ContosoSQLLBPrimary** load balancer blade. Select **Backend pools** and open **BackEndPool1**.

    ![Azure portal showing path to BackEndPool1 on ContosoSQLLBPrimary.](images/EX1-T3-S5.png "Backend pool")

6.  In the **BackendPool1** blade, select **+ Add**(1) and choose the two **SQL VMs**(2). Select **Add**(3) to close. Select **Save**(4) to add these SQL VMs to **BackEndPool1**.

    ![Azure portal showing SQLVM1 and SQLVM2 being added to the backend pool](images/EX1-T3-S6.png "Backend pool VMs")

    > **Note:** The load-balancing rule in the load balancer has been created with **Floating IP (direct server return)** enabled. This is important when using the Azure load balancer for SQL Server AlwaysOn Availability Groups.

7.  From the Azure portal, navigate to the **SQLVM1** virtual machine, select **Connect**, then choose **Bastion**. Connect to the machine using the following credentials:

    - **Username**: `demouser@contoso.com`
    - **Password**: `Demo!pass123`
    
    ![Azure portal showing connection to SQLVM1 using Bastion.](images/EX1-T3-S7.png)
    
    > **Note:** When using Azure Bastion to connect to a VM using domain credentials, the username must be specified in the format `user@domain-fqdn`, and **not** in the format `domain\user`.

    ![Azure portal showing connection to SQLVM1 using Bastion.](images1/E1T3S7.png "Azure Bastion")
   

8.  On **SQLVM1**, select **Start** and then choose **Windows PowerShell ISE**.

    ![Screenshot of the Windows PowerShell ISE icon.](images/image71.png "Windows PowerShell ISE icon")

9.  Copy and paste  the following command into PowerShell ISE and execute it. This will create the Windows Failover Cluster and add all the SQL VMs as nodes in the cluster. It will also assign a static IP address of **10.0.2.99** to the new Cluster named **AOGCLUSTER**.

    ```PowerShell
    New-Cluster -Name AOGCLUSTER -Node SQLVM1,SQLVM2 -StaticAddress 10.0.2.99
    ```

    ![In the PowerShell ISE window, a call out points to AOGCLUSTER.](images1/E1T3S9.png "PowerShell ISE window")

    >**Note:** It is possible to use a wizard for this task, but the resulting cluster will require additional configuration to set the static IP address in Azure.

10. After the cluster has been created, select **Start** and then **Windows Administrative Tools**. Locate and open the **Failover Cluster Manager**.

    ![Screenshot of the Failover Cluster Manager shortcut icon.](images/image154.png "Failover Cluster Manager shortcut icon")

11. When the cluster opens, select **Nodes** and the SQL Server VMs will show as nodes of the cluster and show their status as **Up**.

    ![In Failover Cluster Manager, Nodes is selected in the tree view, and two nodes display in the details pane.](images1/E1T3S11.png "Failover Cluster Manager")

12. If you select **Roles**, you will notice that currently, there aren't any roles assigned to the cluster.

    ![In Failover Cluster Manager, Roles is selected in the tree view, and zero roles display in the details pane.](images1/E1T3S12.png "Failover Cluster Manager")

13. Select Networks, and you will see **Cluster Network 1** with status **Up**. If you navigate to the network, you will see the IP address space, and on the lower tab you can select **Network Connections** and review the nodes.

    ![In Failover Cluster Manager, Networks is selected in the tree view, and the network displays in the details pane.](images1/E1T3S13.png "Failover Cluster Manager")

14. Right-click **AOGCLUSTER,** then select **More Actions**, **Configure Cluster Quorum Settings**.

    ![In Failover Cluster Manager, a call out says to right-click the cluster name in the tree view, then select More Actions, and then select Configure Cluster Quorum Settings.](images/image159.png "Failover Cluster Manager")

15. On the **Configure Cluster Quorum Wizard** select **Next**, then choose **Select the quorum witness**. Then, select **Next** again.

    ![On the Select Quorum Configuration Option screen of the Configure Cluster Quorum Wizard, the radio button for Select the quorum witness is selected.](images/image160.png "Configure Cluster Quorum Wizard")

16. Select **Configure a cloud witness** and **Next**.

    ![On the Select Quorum Witness screen, the radio button for Configure a cloud witness is selected.](images/image161.png "Select Quorum Witness screen")

17. Copy the **storage account name** and **storage account key** values which you noted earlier and paste them into their respective fields on the form. Leave the Azure Service endpoint as configured. Then, select **Next**.

    ![Fields on the Configure cloud witness screen are set to the previously defined settings.](images1/E1T3S17.png "Configure cloud witness screen")

18. Select **Next** on the Confirmation screen.

    ![On the Confirmation screen, the Next button is selected.](images/image163.png "Confirmation screen")

19. Select **Finish**.

    ![Finish is selected on the Summary screen.](images/image164.png "Summary screen")

20. Select the name of the Cluster again and the **Cloud Witness** should now appear in the **Cluster Resources**. It's important to always use a third data center, in your case here a third Azure Region to be your Cloud Witness.

    ![Under Cluster Core Resources, the Cloud Witness status is Online.](images/image165.png "Cluster Core Resources section")

21. Select **Start** and launch **SQL Server 2017 Configuration Manager**.

    ![Screenshot of the SQL Server 2017 Configuration Manager option on the Start menu.](images/image166.png "SQL Server 2017 Configuration Manager option")

22. Select **SQL Server Services**, then right-click **SQL Server (MSSQLSERVER)** and select **Properties**.

    ![In SQL Server 2017 Configuration Manager, in the left pane, SQL Server Services is selected. In the right pane, SQL Server (MSSQLSERVER) is selected, and from its right-click menu, Properties is selected.](images/image167.png "SQL Server 2017 Configuration Manager")

23. Select the **AlwaysOn High Availability** tab and check the box for **Enable Always OnAvailability Groups**. Select **Apply** and then select **OK** on the message that notifies you that changes won't take effect until after the server is restarted.

    ![In the SQL Server Properties dialog box, on the AlwaysOn High Availability tab, the Enable Always OnAvailability Groups checkbox is checked and the Apply button is selected.](images/image168.png "SQL Server Properties dialog box")

    ![A pop-up warns that any changes made will not take effect until the service stops and restarts. The OK button is selected.](images/image169.png "Warning pop-up")

24. On the **Log On** tab, change the service account to `contoso\demouser` with the password `Demo!pass123`. Select **OK** to accept the changes, and then select **Yes** to confirm the restart of the server.

    ![In the SQL Server Properties dialog box, on the Log On tab, fields are set to the previously defined settings. The OK button is selected.](images1/E1T3S24.png "SQL Server Properties dialog box")

    ![A pop-up asks you to confirm that you want to make the changes and restart the service. The Yes button is selected.](images/image171.png "Confirm Account Change pop-up")

25. Return to the Azure portal and open a new Azure Bastion session to **SQLVM2**. Launch **SQL Server 2017 Configuration Manager** and repeat the steps from 21 to 24 above to **Enable SQL AlwaysOn** and change the **Log On** username. Make sure that you have restarted the SQL Service.

26. Return to the Azure portal and open a second Azure Bastion session to **SQLVM2**. This time use `demouser` as the username instead of `demouser@contoso.com` and       use **Password**: `Demo!pass123`

27. Launch SQL Server Management Studio, a new dialog box will open, ensure that **Trust server certificate** is selected and click on **Connect** to sign on to **SQLVM2**. 

    > **Note**: The username for your lab should show **SQLVM2\demouser**.

    ![Screenshot of the Connect to Server dialog box.](images1/E1T3S27.png "Connect to Server dialog box")
    
29. And then expand **Security** and then **Logins**. You'll notice that only `SQLVM2\demouser` is listed.

    ![In SQL Server management studio, SQLVM2 is expanded, then Security is expanded, then Login is expanded. Only the SQLVM2\demouser account is seen.](images1/E1T3S28.png)

30. Right-click on **Logins** and then select **New Login...**

    ![The dialog box from the right-click on Logins is shown with an option to select New Login.](images1/E1T3S29.png)

31. In **Login name:** type **contoso\demouser**, then select **Server Roles**.

    ![The Login-New dialog box is displayed. In the Login name: box, the username contoso\demouser has been typed in. From here, it shows you selected the Server Roles tab in the left side navigation.](images1/E1T3S30.png)

32. Check the box for **sysadmin** and select **OK**.

    ![The Server Roles tab is shown in the Login - New dialog box. In this dialog box, public remains checked, and a check is added to the sysadmin option.](images1/E1T3S31.png)

26. Return to the Azure portal and open a second Azure Bastion session to **SQLVM1**. This time use `demouser` as the username instead of `demouser@contoso.com` and       use **Password**: `Demo!pass123`

27. Launch SQL Server Management Studio, a new dialog box will open, ensure that **Trust server certificate** is selected and click on **Connect** to sign on to **SQLVM1**. 

    > **Note**: The username for your lab should show **SQLVM1\demouser**.

    ![Screenshot of the Connect to Server dialog box.](images1/sqlvm1.png "Connect to Server dialog box")
    
29. And then expand **Security** and then **Logins**. You'll notice that only `SQLVM1\demouser` is listed.

    ![In SQL Server management studio, SQLVM2 is expanded, then Security is expanded, then Login is expanded. Only the SQLVM2\demouser account is seen.](images1/sqlvm2.png)

30. Right-click on **Logins** and then select **New Login...**

     ![The dialog box from the right-click on Logins is shown with an option to select New Login.](images1/sqlvm3.png)

31. In **Login name:**, type **contoso\demouser**, then select **Server Roles**.

    ![The Login-New dialog box is displayed. In the Login name: box, the username contoso\demouser has been typed in. From here, it shows you selected the Server Roles tab in the left side navigation.](images1/E1T3S30.png)

32. Check the box for **sysadmin** and select **OK**. Close the bastion session.

    ![The Server Roles tab is shown in the Login - New dialog box. In this dialog box, public remains checked, and a check is added to the sysadmin option.](images1/E1T3S31.png)

33. Return to your session with **SQLVM1**. Use the Start menu to launch **Microsoft SQL Server Management Studio 20** and connect to the local instance of SQL Server. (Located in the Microsoft SQL Server Tools folder).

    ![Screenshot of Microsoft SQL Server Management Studio 18 on the Start menu.](images/image172.png "Microsoft SQL Server Management Studio 18")

34. Select **Trust server certificate** is selected and click on **Connect** to sign on to **SQLVM1**. **Note**: The username for your lab should show **CONTOSO\demouser**.

    ![Screenshot of the Connect to Server dialog box.](images1/E1T3S33.png "Connect to Server dialog box")
    
1. Expand Databases and verify that the **ContosoInsurance** is present.  

    > **Note:** Skip on to step-49, if ContosoInsurance is already present.

    ![.](images/upd-1.png)
    
1. Right-click on **Databases (1)** and select **New Database (2)**.   

    ![.](images/upd-2.png)

1. On the New Database window, enter **ContosoInsurance (1)** as Database name and then click on **Ok (2)**.

    ![.](images/upd-3.png)
    
1.  Right-click on **ContosoInsurance (1)** and select **Tasks (2)**, and then click on **Back Up... (3)**.

    ![.](images/upd-4.png)

1. On the Back Up Database - Contosolnsurance window, click on **Ok**.

    ![.](images/upd-5.png)

1. When prompted with The backup of database 'Contosolnsurance' completed successfully window, click on **Ok**.

    ![.](images/upd-6.png)

1. Click on **New Query**.

    ![.](images/upd-7.png)

1. Copy and **paste (1)** the following query and then click on **Execute (2)**.

    ```
        USE master;
        GO
        GRANT ALTER ANY AVAILABILITY GROUP TO [NT AUTHORITY\SYSTEM]
        GO
        GRANT CONNECT SQL TO [NT AUTHORITY\SYSTEM]
        GO
        GRANT VIEW SERVER STATE TO [NT AUTHORITY\SYSTEM]
        GO
    
    ```    

    ![.](images/upd-8.png)

1. Wait until the query runs successfully.

    ![.](images/upd-9.png)
 

35. Right-click **Always On High Availability**, then select **New Availability Group Wizard**.

    ![In Object Explorer, AlwaysOn High Availability is selected, and from its right-click menu, New Availability Group Wizard is selected.](images/image174.png "SQ Server Management Studio, Object Explorer")

36. Select **Next** on the Wizard.

    ![On the New Availability Group Wizard begin page, Next is selected.](images/p50upd.png "New Availability Group Wizard ")

37. Provide the name **BCDRAOG** for the **Availability group name**, then select **Next**.

    ![The Specify availability group options form displays the previous availability group name.](images/image176.png "Specify availability group options page")

38. Select the **ContosoInsurance Database**, then select **Next**.

    ![The ContosoInsurance database is selected from the user databases list.](images/image177.png "Select Databases page")

39. On the **Specify Replicas** screen next to **SQLVM1**, select **Automatic Failover**.

    ![On the Replicas tab, for SQLVM1, the checkbox for Automatic Failover (Up to 3) is selected.](images/image178.png "Specify Replicas screen")

40. Select **Add Replica**.

    ![Screenshot of the Add replica button.](images/image179.png "Add replica button")

41. On the **Connect to Server** dialog box enter the Server Name of **SQLVM2** and select **Connect**. **Note**: The username for your lab should show **CONTOSO\demouser**.

    ![Screenshot of the Connect to Server dialog box for SQLVM2.](images1/E1T3S40.png "Connect to Server dialog box")

42. For **SQLVM2**, select Automatic Failover and Availability Mode of Synchronous commit.

    ![On the Replicas tab, for SQLVM2, the checkbox for Automatic Failover (Up to 3) is selected, and availability mode is set to synchronous commit.](images/image181.png "Specify Replicas Screen")

43. Select **Endpoints** and review these that the wizard has created.

    ![On the Endpoints tab, the three servers are listed.](images1/E1T3S42.png "Specify Endpoints screen")

44. Next, select **Listener**. Then, select the **Create an availability group listener**.

    ![On the Listener tab, the radio button for Create an availability group listener is selected.](images/image185.png "Specify Listener screen")

45. Add the following details:

    - **Listener DNS Name**: BCDRAOG
    - **Port**: 1433
    - **Network Mode**: Static IP

    ![Fields for the Listener details are set to the previously defined settings.](images/image186.png "Listener details")

46. Next, select **Add**.

    ![The Add button is selected beneath an empty subnet table.](images/image187.png "Add button")

47. Select the Subnet of **10.0.2.0/24** and then add IPv4 **10.0.2.100** and select **OK**. This is the IP address of the Internal Load Balancer that is in front of the **SQLVM1** and **SQLVM2** in the **Data** subnet running in the **Primary** Site.

    ![The Add IP Address dialog box fields are set to the previously defined settings.](images/image188.png "Add IP Address dialog box")

48. Select **Next**.

    ![On the Listener tab, the Next button is selected.](images1/E1T3S47.png "Next button")

49. On the **Select Initial Data Synchronization** screen, make sure that **Automatic seeding** is selected and select **Next**.

    ![On the Select Initial Data Synchronization screen, the radio button for Automatic seeding is selected. The Next button is selected at the bottom of the form.](images/image192.png "Select Initial Data Synchronization screen")

50. On the **Validation** screen, you should see all green. Select **Next**.

    ![The Validation screen displays a list of everything it is checking, and the results for each, which all display success. The Next button is selected.](images1/E1T3S49.png "Validation screen")

51. On the Summary page select **Finish**.

    ![On the Summary page, the Finish button is selected.](images/image194.png "Summary page")

52. Once the AOG is built, check each task was successful and select **Close**.

    ![On the New Availability Group Results page, a message says the wizard has completed successfully, and results for all steps is success. The Close button is selected.](images1/E1T3S51.png "New Availability Group Results page")

53. Move back to **SQL Management Studio** on **SQLVM1** and expand the **Always On High Availability** item in the tree view. Under Availability Groups, expand the **BCDRAOG (Primary)** item.

    ![In SQL Management Studio, Always On High Availability is expanded in the tree view.](images1/E1T3S52.png "SQL Management Studio")

54. Right-click **BCDRAOG (Primary)** and then select **Show Dashboard**. You should see that all the nodes have been added and are now "Green".

    ![Screenshot of the BCDRAOG Dashboard indicating the status of all SQL Server VMs as healthy.](images1/E1T3S53.png "BCDRAOG Dashboard")

55. Next, select **Connect** and then **Database Engine** in SQL Management Studio.

    ![Connect / Database Engine is selected in Object Explorer.](images/image200.png "Object Explorer")

56. Enter **BCDRAOG** as the Server Name. This will be connected to the listener of the group that you created. **Note**: The username for your lab should show **CONTOSO\demouser**.

    ![In the Connect to Server Dialog box, Server name is BCDRAOG, and the connect button is selected.](images1/E1T3S55.png "Connect to Server Dialog box")

57. Once connected to the **BCDRAOG**, you can select **Databases** and will be able to see the database there. Notice that you have no knowledge directly of which server this is running on.

    ![A call out points to ContosoInsurance (Synchronized) in SQL Management Studio.](images/image202.png "SQL Management Studio")

58. Move back to **PowerShell ISE** on **SQLVM1**. Open a new file, paste in the following script, and select the **Play** button. This will update the Failover cluster with the IP address of the Listener that you created for the AOG.

    ```Powershell
    $ClusterNetworkName = "Cluster Network 1"
    $IPResourceName = "BCDRAOG_10.0.2.100"
    $ILBIP = "10.0.2.100"
    Import-Module FailoverClusters
    Get-ClusterResource $IPResourceName | Set-ClusterParameter -Multiple @{"Address"="$ILBIP";"ProbePort"="59999";"SubnetMask"="255.255.255.255";"Network"="$ClusterNetworkName";"EnableDhcp"=0}
    Stop-ClusterResource -Name $IPResourceName
    Start-ClusterResource -Name "BCDRAOG"
    ```

    ![In the Windows PowerShell ISE window, the play button is selected. The script from the lab guide has been executed.](images1/E1T3S57.png "Windows PowerShell ISE window")

59. Move back to SQL Management Studio and select **Connect** and then **Database Engine**.

    ![In Object Explorer, Connect / Database Engine is selected.](images/image200.png "Object Explorer")

60. This time, put the following into the IP address of the Internal Load balancer of the **Primary** Site AOG Load Balancer: **10.0.2.100**. You again will be able to connect to the server which is up and running as the master. **Note**: The username for your lab should show **CONTOSO\demouser**.

    ![Fields in the Connect to Server dialog box are set to the previously defined settings.](images1/E1T3S59.png "Connect to Server dialog box")

61. Once connected to **10.0.2.100**, you can select **Databases** and will be able to see the database there. Notice that you have no knowledge directly of which server this is running on.

    ![A call out points to the Databases folder in Object Explorer.](images1/E1T3S60.png "Object Explorer")

    > **Note:** It could take a minute to connect the first time as this is going through the Azure Internal Load Balancer.

62. Move back to Failover Cluster Manager on **SQLVM1**, and you can review the IP Addresses that were added by selecting Roles and **BCDRAOG** and viewing the Resources. Notice how the **10.0.2.100** is Online.

    ![In the Failover Cluster Manager tree view, Roles is selected. Under Roles, BCDRAOG is selected, and details of the role display.](images1/E1T3S61.png "Failover Cluster Manager")

You have now successfully set up the SQL Server VMs to use Always On Availability Groups with a Cloud Witness storage account located in another region.

### Task 4: Configure HA for the Web tier

In this task, you will configure a high-availability web tier. This comprises two web server VMs which you will locate behind an Azure load balancer. You will also configure the VMs to access the database using the Always On Availability Group endpoint you created earlier.

1.  In the Azure portal, navigate to **WebVM1**, select **Connect** followed by **Bastion**, and connect to the VM using the following credentials:

    - **Username**: `demouser@contoso.com`
    - **Password**: `Demo!pass123`

2.  In **WebVM1**, open Windows Explorer, navigate to **C:\inetpub\wwwroot** and open the **Web.config** file using Notepad.

    > **Note**: If the **Web.config** change does not run, go to **Start**, **Run** and type **iisreset /restart** from command line.

3.  In the **Web.config** file, locate the **\<ConnectionStrings\>** element and replace **SQLVM1** with **BCDRAOG** in the data source. Remember to **Save** the file.

    ![Notepad is editing the Web.config file. The data source is updated to BCDRAOG.contoso.com.](images1/E1T4S3.png "Web.config file")

4.  Repeat the above steps to make the same change on **WebVM2**.

5.  Return to the Azure portal and navigate to the **ContosoWebLBPrimary** load balancer blade. Select **Backend pools** and open **BackEndPool1**.

    ![Azure portal showing path to BackEndPool1 on ContosoWebLBPrimary.](images1/E1T4S5.png "Backend pool select path")

6.  In the **BackendPool1** blade, select **VNet1 (ContosoRG1)** as the Virtual network. Then select **+ Add** and select the two web VMs. Select **Save**.

    ![Azure portal showing WebVM1 and WebVM2 being added to the backend pool.](images1/E1T4S6.1.png "Backend pool VMs")

7.  You will now check that the Contoso sample application is working when accessed through the load-balancer. In the Azure portal, navigate to the **ContosoWebLBPrimaryIP** resource. This is the public IP address attached to the web tier load balancer front end. Copy the **DNS name** to the clipboard.

    ![Azure portal showing web load balancer public IP, with DNS name highlighted.](images1/E1T4S7.png "Public IP DNS name")

8.  Open a new browser tab and paste the DNS name. The Contoso Insurance sample app is shown. 

    ![Browser screenshot showing the contoso sample application. The address bar shows the DNS name copied earlier.](images1/E1T4S8.png "Contoso app")

    > **Note:** You will test the HA capabilities later in the lab.