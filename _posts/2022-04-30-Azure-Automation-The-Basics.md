---
classes: wide
title: "Azure Automation - The Basics"
header:
  og_image: "/assets/images/ArticleLogos/2022-04-30-Azure-Automation-The-Basics.jpg"  
excerpt: "The Azure based Automation Platform, what is it, what can it do and how does it fit into an Enterprise Scale context?"
categories:
  - Article
tags:
  - Architecture
  - Engineering
  - Level 100
---
*The Azure based Automation Platform, what is it, what can it do and how does it fit into an Enterprise Scale context?*

{% capture notice %}
**Associated Repositories and Articles**<br>
*Lab/Demo Environment:* [Article-2022-04-AzureAutomation-The-Basics](https://github.com/AabyeHald/Article-2022-04-AzureAutomation-The-Basics)<br>
*Related Articles:* The article "Azure Automation - Runbooks" will be coming soon
{% endcapture %}
<div class="notice">{{ notice | markdownify }}</div>

The purpose of this article is to give some insights into what an Azure based Automation Platform is and what it can do - I will also be providing my take on when it should be used and for what.

The key components of the Azure based Automation Platform are shown below, this is also what is included in the Lab/Demo Environment associated with this article.

![Automation Platform](/assets/images/Articles2022/Articles2022-Automation%20Platform.png "Automation Platform"){: .align-center}

Out of the box the Azure Automation Account provides a Scheduler, that is able to schedule Runbooks to be executed at specific times - a concept that is older than the current millennium.<br>
If the Azure Automation Account is deployed with a linked Log Analytics Workspace, Hybrid Workers and a Git, like GitHub or Azure DevOps - the last millenium scheduler moves to the next level and becomes something more suitable for this day and age.

In the following sections, I will dive into some more details on some of the central components.

# Log Analytics Workspace
One of the challenges for all platforms, especially when they start to scale, is to gain insights into what happens on the platforms and maintain the capability to act upon that insight.<br>
In other words, you need to be able to:

1. Collect data at scale (insights)
2. Query data at scale (act)

Log Analytics meets those 2 requirements nicely, which is why Log Analytics is a key component in an Azure based Automation Platform.

A typical usecase could be the collection of Virtual Machine inventory data into Log Analytics. This data can be queried and compared to data from Windows Update, the result of this comparison can then be used to schedule updates targeted specific, or all, Virtual Machines. After the scheduled update, change tracking can then be used to provide evidence that the update deployments to specific Virtual Machines was in compliance with specified service windows.

Another, more traditional usecase could be to provide Runbook execution data for monitoring of scheduled Runbooks and alerting on Runbook exceptions.

## VM Insights
In relation to an Azure based Automation Platform where Hybrid Workers are used, which is almost always recommended, VM Insights are one of the more useful but often overlooked features of Log Analytics.<br>
VM Insights provides detailed capacity insights into the hybrid workers, making it easier to scale the worker environment correctly.

![VMInsigths](/assets/images/Articles2022/Article2022-AutomationAccount-VMInsights.png "VMInsigths"){: .align-center}

As Hybrid Workers, in most cases, are used to perform jobs on environments surrounding the Hybrid Workers, the ability for the workers to connect to other systems either directly, or using an API becomes important. The direct connections are typically remote shells used for server administration, wheras the API could be a Cloud API like one of the Azure APIs.<br>
This is why knowing something about what the workers connects to and the quality of the connecitons becomes of some interest.

![VMInsigths-Connections](/assets/images/Articles2022/Article2022-AutomationAccount-VMInsights-Connections.png "VMInsigths-Connections"){: .align-center}

In other words, VM Insights gives a lot of the information required for proactive professional operations of the Hybrid Workers - or any other server for that matter.

# Automation Account
The Automation Account is basically a handfull of shared features supported by a scheduler - the shared features includes fancy reporting, variable and secret handling, etc. most of which will be covered later in this article.<br>
Another shared feature is the Sandbox, which is a place to execute Runbooks, if no other Hybrid Workers are available. The Sandbox is operating as a service, using a "fair share" principle that enforces certain limits on the Runbooks in terms of resource usage and execution time.<br>
The Sandbox is fine for some workloads, but not great, and most implementations ends up using Hybrid Workers for the Runbooks.

## Hybrid Workers
The Hybrid Workers are where all the jobs are executed, when not using the Automation Account Sandbox. Basically a Hybrid Worker is a Linux or Windows VM.<br>
Apart from the name, the Hybrid Workers are not necessarily hybrid - they can be hybrid, which usually involves Azure Arc for Servers or they can be Azure based.<br>
A Hybrid Worker comes in 2 flavours:
1. System Worker
2. User Worker

### System Worker
The System Worker only supports a set of hidden system runbooks used for the Update Management processes on both Windows and Linux. A System Worker are not part of any Worker Group, which means that a Runbook cannot be scheduled to run on System Workers. A hidden system Runbook can be scheduled however, which happens when Updates are scheduled to be deployed.<br>
A System Worker is installed automatically when a VM is associated with a Log Analytics Workspace that are linked to an Automation Account. The System worker can coexists with a User Worker.

### User Worker
A User Worker has a different featureset than a System Worker - this type of worker can execute user-defined runbooks, but not hidden runbooks. A User Worker are a member of a Worker Group, on which a Runbook can be scheduled to run.<br>
When registering a User Worker, the easiest way is to start out with a System Worker - then the script below will be all you need:

{% highlight PowerShell linenos %}
# Register the Virtual Machine workers
$RegistrationInfo = Get-AzAutomationRegistrationInfo -ResourceGroupName $ResourceGroupNameAutomationAccount -AutomationAccountName $AutomationAccountName

# Generating the RunCommand script
@"
Import-Module "C:\Program Files\Microsoft Monitoring Agent\Agent\AzureAutomation\7.3.1417.0\HybridRegistration\HybridRegistration.psd1"
Add-HybridRunbookWorker -GroupName "HybridWorkers" -Url $($RegistrationInfo.Endpoint) -Key $($RegistrationInfo.PrimaryKey)
"@ | Out-File Script.ps1

# Loop the VMs and register them as User Workers
foreach ($VM in Get-AzVM -ResourceGroupName $ResourceGroupNameWorker) {
    $null = Invoke-AzVMRunCommand -ResourceGroupName $ResourceGroupNameWorker -VMName $VM.Name `
    -ScriptPath ".\Script.ps1" -CommandId "RunPowerShellScript" -AsJob
}
# Cleaning up RunCommand Script
Remove-Item ".\Script.ps1" -Force
{% endhighlight %}

This script will register all VMs in a Resource Group ($ResourceGroupNameWorker) as User Workers, using the RunCommand in Azure. The Custom Script extension could have been used as well, in fact the Custom Script Extension would be required if the VMs where on-premises VMs using Azure Arc for Servers.

## Schedules
A Schedule is a construct that can trigger a certain job at a specific time, the job being either an update deployment or a Runbook.

An Azure Automation Account uses the Coordinated Universal Time or UTC standard. Although UTC is in sync with GMT timezone, this sometimes makes it difficult to get a job to run on the expected time.

Consider this snippet:

{% highlight PowerShell linenos %}
$AutomationAccount = Get-AzAutomationAccount
New-AzAutomationSchedule -AutomationAccountName $AutomationAccount.AutomationAccountName `
        -ResourceGroupName $AutomationAccount.ResourceGroupName `
        -Name "Schedule01" -StartTime 20:00 -OneTime -TimeZone UTC
{% endhighlight %}

One would expect the schedule to be at 20:00 UTC or 08:00 PM UTC - but this is what happened:

{% highlight PowerShell linenos %}
StartTime              : 24/04/2022 18.00.00 +00:00
ExpiryTime             : 24/04/2022 18.00.00 +00:00
IsEnabled              : True
NextRun                : 24/04/2022 18.00.00 +00:00
Interval               : 
Frequency              : Onetime
MonthlyScheduleOptions : 
WeeklyScheduleOptions  : 
TimeZone               : Etc/UTC
ResourceGroupName      : rg-weu-AutomationPlatform-demo-001
AutomationAccountName  : aut-weu-AutomationPlatform-demo-001
Name                   : Schedule01
CreationTime           : 24/04/2022 17.27.52 +02:00
LastModifiedTime       : 24/04/2022 17.46.14 +02:00
Description            :
{% endhighlight %}

To make this work properly you will have to take the UTC/GMT Offset into account, including the Daylight Saving Time (DST):

{% highlight PowerShell linenos %}
$AutomationAccount = Get-AzAutomationAccount
$StartTime = (Get-Date 20:00).AddHours([System.TimeZoneInfo]::Local.GetUtcOffset((Get-Date)).TotalHours)
New-AzAutomationSchedule -AutomationAccountName $AutomationAccount.AutomationAccountName `
        -ResourceGroupName $AutomationAccount.ResourceGroupName `
        -Name "Schedule01" -StartTime $StartTime -OneTime -TimeZone UTC
{% endhighlight %}

Notice line 2, where the UTC offset is added. The result is now the expected schedule:

{% highlight PowerShell linenos %}
StartTime              : 24/04/2022 20.00.00 +00:00
ExpiryTime             : 24/04/2022 20.00.00 +00:00
IsEnabled              : True
NextRun                : 24/04/2022 20.00.00 +00:00
Interval               : 
Frequency              : Onetime
MonthlyScheduleOptions : 
WeeklyScheduleOptions  : 
TimeZone               : Etc/UTC
ResourceGroupName      : rg-weu-AutomationPlatform-demo-001
AutomationAccountName  : aut-weu-AutomationPlatform-demo-001
Name                   : Schedule01
CreationTime           : 24/04/2022 17.27.52 +02:00
LastModifiedTime       : 24/04/2022 17.52.19 +02:00
Description            :
{% endhighlight %}

As each Schedule can be defined in its own timezone, the scheduling can get very complicated fairly fast.<br>
My recommendation would be to keep schedules in UTC, especially if you have to schedule jobs across timezones - this makes scheduling simpler and more "human readable".

A Schedule can be onetime, or recurring on a Hourly, Daily, Weekly or Monthly basis. The recurring schedules can be configured to be on specific weekdays, specific days of month, etc. This should make it possible to configure most of the recurrences imaginable.<br>
If that is not suficient, then a Schedule can be linked to zero or more Runbooks and a Runbook can likewise be linked to zero or more Schedules, which makes it possible to use 4 hourly Schedules to run a Runbook each quarter hour. So by combining multiple schedules, I would think that all possible requirements for scheduling and recurrence can be met.

The time of day the schedule is set to run, is the same as the start time. This means that a hourly schedule with a start time of 12:15 will run hourly at 15 minutes past. If the schedule is Daily, Weekly or Monthly, the time would be 12:15 at the specific day.

Potential parameters for the Runbooks are defined in the Schedule/Runbook link, a topic which is outside the scope of this article.

## Update Management
Another important part of the Azure Automation Account featureset is the Update Management feature, which as implied above works with the System Workers. This means that if a VM is associated with a Log Analytics Workspace, that has an Automation Account linked, the VM will be assessed for updates using the update source defined on that specific VM - normally that would be someting like Windows Update or WSUS.

So about ½ an hour after a VM is associated to the Log Analytics Workspace, the missing updates on the VM will be included in a list like this:

![UpdateManagement](/assets/images/Articles2022/Articles2022-AutomationAccount-UpdateManagement.png "UpdateManagement"){: .align-center}

There is even a more "state of the complete environment" kind of report available as well:

![UpdateManagement-Report](/assets/images/Articles2022/Articles2022-AutomationAccount-UpdateManagement-Report.png "UpdateManagement-Report"){: .align-center}

This report allows drilling down into details on each and every single VM, making it a very strong reporting tool for Update Management.

Worth to note is that the report is showing live data - That fact has been known to provide interesting discussions with the Security Team right after Patch Tuesday, before the first scheduled service window.<br>
It might take a while to convince a Security Team, used to monthly reporting, that the servers are being patched according to agreement, but that the servers naturally shows as lacking behind until the service windows are reached - that information has probably never been available before.

## Inventory and Change Tracking
The Inventory and Change Tracking features are not directly linked to any Automation, but they are a very nice side effect of the data collected from a System Worker (which as you might remember is any VM associated with a Log Analytics Workspace that has been linked to an Automation Account).

The Inventory feature provides lists of software and services installed and which services are running on a Server - the list below only shows the software information, similar information are available for services including status information for the services.

![Inventory](/assets/images/Articles2022/Articles2022-AutomationAccount-Inventory.png "Inventory"){: .align-center}

When something changes, Change Tracking registers the change like this:

![ChangeTracking](/assets/images/Articles2022/Articles2022-AutomationAccount-ChangeTracking.png "ChangeTracking"){: .align-center}

![ChangeTracking-Software](/assets/images/Articles2022/Articles2022-AutomationAccount-ChangeTracking-Software.png "ChangeTracking-Software"){: .align-right}

It is possible to drill down into the Change Tracking list, to get even more detailed information about what has been changed - nicely presented with before and after values of specific properties. The example shown is the change recorded when the Dependency Agent was installed.

The best part is that all this data is searchable in the Log Analytics Workspace directly using KUSTO - this means that the task of identifying which servers has specific versions of software or services installed becomes a task of minutes, not hours - even without implementing a formal Configuration Management Database (CMDB).

The KUSTO query below lists all Servers, having the "Dependency Agent" installed and the "WindowsAzureGuestAgent" service running - as easy as that.

{% highlight console linenos %}
ConfigurationData
| where SoftwareName contains "Dependency Agent"
| summarize arg_max(TimeGenerated, *)
| union ConfigurationData
| where SvcName contains "WindowsAzureGuestAgent" and SvcState contains "running"
| summarize arg_max(TimeGenerated, *)
| project Computer
{% endhighlight %}

## Runbooks
High-level 2 types of Runbooks exists, Python and PowerShell.<br>
The PowerShell type Runbook again have 2 subtypes, PowerShell Workflow and PowerShell Script, both of which have a Graphical subtype.

I will not comment anymore on the graphical types, as they in my view are a poor attempt at mimicking some functionality in PowerApps, just without the cleverness and usability of PowerApps.

A Runbook is able to handle parameters and can be scheduled to run by linking the Runbook to a Schedule - potential parameters are defined in the link.

### Python Runbooks
Azure Automation supports currently both Python 2 and Python 3 in both the sandboxed and Hybrid Worker environments. Even Python libraries from 3rd parties are supported, the packages just have to be imported into the Automation Account or Hybrid Worker.

Python is not a native Microsoft language however and it still shows in the limitations and strange behaviour department. E.g. sys.stderr is not supported by Azure Automation Accounts, which means that special care needs to be taken with exception handling.

### PowerShell Runbooks
At the time of this article, PowerShell Core (7.x) is in preview on Azure Automation. This means that there are certain limits to what works and what does not - most notable limits are the missing Source Control integration and Runbook signing for PowerShell Core Runbooks.

PowerShell 5.1 have been fully supported by Azure Automation for a long time, so for now consider using this version untill PowerShell Core moves out of preview.

I am working on an article specifically on how to write a good Runbook for Azure Automation, I will link to this article here when it is ready.

## Shared Resources
When writing Runbooks, it becomes nessesary to handle some resources that are to be available across Runbooks. These shared resources can be:
- Certificates
- Credentials
- Connections
- Variables

Certificates can be used to make Connections, based on Service Principals and defined in the Connections shared resource.<br>
Credentials are a simple username/password thing.<br>
Variables are just simple variables that comes in the most common types like String, Boolean, Integer and DateTime. And added perk with variables are that they can be encrypted, which means that they can be used to store secret data like API keys and the like in a secure fasion.

So what the shared resources provide, is a secure shared repository of resources accessible from within the Runbooks, making the processes of rotating keys, renewing certificates, updating API keys, etc. much more simple and secure.

# Enterprise Scale Context
To paint the big picture, putting Azure Automation into an Enterprise Scale context, then this is how it looks like in the resource structure:

![Enterprise Scale](/assets/images/Articles2022/Articles2022-Automation%20Platform%20Enterprise%20Scale.png "Enterprise Scale"){: .align-center}

When building Enterprise Scale following the guidelines and recommendations from the [Microsoft Cloud Adoption Framework](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/ "Microsoft Cloud Adoption Framework") the concept of "Governance by Policy" is one of the themes - it is a really good idea to think Policies into the Governance and Operations, but unfortunately Policies are not the solution to everything and thats where Azure Automation comes into play.

The obvious limitations for Policies are the Update Management, Change Tracking and Inventorying described above, but also verification of tag values like costcenters, application owners and such things are outside the realm of Policies.

By using Service Principals for connecting to things, the existing RBAC models are easily applied and the Runbooks are able to work on the entire infrastructure, also the on-premises parts using Azure Arc. Runbooks can even operate across tenants.

Viewed from another angle, when an Application team is working, you want them to focus on the application platform that they are developing, not on Update Management and similar things - Use the Azure based Automation Platform for that.

# Key Takeaways
- The Azure based Automation Platform are able to automate VM management across the hybrid platform and provide valuable insights.
- The Azure based Automation Platform are able to automate interactions with APIs - like the Azure Resource Manager APIs or APIs provided by ITSM platforms.
- The Azure based Automation Platform are good for automating governance, filling in where Azure Policies does not do the job.
- The Azure based Automation Platform works perfectly across tenants.
- The Azure based Automation Platform works great with Source Control platforms like Azure DevOps and GitHub, providing scaleability.