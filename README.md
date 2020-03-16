# Hi, Jonas here! 
Or as we say in the north of Germany: **"Moin Moin!"**<br>
I am a Microsoft Premier Field Engineer (PFE) based in Hamburg and a while back (years in fact) I was asked to analyze the update compliance status of a customers SCCM (ConfigMgr/MECM) environment. <br>
I used different reports to look for clients not installing the necessary updates, but it was time consuming and I was missing a general overview with some meanigful KPIs. I ended up with a comprehensive SQL query and an Excel sheet, but changed that to a SQL Server Reporting Services (SSRS) report and made that available to several departments in the organization later on.<br>
As mentioned before, it's been a while since I created the report and if I would start now it would be a PowerBI version or I would simply grab one of the PowerBI reports available right now, but since I still use the report and find it quite helpful, I decided to share that with the rest of the world.

# TL/DR
The following report should help you identify update problems within a specific collection and is designed to work well for a few thousand clients. If you have more then 10k systems, then the query might run long and you might need to improve it or run it not within business hours to show results.<br>
The installation guide for the custom update reporting can be found at the end of this post but you should at least start with the "Some key facts and prerequisites" section.<br>
If you're just looking for the SQL statement behind the report, copy the query from the "UpdatesSummary.rsd" file and use it in SQL directly. 

# Some key facts and prerequisites:
- The report is made to show the update compliance status of members of a collection or multiple collections no matter what type of systems are a member or which or how many updates are deployed to each of the systems.
- If you have a simple group of systems and deploy every needed update with one deployment, the deployment status might be enough, but if you have a more complex setup, you might want to see details based on a specific group of systems no matter if, how or how many updates are deployed to each system.
- The report will also count updates deployed as "available" and is not made to just focus on updates deployed as "required"
- The report consists of multiple KPIs to indicate the update compliance or update/client health state and should give you an overview from different viewpoints to help identify problematic systems or a flaw in your patch strategy. 
- The report will use data from the WMI class Win32_Quickfixengineering which needs to be enabled in the hardware inventory client settings. The class is only used to determine the last installation of A security update to identify systems which seem to be fine, but have never installed anything.
- The report is also using the LastLogonTimeStamp from AD System Discovery to visually show systems which have not logged on to the domain in a while and which might be disposed already and could be deleted from the SCCM database. If you don't use AD system discovery the report will show all systems of the specified collection as not compliant in the pie chart "Last ADDS logon" (12).
  - AD system discovery is no hard requirement to run the report
- The report does not show historical data and will always show the current status. So if you change a deployment in the middle of the month, the compliance percentage will drop almost immediately
- I have defined "compliant" to be a system which has:
   - all the updates installed which are deployed
   - the last security update installation in Win32_Quickfixengineering was in the current month (not neccesarily the monthly security rollup, just one security update)
- The update report has multiple sub-reports to drill further down and each report will use the same dataset
- The SQL query of the dataset is made to filter out Defender Update Deployments, because they normally will be changed every x hours and could interfere with the overall compliance state and should be monitored with other reports. 
- The 2nd Level sub-report per system will also show Defender updates, even if they are filtered out on the dashbaord
- The SQL query might run long in bigger environments (<10.000 clients) depending on SQL performance and SQL maintenance
- There are several sub-reports with the same look and feel, because it was simpler to copy the report and just change the filter for the specific need.
- Each sub-report will be hidden in SSRS to avoid direct usage and keep the folder as clean as possible.
- The reports are made on SSRS 2017. I haven't tested other versions. 

# The report explained:
The main report dashboard looks like this:

![Update dashboard](/.attachments/UpdateReporting001.PNG)

The report has basically two sub-reports. One to show you a list of systems in a specific state and one for a single system view and a list of missing updates.

![Update dashboard levels](/.attachments/UpdateReporting001-level.PNG)


I used different KPIs to measure update compliance and the following report combines all that into one dashboard. The main KPI is the first bar and all the others should simply help identify patch problems or flaws in your deployment strategy.  


