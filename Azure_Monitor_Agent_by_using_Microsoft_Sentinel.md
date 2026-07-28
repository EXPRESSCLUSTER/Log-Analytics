# Azure Log Analytics with Azure Monitor Agent by using Microsoft Sentinel

This guide details how to set up an Azure Monitor Agent (AMA) by using Microsoft Sentinel to collect ECX log files from on-premises Windows server nodes and monitor them for error events.

Note that the legacy Log Analytics agent (MMA) was deprecated by Microsoft in August 2024, so this guide uses the Azure Monitor Agent instead. These steps are for ECX VM nodes in an on-premises environment.

## 1. Activate Microsoft Sentinel

Search for and select Microsoft Sentinel in the Azure portal.
Select a workspace to use, or create a new workspace.

## 2. Select "Windows Security Events with AMA" from Data Connectors

A data connector specifies how data is ingested into Microsoft Sentinel.
Select **Windows Security Events with AMA** from the Sentinel Data Connectors list.

![Select a Data Connector](images/image1_Select-Data-Connectors.png)

## 3. Create data collection rules

A Data Collection Rule defines the data collection process in Azure Monitor. It specifies what will be collected, where to send the data, and how it will be transformed.

![Create data collection rules](images/image2_Create-data-collection-rules.png)

## 4. Install AMA and select target VMs for log collection

Install the Azure Monitor Agent in the Azure Portal.
Select the VMs where you want to deploy the Azure Monitor Agent.

## 5. Select the type of Windows Security Event log

Depending on the application, you can select which event content to acquire. Refer to the [official documentation](https://learn.microsoft.com/en-us/azure/sentinel/windows-security-event-id-reference) for details.

## 6. Create

After setting the parameters, the Azure Monitor Agent will be deployed within a few minutes, and you'll be able to check the Windows Security Event Log.

---

# Alert messages to Microsoft Teams by using Microsoft Sentinel

When monitoring the operation of Microsoft Sentinel, you can send an incident alert to Microsoft Teams when a high-severity incident occurs.

## 1. Enable "Post message to Teams"

Select **Post message to Teams** from the **Playbook templates** in Automation.
Confirm that **PostMessageTeams** has been created in the logic app.
Since authentication to Microsoft Teams isn't established immediately after setup, configure it from the designer screen.

![Enable Post message to Teams](images/imsge3-post-teams-rule.png)

## 2. Teams channel settings

Log in to your Microsoft Teams account and set the target team name and channel name in the logic app designer settings.

![Setting Teams channel](images/image4-setting-teams-chanel.png)

## 3. Test

Check alert messages using the Microsoft Defender for Cloud sample alert feature:

1. Enable the connector on the Microsoft Sentinel side.
2. Create a sample alert from Microsoft Defender for Cloud.
3. Verify the alert message appears on the Microsoft Teams side.
