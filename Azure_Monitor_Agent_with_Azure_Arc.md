# Azure Log Analytics with Azure Arc-enabled servers and Azure Monitor Agent

This guide details how to set up Azure Monitor to collect ECX log files from on-premises Windows VM server nodes, monitor them for error events, and send email alerts when errors are detected.

Note that the legacy Log Analytics agent (MMA) was deprecated by Microsoft in August 2024, so this guide uses the Azure Monitor Agent (AMA) instead. These steps are for ECX VM nodes in an on-premises environment.

> If you're using Microsoft Sentinel rather than connecting servers via Azure Arc directly, see [Azure_Monitor_Agent_by_using_Microsoft_Sentinel.md](Azure_Monitor_Agent_by_using_Microsoft_Sentinel.md) instead.

## Prerequisites

- An active Azure subscription with permissions to create Resource Groups, Log Analytics workspaces, and Data Collection Rules (Contributor role or equivalent at the subscription/resource group level).
- Administrator access on the on-premises ECX Windows server nodes to run the Azure Arc onboarding script.
- Outbound internet connectivity from the on-premises server to Azure endpoints (required for the Azure Connected Machine agent).
- EXPRESSCLUSTER installed and logging to the default log path (`C:\Program Files\EXPRESSCLUSTER\log\`), or a known custom log path.

In order to use Azure Monitor Agent to analyze ECX log files, you'll need to create the following resources in Azure:

1. Resource Group
2. Log Analytics workspace
3. On-premises server prep (Azure Arc)
4. Custom table
5. Data Collection Rule
    - Data collection endpoint
6. Alert Rule
    - Action group

## 1. Resource Group

Log into the [Azure Portal](https://portal.azure.com/) and create a **Resource Group** for this solution.

1. Search for and select **Resource groups**.
2. Click **Create**.
    - Select your **Subscription**.
    - Enter a unique name for your **Resource group**.
    - Choose the appropriate **Region** to use for all of the resources to be created.
3. Click **Review + Create**.
4. Click **Create**.

*Note: you'll use this resource group for all of the resources created for this project.*

## 2. Log Analytics workspace

1. Search for **Log Analytics workspaces** in the Azure Portal.
2. Click **Create**.
3. Choose your **Subscription** and the **Resource Group** just created. Enter a unique **Name** and appropriate **Region**. Add **Tags** if needed.
4. Click **Review + Create**.
5. Click **Create**.

## 3. On-premises server prep

Azure Arc needs to be enabled on the on-premises server in order to send log files to Azure Monitor. You'll need to deploy and configure the **Azure Connected Machine agent** on your server and then connect it to Azure. This can be done manually, but the simplest way is to download a script that automates the process. This script downloads and installs the **Connected Machine agent** and connects it to the **Azure Monitor Agent** extension under Azure Arc.

1. Log into the [Azure Portal](https://portal.azure.com/).
2. Search for and select **Servers - Azure Arc**.
3. Click **Add** and choose **Generate script** (for a single server) to run on your target server.
    ![Generate Script](images/Installed%20Generate%20Script.png)
4. Review the **Prerequisites** page and click **Next**.
5. On the **Resource details** tab:
    - Select your **Subscription**.
    - Select the appropriate **Resource Group**.
    - Select the appropriate **Region**.
    - Select the appropriate **Operating system**.
    - Select the appropriate **Connectivity method** (usually Public endpoint).
6. Click **Next**.
7. On the **Tags** tab:
    - Enter values for the **Physical location tags** (if desired), and any other custom tags as needed.
8. Click **Next**.
9. Click **Download** and review the script execution instructions.
10. Click **Close**.
11. Copy the downloaded script to your target server.
12. Open an elevated PowerShell window on your server, change to the directory containing the script, and run `./OnboardingScript.ps1`.
    *Note: you may need to change the execution policy to run the script. The script will prompt you to enter your Azure credentials to connect to Azure. An Azure Arc-enabled resource will be created for your server and associated with the agent.*
13. Verify this succeeded by returning to the Azure Portal and accessing the **Azure Arc - Servers** page. Your server should be listed with the status *Connected*, and Azure Monitor Agent should be installed as an extension of this Azure Arc server resource. The **Azure Connected Machine Agent** will have been installed on your on-premises server.
    ![Azure Arc Server](images/Installed%20Azure%20Arc%20Server.png)
    ![Azure Arc Extension](images/Installed%20Azure%20Arc%20Extension.png)

## 4. Custom table

A custom table needs to be created in the Log Analytics workspace for the log data that will be collected. The Data Collection Rule, created in the next section, will channel log file data into this table.

1. Copy the code below and change the parameters in braces to match your Azure environment. This can be modified to add other columns if needed. Be careful not to add extra spaces or indentation to the script beyond what's shown, since PowerShell here-strings are sensitive to this. You'll choose your own `TableName` (replace the placeholder in both places it appears). Also be sure to swap out the `subscription`, `resourcegroup`, and `Workspacename` placeholders with your actual values.

```powershell
$tableParams = @'
{
  "properties": {
    "schema": {
      "name": "{TableName}_CL",
      "columns": [
        {
          "name": "TimeGenerated",
          "type": "DateTime"
        },
        {
          "name": "RawData",
          "type": "String"
        }
      ]
    }
  }
}
'@

Invoke-AzRestMethod -Path "/subscriptions/{subscription}/resourcegroups/{resourcegroup}/providers/microsoft.operationalinsights/workspaces/{WorkspaceName}/tables/{TableName}_CL?api-version=2021-12-01-preview" -Method PUT -payload $tableParams
```

2. The easiest way to create this table is from an **Azure Cloud PowerShell** command line in Azure. From the Azure Portal, press the **Cloud Shell** button in the top-right bar, then select **PowerShell**. Copy and paste the script and press Enter to execute it.
3. To verify the table was created, return to your **Log Analytics workspace** in Azure and click **Tables** under **Settings**.
    *Note: this script and these instructions are based on the [Microsoft Learn documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/agents/data-collection-text-log?tabs=portal).*
    ![Log Analytics Workspace Table](images/Installed%20Tables.png)

## 5. Data Collection Rule

A Data Collection Rule defines the data collection process in Azure Monitor. It specifies what will be collected, where to send the data, and how it will be transformed. The rule below collects ECX custom logs from on-premises ECX servers.

1. Search for **Monitor** in the Azure Portal to access the Monitor menu.
2. Click **Data Collection Rules** in the left blade, under **Settings**.
3. Select **Create**.
4. On the **Basics** tab:
    - Enter a unique **Rule Name** for this rule.
    - Select your **Subscription**.
    - Select the **Resource Group** created earlier.
    - Select the correct **Region**.
    - Select the **Platform Type** (Windows or Linux).
    *Note: a Data Collection Endpoint will be created in a later step.* Click **Next: Resources**.
5. On the **Resources** tab, click **Add resources**.
    ![DCR Resources](images/Installed%20DCR%20Resources.png)
6. Expand your resource group to show the Azure Arc-enabled servers. Check the boxes next to the servers to include in scope and click **Apply**.
7. Click **Create endpoint**.
    *Note: an endpoint is needed for Custom Text Logs.*
    - Enter a unique Endpoint Name.
    - Select your Subscription.
    - Select the Resource Group created earlier.
    - Select the same Region used previously.

    Click **Review + Create**, then click **Create**.
8. Check the box next to **Enable Data Collection Endpoints** to show the Data Collection Endpoint column in the lower table.
9. Select the Data collection endpoint just created, in the new column, for each server.
10. Return to the **Basics** tab and select the Data Collection Endpoint just created.
11. Click **Next: Resources**, then **Next: Collect and deliver** to continue.
12. Click **Add data source**.
13. Open the dropdown under **Data source type** to see the available options.
    ![Data Source Type](images/Installed%20Data%20Source%20Type.png)
14. Select **Custom Text Logs**.
    *Note: if no data collection endpoint was created and selected earlier, this option won't be available.*
15. Enter the following in the Data source window:
    - **File pattern**: `C:\Program Files\EXPRESSCLUSTER\log\userlog*.log`
    - **Table name**: the table name created previously (don't forget the `_CL` suffix).
    - **Transform**: leave as `source`.
16. Click **Next: Destination**.
17. Confirm the following:
    - **Destination type**: Azure Monitor Logs.
    - **Subscription** is correct.
    - **Account or namespace** is set to your Log Analytics workspace.
18. Click **Add data source**.
    The **Data source** column should show **Custom Text Logs**, and **Destination(s)** should show **Azure Monitor Logs**.
19. Click **Next: Review + Create**.
20. If everything looks good, click **Create**.
21. Click **Go to resource** to view the newly created resource.

## Verify that the text logs are being populated

1. Open the Azure **Monitor** page.
2. Click **Logs** in the left blade.
3. Close the **Queries** popup window.
4. Click **Select scope** in the upper left, expand your Resource group, select your Log Analytics workspace, and click **Apply**.
    ![Select a Scope](images/Installed%20Select%20a%20scope.png)
5. Enter the name of your custom log file in the query window (expand the **Custom Logs** list if you need a reminder). Click **Run**.
6. If you've given the system enough time to collect logs and nothing is displayed for the default **Last 24 hours** period, change the **Time range** to a longer period and try again.
    *Note: all log entries collected over that time period will display, including "INFO" events.*
    ![Basic Log Query](images/Azure%20Monitor%20Basic%20Log%20Query.png)
7. Enter the following query to view organized error events:

```kql
<log_name>_CL
| where RawData contains "ERROR"
| order by TimeGenerated asc
| project TimeGenerated, ComputerName=tostring(split(_ResourceId, "/")[-1]), RawData
```

![Query for ERROR](images/Azure%20Monitor%20Log%20Query%20for%20ERROR.png)

*Note: changing `contains` to `contains_cs` performs a case-sensitive query.*

## 6. Alert Rule

Once Azure starts collecting ECX logs, you can create an Alert Rule to notify the ECX administrator when error messages are logged.

1. Search for **Monitor** in the Azure Portal to access the Monitor menu.
2. Click **Alerts** in the left blade.
3. Click **Create -> Alert rule**.
4. On the **Scope** tab, a window called **Select a resource** should automatically appear. Expand your Resource group and check the box next to your Log Analytics workspace. Click **Apply**.
    ![Select a Resource](images/Installed%20Select%20a%20resource.png)
5. Click **Next: Conditions**.
6. Set the following on the **Condition** tab:
    - **Signal name**: **Custom log search**
      *Note: this will expand more options.*
    - **Search query**:
    ```kql
    <table_name_CL>
    | where RawData contains "ERROR"
    ```
    ![Alert Log Query](images/Installed%20Alert%20Log%20Query.png)
    *Note: you can test this query. If it doesn't return any results, adjust the Time range. Click **Continue Editing Alert** to close this window.*
    - **Measurement**: leave **Measure** as **Table rows**, **Aggregation type** as **Count**, and **Aggregation granularity** as **5 minutes**.
    - **Split by dimensions**:
      - **Resource ID column**: leave as `_ResourceId`.
      - Set a dimension to include the log entry that triggered the alert in the email:
        - **Dimension name**: `RawData`
        - **Operator**: `=`
        - **Dimension values**: Select all
        - **Include all future values**: check the box
    - **Alert logic**:
      - **Operator**: Greater than
      - **Threshold value**: 0
      - **Frequency of evaluation**: 5 minutes

    Leave the other settings at their defaults and click **Next: Actions**.
7. Click **Create action group** under the **Actions** tab.
    *Note: the action group tells Azure what to do when an alert is received.*
8. Enter the following on the **Basics** tab:
    - Select your Subscription.
    - Select the appropriate Resource Group.
    - Select the appropriate Region.
    - Enter a unique name for the Action group name.
    - The Display name will show up in all notifications; change it from the default if you like.
9. Click **Next: Notifications**.
10. Choose the following on the **Notifications** tab:
    - **Notification type**: Email/SMS message/Push/Voice.
    - **Name**: a name of your choice.
      *Note: if a popup didn't appear to let you add or edit the Email/SMS message/Push/Voice action, click the pencil icon.*
    - Check the box next to **Email** and enter the email address to receive notifications.
    - Select **Yes** to enable the common alert schema and click **OK**.
11. Click **Next: Actions**.
12. The settings on the **Actions** tab don't need to be modified. They can be used for more advanced actions if needed, such as webhooks, Azure Functions, and Logic Apps.
13. Click **Next: Tags** and add any tags as needed.
14. Click **Next: Review + Create**.
15. If everything looks good, click **Create**.
    The action group just created should be listed under Action group name.
    ![Action Group Created](images/Installed%20Action%20Group%20Created.png)
16. Click **Next: Details** to continue creating the alert rule.
17. On the **Details** tab, modify the following as needed:
    - Select your Subscription.
    - Select the appropriate Resource Group.
    - **Severity**: 1 - Error.
    - **Alert rule name**: a name of your choice.
    - **Alert rule description**: optional.
    - Select the appropriate Region.
    - **Advanced options**: set Custom properties (if desired).
18. Click **Next: Tags**.
19. Add any tag values and click **Next: Review + Create**.
20. Click **Create**.
    A notification in the top-right will confirm the new rule was created successfully. You'll be brought back to the main alerts page, which shows alerts fired within the selected time frame.
    ![Alerts Page](images/Installed%20Alerts%20Page.png)

    Click **Alert rules** to view and edit alert rules. New rules will show in the list and be enabled.
    ![Alert Rule List](images/Installed%20Alert%20Rule%20Created.png)

## Test the Alert Rule

1. Log into the standby node of your ECX cluster.
2. Disable the network adapter for at least 30 seconds.
3. Alternatively, you can create your own error message (skip to step 6 if you've already disabled the network adapter).
4. Open an elevated command prompt with Admin rights.
5. Enter the following command and press **Enter** (the message can be modified):
```
clplogcmd -m "Test error occurred. Logging now." -l ERROR
```
6. Wait approximately 5 minutes for an email to arrive with an error alert message.
7. Click **View the alert in Azure Monitor** to view details in Azure.

### Emailed Alert Sample
![Emailed Alert Sample](images/Email%20Alert.png)

### New Alert Details in Azure
![Alert Details](images/Alert%20Details.png)

### New Alert in Monitor Alerts
![New Alert in Alerts](images/Alert%20Logged.png)

### New Alert Query in Logs
![New Alert Query in Logs](images/Alerted%20Error%20in%20Log.png)

## Addendum

Workbooks are also a good resource for viewing log data in a clean interface. Multiple logged events (from different tables, if desired) can be redirected into workbooks, with queries to organize the data.

### Create a Workbook for ECX Log Data

1. Search for **Monitor** in the Azure Portal to access the Monitor menu.
2. Click **Workbooks** in the left blade.
3. Create a new workbook by clicking **New**.
4. Click **Add** -> **Add query**.
5. Set the following parameters:
    - **Data source**: Logs
    - **Resource type**: Log Analytics
    - **Log Analytics workspace**: click **Load all subscriptions**, then select your subscription.
    - **Time Range**: leave at the default **Last 24 hours**, or change to a longer period.
    - **Size**: leave visualization size at the default **Medium**, or change it.
    - **Log Analytics workspace Logs Query**: enter the following:
      ```kql
      <Tablename>_CL
      | where RawData contains "ERROR"
      | order by TimeGenerated desc
      | project TimeGenerated, RawData, _ResourceId
      ```
6. Click **Run Query**.
7. Click **Advanced Settings** and change the **Step name** and **Chart title** to something more meaningful.
8. Click the **Save** icon.
9. In the **Save As** window, enter a meaningful Title, Resource group, and Location.
10. Click **Apply**.
11. Click **Done Editing**.

This workbook can be pinned for easy access and will automatically update as new log events are captured, with no need to re-run the query when it's opened. If no results appear, edit the Time Range. See the [workbooks overview](https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/workbooks-overview) for more on what workbooks provide.

![Workbook](images/Installed%20Workbook.png)

### Parse User*.log contents into columns

The following Kusto query parses the ECX `User*.log` file into columns from a table.

Sample raw data input:
```
2023/01/20 16:34:51.571  00000d40 00000d9c INFO  [cmd  ] clpcl.exe : 0 : Command succeeded.
```

```kql
ECXUserLog_CL
| extend part1 = split(RawData, ' [')          // ["2023/01/20 16:34:51.571  00000d40 00000d9c INFO ", "cmd  ] clpcl.exe : 0 : Command succeeded."]
| extend part2 = part1[0]                       // 2023/01/20 16:34:51.571  00000d40 00000d9c INFO
| extend part3 = part1[1]                       // cmd  ] clpcl.exe : 0 : Command succeeded.
| extend part4 = split(part3, '] ')             // ["cmd  ", "clpcl.exe : 0 : Command succeeded."]
| extend module = part4[0]                      // cmd
| extend message = part4[1]                     // clpcl.exe : 0 : Command succeeded.
| extend part5 = split(part2, '  ')             // ["2023/01/20 16:34:51.571", "00000d40 00000d9c INFO"]
| extend tmptime = replace_string(tostring(part5[0]), @'/', '-')  // 2023-01-20 16:34:51.571
| extend date_time = todatetime(tmptime)        // 1/20/2023 4:34:51.571 PM
| extend part6 = split(part5[1], ' ')           // ["00000d40", "00000d9c", "INFO"]
| extend processID = part6[0]                   // 00000d40
| extend threadID = part6[1]                    // 00000d9c
| extend event = part6[2]                       // INFO
| project date_time, processID, threadID, event, module, message, Computer
```

Formatted table output:

| date_time (UTC) | processID | threadID | event | module | message | Computer |
|---|---|---|---|---|---|---|
| 1/20/2023, 4:34:51.571 PM | 00000d40 | 00000d9c | INFO | cmd | clpcl.exe : 0 : Command succeeded. | ECX01 |

### Azure Arc Extension Notes

Several Azure Arc extensions are available to run scripts on the connected server. These can be useful for remotely installing applications, securely connecting to servers for troubleshooting, or collecting log files.

#### Custom Script Extension for Windows

Automatically launches and executes machine customization tasks post-configuration. It can be used to install OpenSSH Server for Windows to enable SSH connections.
*Note: an Azure storage account is required for the script.*

#### OpenSSH for Windows

SSH for Arc-enabled servers enables SSH-based connections to Arc-enabled servers without requiring a public IP address or additional open ports. This can be used interactively, automated, or with existing SSH-based tooling, giving existing management tools greater reach into Azure Arc-enabled servers. An administrator can connect to a server from anywhere.

See [SSH access to Azure Arc-enabled servers](https://learn.microsoft.com/en-us/azure/azure-arc/servers/ssh-arc-overview?tabs=azure-cli).

#### Azure Automation Windows Hybrid Worker

The Azure Automation Hybrid Worker extension allows runbooks to execute directly on Azure or non-Azure machines, including Arc-enabled servers and Arc-enabled VMware VMs. A Hybrid Worker group with Hybrid Runbook Workers is designed for high availability and load balancing by distributing jobs across multiple workers. Scripts can be scheduled or run on-demand. An alert can use an associated action group to run a script in a runbook.

##### Configure an alert to run an Azure Automation runbook

1. Create an Automation Account.
2. Create a (User) Hybrid Worker group.
    Add one or more on-premises ECX VMs to the group.
    *Note: when a non-Azure machine is added to the group, the Hybrid Worker extension is installed automatically. The script will only run on one worker in the group.*
3. Create a runbook.
    - **Runbook type**: PowerShell.
    - Add PowerShell commands and test.
4. Edit the Action group (used by an alert).
    - **Action type**: Automation Runbook.
    - **Selected**: the runbook previously created.
      - Configure runbook:
        - **Runbook source**: User
        - **Automation account**: the automation account previously created
        - **Runbook**: the runbook previously created
      - Configure parameters:
        - **Run on**: Hybrid Worker
        - **Choose Hybrid Worker group**: the hybrid worker group previously created

See [Use an alert to trigger an Azure Automation runbook](https://learn.microsoft.com/en-us/azure/automation/automation-create-alert-triggered-runbook?WT.mc_id=Portal-Microsoft_Azure_Automation).

Simple sample PowerShell script in a runbook:    
![PowerShell Script](images/Runbook%20With%20Script.png)

Output of the PowerShell script from the Azure runbook log:    
![PowerShell Script Output](images/Runbook%20Output.png)