| Nr | Name                    | Description |
|----|-------------------------|----------------------------------|
| 1      | Filter Collection Name  | A filter to easily find the collections you are looking for. Especially helpful if you have a lot of them. <br> If you don't know the correct name of the collection use the % sign as a wildcard. <br> The filter will filter the result of the "Choose Collections" parameter and reduce the number of collections visible in the drop down list.            |
| 2      | Choose Collections      |  The drop down list will show collections based on the filter you set.<br> You can choose just one collection or multiple ones.<br>If you choose more then one collection, the combined compliance status of all the systems will be shown in the report.<br>The report will always open with a default collection if the filter and the collection is set correctly during setup. Meaning, if the filter is set to "All%" and the default CollectionID is set to "SMS00001" the "All Systems" collection will be used.            |
| 3      | Show report      |  Will run the report with the currently selected collections           |
| 4      | Update compliance      |   "Compliant" (green bar) means, all the deployed updates to the systems are installed and at least one security update was installed within the month. The report is using the Win32_QuickfixEngineering class to determine the last installation time. (See also the: "Some key facts and prerequisites" section) <br> Click either on the green bar to get a sub-report which shows a list of compliant systems or the yellow bar to get a list of non compliant systems.          |
| 5      |  Updates approved     | The green bar will indicate that all the security and critical updates each system needs are deployed and could be installed by the system. <br> The yellow bar indicates systems which are missing security and critical updates which are currently not deployed to the systems. <br> It could mean that your update group is simply missing some important updates, which should be deployed. <br> You can click on the yellow bar to get a list of the updates missing for the systems in the chosen collection/s.          |
| 6      |Last Rollup Installed       | Green means the system has either the last or the current rollup installed. <br> Either the cumulative update or the Security Monthly Quality Rollup like this: <br> _2020-01 Cumulative Update for Windows%_ <br> _2020-01 Security Monthly Quality Rollup%_ <br> <br>Yellow means, the system is missing the rollup of the last month. <br> Since Microsoft is releasing updates with a year and date prefix, it is easy to determine the rollup of a given month by just that prefix. Like 2020-01 for the January rollup of 2020. <br> Click either on the green bar to get a sub-report which shows a list of compliant systems or the yellow bar to get a list of non compliant systems.            |
| 7      | Current Rollup Installed     | Green means the system has the current rollup installed. <br> Either the cumulative update or the Security Monthly Quality Rollup like this: <br> _2020-02 Cumulative Update for Windows%_ <br> _2020-02 Security Monthly Quality Rollup%_  <br> <br> Yellow means, the system is missing the current rollup. <br>Click either on the green bar to get a sub-report which shows a list of compliant systems or the yellow bar to get a list of non compliant systems.  <br> <br> Keep in mind that the green bar depends on when you open up the report. So if you want to report the compliance for lets say January, but you open up the report the 1st of February, then the current rollup bar will be using February as the current month and should only show yellow. <br> In that case the "Last Rollup Installed" is a good indicator, because it will show the rollup compliance based on January.            |
| 8      | Reboot pending      | Green means there is no reboot pending. <br> Yellow means, the system needs a reboot. Since the data is coming from the SCCM client via fast channel and not via hardware inventory or other method, the status should update quite fast.  <br>Click on the yellow bar to get to a sub-report of systems in need for a reboot.           |
| 9      |   WSUS-Scan Error    |  Green means there is no problem with the WSUS client scanning for updates. <br> Yellow means, the system reported a WSUS client scan error and the WSUS client should be checked. <br> The WindowsUpdate.log is a good starting point. <br> Click on the yellow bar to get to a sub-report of systems with a wsus client scan error.           |
| 10     | Last Update Installation      |   The pie chart is using data from Win32_QuickfixEngineering and is divided into three parts. <br> Group A (green) systems were the last security update was installed in the current month. <br>	Group B (yellow) systems were the last security update was installed in the last month.<br>Group C (red) systems were the last security update was installed before two or more month.          |
| 11     | Last Reboot      |  The pie chart is using data from hardware inventory and is divided into three parts. <br>Group A (green) systems were the last reboot was in the current month. <br>Group B (yellow) systems were the last reboot was in the last month. <br>Group C (red) systems were the last reboot was before two or more month           |
| 12     | Last ADDS Login      | The pie chart is using data from AD system discovery and is divided into three parts. <br> Group A (green) systems were the last logon in AD was in the current month. <br>	Group B (yellow) systems were the last logon in AD was in the last month. <br>Group C (red) systems were the last logon in AD was before two or more month            |
| 13     | Last SCCM Policy request      |  The pie chart is using default SCCM data and is divided into three parts. <br>	Group A (green) systems were the last policy request was in the current month. <br>	Group B (yellow) systems were the last policy request was in the last month. <br>Group C (red) systems were the last policy request was before two or more month           |
| 14     |  Top 10 systems with missing updates     | A list of the top 10 systems with the most missing updates. You might want to check those systems first.<br> I also tried the top 10 by month since last security update installation, but that list mostly contained systems which are decommissioned or off for a while, so I changed it to most missing updates.  |


