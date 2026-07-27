# ECX log analysis with Azure Log Analytics

To analyze an ECX log file in Azure Log Analytics, the log file must meet the following criteria:

- The log must either have a single entry per line, or use a timestamp matching one of the following formats at the start of each entry:
    ```
    YYYY-MM-DD HH:MM:SS
    M/D/YYYY HH:MM:SS AM/PM
    Mon DD, YYYY HH:MM:SS
    yyMMdd HH:mm:ss
    ddMMyy HH:mm:ss
    MMM d hh:mm:ss
    dd/MMM/yyyy:HH:mm:ss zzz
    yyyy-MM-ddTHH:mm:ssK
    ```
- The log file must not use circular logging. This is the log rotation behavior where a file is overwritten with new entries, or renamed and then the same file name is reused for continued logging.
- The log file must use ASCII or UTF-8 encoding. Other formats, such as UTF-16, aren't supported.
- On Linux, time zone conversion isn't supported for timestamps in the logs.
- As a best practice, the log file should include the date and time it was created, to prevent log rotation from overwriting or renaming it.

## Converting character encoding and date/time format

The ECX for Windows log file is encoded in SJIS and uses the date/time format `YYYY/MM/DD HH:MM:SS.ZZZ`. The following commands convert it to UTF-8 with the `YYYY-MM-DD HH:MM:SS` format:

1. Install [Git for Windows](https://gitforwindows.org/).
2. Open `Git Bash`, then run the following commands.
3. Change directory to the location where the ECX log files were extracted:
    ```sh
    cd /c/Users/USER-A/Downloads/SampleCluster/node-1/log
    ```
4. Convert the character encoding from SJIS to UTF-8:
    ```sh
    iconv -f SJIS -t UTF-8 userlog.00.log > userlog.00.utf8.log
    ```
5. Convert the date/time format:
    ```sh
    sed -i -r 's/^(....)\/(..)\/(.. ..:..:..)\./\1-\2-\3 /' userlog.00.utf8.log
    ```

Note: Kusto's `todatetime()` command returns a `NULL` string if the `YYYY/MM/DD HH:MM:SS.ZZZ` format is used. Replacing forward slashes with hyphens (`YYYY-MM-DD HH:MM:SS.ZZZ`) resolves this.

## Custom logs

> **A note on the Log Analytics agent**: the steps below use the legacy Log Analytics agent (MMA), which Microsoft deprecated in August 2024. For new setups, use the Azure Monitor Agent (AMA) instead — see [Azure_Monitor_Agent_with_Azure_Arc.md](Azure_Monitor_Agent_with_Azure_Arc.md) for on-premises servers connected via Azure Arc, or [Azure_Monitor_Agent_by_using_Microsoft_Sentinel.md](Azure_Monitor_Agent_by_using_Microsoft_Sentinel.md) for a Microsoft Sentinel-based setup. The steps below are kept for reference and for existing deployments still on the legacy agent.

First you'll need a Log Analytics workspace in Azure. Search for *Log Analytics workspaces* from the Azure home page and create a new workspace.

### Link to log files on your PC

Install the Log Analytics agent on your PC.

1. Open the Log Analytics workspace and go to *Agents Management*.
2. Select the Windows servers tab or the Linux servers tab, depending on the OS where your log files exist.
3. Expand *Log Analytics agent instructions*.
4. Download the appropriate agent for your OS.
5. Copy the *Workspace ID* and the *Primary key* (used later to link your PC to the Azure Log Analytics workspace).
6. Install the Log Analytics agent on your PC and enter the *Workspace ID* and *Primary key* in the appropriate fields.

Add custom logs:

1. Open the Log Analytics workspace you created and click *Custom logs* under *Settings*.
2. Click *Add custom log* under the *Custom tables* tab.
3. Select one of the log files in the log files folder on your PC as a sample log. Click Next.
4. The contents of the log file should display in the Preview window. Choose *New line* as the delimiter. Click Next.
5. Choose the appropriate OS as the *Type*, then enter the path to the log files.
    e.g. `C:\temp\logfiles\*.log`
    Click Next.
6. Enter a name for the log file. Note that `_CL` will be automatically appended. Click Next.
7. Click *Create*.

## The first step for analyzing logs

The following examples assume `NODE1_CL` as the Type of the custom log. Open the Log Analytics workspace you created and click *Logs* under *General*.

### Extracting all entries

Enter the following, then click `Run`:

```KQL
search * | where Type == "NODE1_CL"
```

or simply:

```KQL
NODE1_CL
```

The output should look like the following.
*Note: this behavior seems to only become available around 24 hours after uploading the log file.*

![ScreenShot_20230131_122743.png](ScreenShot_20230131_122743.png)

### Extracting ERRORs

Enter the following, then click `Run`:

```KQL
NODE1_CL
| extend ERR = extract("(ERROR)", 1, RawData)
| where ERR == "ERROR"
| distinct RawData
```

![ScreenShot_20230203_125200.png](ScreenShot_20230203_125200.png)

## Alert mail

### Create an alert rule

Log query alert rules create an alert when a log query returns a particular result.

1. Select **New alert rule** to create a new alert rule based on the current log query.
2. Set the **Operator** and **Threshold value** in the alert logic. An alert is created when this condition is true.
3. Click **Add action groups** to add one to the alert rule. Select a Subscription and Resource group for the action group, and give it an Action group name (shown in the portal) and a Display name (shown in email and SMS notifications).
4. Configure the remaining settings in the Alert rule details section (Alert rule name, Description, Severity, and whether to enable the alert upon creation).

### Notes on alerting

Alert rules are most useful when there's a specific, well-defined condition to notify on — for example, a spike in `ERROR` entries within a short window, rather than every log line being ingested. When setting up alerting for a custom log, define the query and threshold around the specific condition you want to be notified about, rather than alerting on all incoming data.

## Methods to analyze ECX log files in Azure

1. **Azure Arc-enabled servers with Azure Monitor Agent** — detailed instructions for creating the Azure resources and configuring on-premises ECX VM servers are in [Azure_Monitor_Agent_with_Azure_Arc.md](Azure_Monitor_Agent_with_Azure_Arc.md).
2. **Azure Monitor Agent via Microsoft Sentinel** — an alternative setup that also configures Microsoft Sentinel and Teams alert integration; see [Azure_Monitor_Agent_by_using_Microsoft_Sentinel.md](Azure_Monitor_Agent_by_using_Microsoft_Sentinel.md).

## Methods to analyze ECX log files with Perl script
A Perl script has been added to this repository which can help with ECX log file troubleshooting by ordering trouble events by the time that they occurred. The script file is called [merge.pl](merge.pl).