## Sub-reports

Almost each bar or pie chart links to a sub-report to show compliance state of that subset of systems to give you better visibility.
This is an example of all the uncompliant systems from the first bar in the dashboard and is basically how the Excel list in the early days looked like:

![Update sub-report](/.attachments/UpdateReporting002.PNG)

| Nr | Name                    | Description |
|----|-------------------------|----------------------------------|
| 1      |  Name |   Name of the system |
| 2      |  OSType |   OS name coming from hardware inventory |
| 3      |  Client Version |   The SCCM client version, which should normally be the same for all systems |
| 4      |  WSUSVersion |  The WSUS client version, which should be the same for each OS type |
| 5      |  Defender Pattern |  The currently installed Defender pattern version. <br> Just as reference. A very old version can also indicate a software update problem. <br> Should be empty if you're not using Defender as your AntiVirus solution.  |
| 6      |  Pending Reboot |  Will show yes if a reboot is pending. The type of reboot can be found in the ConfigMgr/MECM console in the "Pending Restart" column.  <br> (You might need to add the column first)|
| 7      |  Days since last online |   The column is using the last time the ConfigMgr/MECM client requested a policy. <br> If no data is available the value will be 999 <br>A greater value might explain a missing update, because the client is off or not working anymore and not capable of installing updates.|
| 8      |  Days since last AADSLogon |   The column is using the "LastLogonTimeStamp" from "AD System Discovery"<br> If no data is available the value will be 999 <br> A greater value might explain a missing update, because the client is off or not working anymore or simply has been disposed and is not capable of installing updates. |
| 9      |  Days Since Last Boot |   This column is using hardware inventory data to calculate the last time the system was booted. <br> Note: If the reboot is older then ~30 days, this might indicate a problem, because the system should have been rebooted by the SCCM client after the update installation if you allow that to happen. <br> If no data is available the value will be 999 <br> A greater value might explain a missing update, because the client is off or not working anymore or simply has been disposed and is not capable of installing updates.  |
| 10      |  Month Since Last Update Install |   This column is using data from Win32_Quickfixengineering and is either an indicator of missing hardware inventory or an indication of an update installation problem. <br> The value will be shown in red if it is greater or equal to two month. |
| 11     |  Missing Updates All |   Count of all missing updates of all possible update categories for that system not just the ones that are deployed.  |
| 12      |  Missing updates Approved |   This is the most important column and shows the count of missing updates which are deployed to the system. <br> The value will be shown in red if it is greater or equal to one. |
| 13      |  Last Rollup Status |   The status will be "Missing" or "Installed" depending on the installation status of the rollup of the last month. The  Status will also be "Installed" if the rollup of the current month is installed, because each rollup is cumulative. <br> The value will be shown in red if it the status is "Missing".  |
| 14      |  Current Rollup Status |  The status will be "Missing" or "Installed" depending on the installation status of the rollup of the current month. <br> (As mentioned before, the status depends on the on time the report was opened. See 7 above.) <br> The value will be shown in red if it the status is "Missing". |
| 15      |  Update Collections |  A list of collections were the system is a member of and for which are updates deployed. <br> A missing collection name for example might indicate that the system is not part of the correct or not all needed collections and might not receive updates because of that.  |



# 2nd Level sub-report per system
If you click on the number of missing updates in the column "missing updates appoved" a per system sub-report will be opend which will only show the specific missing updates for the selected system.
The report also shows installations errors if any happened.  Each error will link to a bing search with the hex value of the error.
The search looks like this: SEARCHSTRING

![Update sub-report per system](/.attachments/UpdateReporting003.PNG)

The sub-report is pre-filtered to only show "Deployed and missing updates", but the filter can be ajusted to show: "Not deployed but missing" or just "All Updates per device" to get the full list.

# How to subscribe to a report
LOREM IPSUM

# How to install
1. Make sure you have enabled **Win32_Quickfixengineering** in the client settings for hardware inventory
1. You could also use AD System Discovery to have further data, but that's no hard requirement. 
1. Either clone the repository or download the whole content.
1. Copy the whole contentn on the SQL Server Reporting Services Server (SSRS) 1. Create a new folder on the report server website were the reports should be imported to.
   1. The folder should be under the normal configmgr folder, but can also be at the root level of your Reporting Services Server. But keep in mind that report subscriptions are only visible in the SCCM console, if the report, you have subscribed for, is below the normal configMgr folder. <br> The subscription will not be visible in the SCCM console if the report was placed at the root level.
1. Start a PowerShell session as admin. 
	a. The user running PowerShell also needs to have admin rights on the SQL Reporting Services Server in order to upload the reports
1. Change the directory to the folder were the import script **"Import-SSRSReports.ps1"** can be found.
1. Start the script **"Import-SSRSReports.ps1"** with the appropriate parameters (see below)
   1. The script will copy each rdl and rsd file from the **"Sourcefiles"** folder to a new **"work"** folder in the same directory the script resides.
	 1. The script will then simply replace some values with the parameter values you provided
	 1. The script will then upload the datasets and the reports to the server and the folder you provided as parameters
	 1. The files in the **"work"** folder will not be deleted and can be used as a backup or for manual uploads if necessary and will contain the data you provided as parameters to the script
	 1. **IMPORTANT:** If you need to re-run the script, delete each report and dataset on the Reporting Server in the folder you chose for the reports first. It is possible to overwrite the reports automatically, but I faced some issues with some settings not to be overwritten as desired and don't use that method anymore. That's why everything should be deleted first. 


| Parameter | Required  | Example value                    |Description |
|-----------|-----------|----------------------------------|----------------------------------|
|ReportServerURI| Yes |  http://reportserver.domain.local/reportserver | The URL of the SQL Reporting Services Server. <br> Can be found in the ConfigMgr Console or the report **LOREM IPSUM**|
| TargetFolderPath | Yes  | Example value |ConfigMgr_P11/{5C6358F2-4BB6-4a1b-A16E-8D96795D8602} |
| DefaultCollection | No  | Example value |SMS00001 |
| DefaultCollectionFilter | No  | Example value | All% |
| DoNotHideReports | No  | 'Software Updates Compliance - Overview','Compare Update Compliance' |Array of reports which should not be set to hidden. You should not use the parameter unless you really want more reports to be visible.|
| Upload | No  | $true |If set to $false the reports will not be uploaded. That might be helpful, if you do not have the rights to upload and need to give the files to another person for example. In that case, just use the report files in the work folder |
| UseViewForDataset | No  | $false |All reports can either use a dataset called "UpdatesSummary", which is the default and will execute the full sql query right from the Reporting Services Server, or a dataset called "UpdatesSummaryView" which will select from a sql view which needs to be created first. (I will not explain that process in detail) <br> $false will use the default dataset and $true will use the dataset using a sql view. |
| ReportSourcePath | No  | $PSScriptRoot or "C:\Temp\Reports"                   |The script will use the script root path to look for a folder called "Sourcefiles" and will copy all the report files from there.  But you could also provide a different path where the script should look for a "Sourcefiles" folder|

  ## Example: 
.\Import-SSRSReports.ps1 -ReportServerURI "http://reportserver.domain.local/reportserver" -TargetFolderPath  "ConfigMgr_P11/Custom_UpdateReporting" -TargetDataSourcePath = "ConfigMgr_P11/{5C6358F2-4BB6-4a1b-A16E-8D96795D8602}"

```code
PS> Get-Gelp .\Import-SSRSReports.ps1 -Full

NAME
    .\Import-SSRSReports.ps1
    
SYNOPSIS
    Uploads SQL Server Reporting Services report and dataset files.
    
    
SYNTAX
    .\Import-SSRSReports.ps1 [-ReportServerUri] <String> [-TargetFolderPath] <String> [-TargetDataSourcePath] <String> [[-DefaultCollectionID] <String>] [[-DefaultCollectionFilter] <String>] 
    [[-DoNotHideReports] <Array>] [[-Upload] <Boolean>] [[-UseViewForDataset] <Boolean>] [[-ReportSourcePath] <String>] [<CommonParameters>]
    
    
DESCRIPTION
    The script will change the content of rdl and rsd files and will upload them to a SQL Server Reporting Services (SSRS) of your choice.
    The rdl and rsd files contain specific strings which are simply replaced by the parameter values of this script. 
    
    Disclaimer
    This sample script is not supported under any Microsoft standard support program or service. This sample
    script is provided AS IS without warranty of any kind. Microsoft further disclaims all implied warranties
    including, without limitation, any implied warranties of merchantability or of fitness for a particular
    purpose. The entire risk arising out of the use or performance of this sample script and documentation
    remains with you. In no event shall Microsoft, its authors, or anyone else involved in the creation,
    production, or delivery of this script be liable for any damages whatsoever (including, without limitation,
    damages for loss of business profits, business interruption, loss of business information, or other
    pecuniary loss) arising out of the use of or inability to use this sample script or documentation, even
    if Microsoft has been advised of the possibility of such damages.
    

PARAMETERS
    -ReportServerUri <String>
        The URL of the SQL Reporting Services Server. Like this for example: http://reportserver.domain.local/reportserver
        
        Required?                    true
        Position?                    1
        Default value                http://reportserver.domain.local/reportserver
        Accept pipeline input?       false
        Accept wildcard characters?  false
        
    -TargetFolderPath <String>
        The folder were the reports should be placed in. I created a folder called "Custom_UpdateReporting" below the default SCCM reporting folder. My sitecode is P11, so the default folder is called "ConfigMgr_P11".
        Like this for example: "ConfigMgr_P11/Custom_UpdateReporting"
        Use '/' instead of '\' because it's a website
        
        Required?                    true
        Position?                    2
        Default value                ConfigMgr_P11/Custom_UpdateReporting
        Accept pipeline input?       false
        Accept wildcard characters?  false
        
    -TargetDataSourcePath <String>
        The path should point to the default ConfigMgr datas ource. 
        In my case the Sitecode is P11 and the default data source is therefore in the folder "ConfigMgr_P11" and has the ID "{5C6358F2-4BB6-4a1b-A16E-8D96795D8602}"
        The path with the default folder is required. Like this for example: "ConfigMgr_P11/{5C6358F2-4BB6-4a1b-A16E-8D96795D8602}""
        Use "/"" instead of "\"" because it's a website
        
        Required?                    true
        Position?                    3
        Default value                ConfigMgr_P11/{5C6358F2-4BB6-4a1b-A16E-8D96795D8602}
        Accept pipeline input?       false
        Accept wildcard characters?  false
        
    -DefaultCollectionID <String>
        
        Required?                    false
        Position?                    4
        Default value                SMS00001
        Accept pipeline input?       false
        Accept wildcard characters?  false
        
    -DefaultCollectionFilter <String>
        The filter is used to find the collection you are interested in and the value needs to match the name of the collection you choose to be the default collection for the parameter "defaultCollection". 
        In my case "All%" or All Syst% or "Servers%" to get the "Servers of the environment" collection for  example.
        
        Required?                    false
        Position?                    5
        Default value                All%
        Accept pipeline input?       false
        Accept wildcard characters?  false
        
    -DoNotHideReports <Array>
        Array of reports which should not be set to hidden. You should not use the parameter unless you really want more reports to be visible.
        
        Required?                    false
        Position?                    6
        Default value                @('Software Updates Compliance - Overview','Compare Update Compliance','Software Updates Compliance - Offline Scan Results')
        Accept pipeline input?       false
        Accept wildcard characters?  false
        
    -Upload <Boolean>
        If set to $false the reports will be changed to have the correct values, but will not be uploaded. 
        That might be helpful, if you do not have the rights to upload and need to give the files to another perso, so that they can be uploaded manually
        
        Required?                    false
        Position?                    7
        Default value                True
        Accept pipeline input?       false
        Accept wildcard characters?  false
        
    -UseViewForDataset <Boolean>
        All reports can either use a dataset called "UpdatesSummary", which is the default and will execute the full sql query right from the Reporting Services Server, or a dataset called "UpdatesSummaryView" which will select 
        from a sql view which needs to be created first. (I will not explain that process in detail)
        $false will use the default dataset and $true will use the dataset using a SQL view.
        
        Required?                    false
        Position?                    8
        Default value                False
        Accept pipeline input?       false
        Accept wildcard characters?  false
        
    -ReportSourcePath <String>
        The script will use the script root path to look for a folder called "Sourcefiles" and will copy all the report files from there. 
         But you could also provide a different path where the script should look for a "Sourcefiles" folder.
        
        Required?                    false
        Position?                    9
        Default value                $($PSScriptRoot)
        Accept pipeline input?       false
        Accept wildcard characters?  false
        
    <CommonParameters>
        This cmdlet supports the common parameters: Verbose, Debug,
        ErrorAction, ErrorVariable, WarningAction, WarningVariable,
        OutBuffer, PipelineVariable, and OutVariable. For more information, see 
        about_CommonParameters (http://go.microsoft.com/fwlink/?LinkID=113216). 
    
INPUTS
    None. You cannot pipe objects to Import-SSRSReports.ps1
    
    
OUTPUTS
    Just normal console output. Nothing to work with.
 
    -------------------------- EXAMPLE 1 --------------------------
    
    PS>.\Import-SSRSReports.ps1 -ReportServerURI "http://reportserver.domain.local/reportserver" -TargetFolderPath  "ConfigMgr_P11/Custom_UpdateReporting" -TargetDataSourcePath = 
    "ConfigMgr_P11/{5C6358F2-4BB6-4a1b-A16E-8D96795D8602}"
    
    -------------------------- EXAMPLE 2 --------------------------
    
    PS>.\Import-SSRSReports.ps1 -ReportServerURI "http://reportserver.domain.local/reportserver" -TargetFolderPath  "ConfigMgr_P11/Custom_UpdateReporting" -TargetDataSourcePath = 
    "ConfigMgr_P11/{5C6358F2-4BB6-4a1b-A16E-8D96795D8602}" -Upload $false

    -------------------------- EXAMPLE 3 --------------------------
    
    PS>.\Import-SSRSReports.ps1 -ReportServerURI "http://reportserver.domain.local/reportserver" -TargetFolderPath  "ConfigMgr_P11/Custom_UpdateReporting" -TargetDataSourcePath = 
    "ConfigMgr_P11/{5C6358F2-4BB6-4a1b-A16E-8D96795D8602}" -DefaultCollection "P1100012" -DefaultCollectionFilter "All Servers of Contoso%"
    
RELATED LINKS
    https://github.com/jonasatgit/updatereporting
``` 



# Some tipps

LOREM IPSUM