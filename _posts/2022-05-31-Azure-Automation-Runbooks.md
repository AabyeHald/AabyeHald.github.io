---
classes: wide
title: "Azure Automation - Runbooks"
header:
  og_image: "/assets/images/ArticleLogos/2022-05-31-Azure-Automation-Runbooks.jpg"
excerpt: "How to write a very basic Runbook with comments, using Source Control to publish the Runbook to Azure Automation and GitHub Workflows to publish the Runbook comments to a Wiki,"
categories:
  - Article
tags:
  - Engineering
  - Level 200
---
*How to write a very basic Runbook with comments, using Source Control to publish the Runbook to Azure Automation and GitHub Workflows to publish the Runbook comments to a Wiki.*

{% capture notice %}
**Associated Repositories and Articles**<br>
*Lab/Demo Environment:* [Article-2022-05-AzureAutomation-Runbooks](https://github.com/AabyeHald/Article-2022-05-AzureAutomation-Runbooks)<br>
*Related Articles:* [Azure Automation - The Basics]({% post_url 2022-04-30-Azure-Automation-The-Basics %})
{% endcapture %}
<div class="notice">{{ notice | markdownify }}</div>

Over time, I have seen a lot of really strange looking runbooks and some fairly complex configurations around working with the runbooks, getting the runbooks updated and monitored. Sometimes it seems that a Windows Scheduler Scripts has just been moved to an Azure Automation Runbooks without any further thoughts into the finer details.

In this article, I will give my perspective on how Runbooks should be handled.

For that purpose, I have chosen to focus on a category of Runbook that are used to manipulate the Azure APIs, supporting Enterprise Scale governance as well as doing som "classic reporting" where that is still required.<br>
Typical tasks for such runbooks are:

- Tag value validation, as Azure Policy will handle the key validation, but comes up short with the values.
- Start/Stop VMs.
- Identify and flag unused resources.
- Classic reports on Subscriptions, Cost, amount of resources, etc.

The list goes on and on and contains basically everything you need to do more than once, that cannot be handled by Azure Policy.<br>
And before you wonder too much - yes, the classic reporting is still a thing that are needed. Now a days, it is moving into connecting older systems with a more realtime world in cloud. Sometimes when management wants a monthly report, they do not want a live dahsboard.

So lets dive into how such runbooks might look.

# The Base Runbook
To be able to work with Runbooks, I find that everything is more smooth if the following items are considered:

- Runbooks stays quit, when executed normally.
- Errors that prevents Runbooks from reaching a desired result, should be terminal and fail the Runbook.
- A Runbook should be able to handle concurrent execution and should not interfere with other Runbooks.
- A Runbook should be executable from a local terminal as well as from Azure Automation without changing anything.

A template for such a Runbook is available in the associated ```Lab/Demo Environment``` referenced at the top of this article.

First, lets look at the initialization of the Runbook:

{% highlight powershell linenos %}
  # Set the Verbose preference - this will make the script stay silent when run as Runbook
  $VerbosePreference = "Continue"

  # Make all errors terminating, unless otherwise specified individually
  $ErrorActionPreference = "Stop"

  # Make sure context is scoped to process
  Write-Verbose -Message "$(Get-Date -Format FileDateTimeUniversal) `t- Initializing Runbook, disabling credential caching"
  $null = Disable-AzContextAutosave -Scope Process
{% endhighlight %}

Line 2 makes sure that the script does not squawk when run as a Runbook, this makes the streams used for Runbook output cleaner and more usable for actual Runbook output. Line 8 outputs an actual verbose message, either when the Runbook is executed in edit mode og locally.<br>
Line 5 makes all script errors stopping errors, terminating script execution. This preference can be overwritten when needed in the script, but setting it globally makes it more likely that errors that slips through are actually handled in the script.<br>
Line 9 prevents sessions and contexts to be shared across processes, hence minimizing the risk of concurrency errors, when multiple Runbooks are executed simultaneously. Also line 9 shows how to silence a cmdlet that returns a value to the output stream, which again cleans up the output streams.

Now, lets look at the code that connects to the Azure API:

{% highlight powershell linenos %}
  # Figure out if we are running in Azure Automation and connect accordingly
  Write-Verbose -Message "$(Get-Date -Format FileDateTimeUniversal) `t- Connect to Azure Resource Manager"
  if ($PSPrivateMetadata.JobId.Guid) {
      # We are in Azure Automation
      Write-Verbose -Message "$(Get-Date -Format FileDateTimeUniversal) `t- We are running in Azure Automation"
      if ($ConnectionName) {
          # Connect using Azure Automation Connection
          Write-Verbose -Message "$(Get-Date -Format FileDateTimeUniversal) `t- Retrieve connection $ConnectionName"
          $Connection = Get-AutomationConnection -Name $ConnectionName
          Write-Verbose -Message "$(Get-Date -Format FileDateTimeUniversal) `t- Connecting to Azure tenant: $($Connection.TenantID)"
          $null = Connect-AzAccount -ServicePrincipal `
              -Tenant $Connection.TenantID `
              -ApplicationId $Connection.ApplicationID `
              -CertificateThumbprint $Connection.CertificateThumbprint    
      }
      else {
          # Connect using Azure Automation Managed Identity
          Write-Verbose -Message "$(Get-Date -Format FileDateTimeUniversal) `t- Connect using Managed Identity"
          $null = Connect-AzAccount -Identity
      }
  }
  else {
      # We are local - make sure we have a connection
      Write-Verbose -Message "$(Get-Date -Format FileDateTimeUniversal) `t- We are running local"
      $null = Get-AzAccessToken -ErrorVariable ResultError -ErrorAction SilentlyContinue
      if ($ResultError) {
          Write-Verbose -Message "$(Get-Date -Format FileDateTimeUniversal) `t- We do not have a connection, fix that"
          $null = Connect-AzAccount
      }
  }
{% endhighlight %}

Line 3 checks for a JobId, which will only be available when run as a Runbook, hence we do not run locally. If we are a Runbook, use a named connection if any is defined, otherwise go with the managed identity of the Automation Account.<br>
Line 22 and onwards handle the local connection. If we have a connection, use that - If we do not have a connection, make one.

By adding code to the ```Demo-RunbookTemplate.ps1``` it is easy to actually do stuff with the connection created.<br>
Almost all of my Runbooks, follow a similar recipe.

## How to Develop Runbooks
One of the key elements of the Runbook template shown above, is the way the Runbook handle connections based on the execution environment. This makes it possible to execute the Runbook locally from a Windows client, using personal credentials, without having to change anything in the Runbook.<br>
Usually I will also parameterise potential Azure Automation variables and secrets, making it possible to pass these items to the Runbook from the local terminal as well. Then the same principles apply: detect the execution environment, then act accordingly - without the need to change the Runbook.

The result of doing it this way is:
- Ability to develop and test locally.
- Target (dev/test/prod) environments selected at connection time.
- No delaying Source Control for Dev/Test, but all benefits of Source Control available for Production.

When working with multiple tenants, the TenantId of the target tenant could be passed as parameter as well.

# Runbook Monitoring
![Job Status](/assets/images/Articles2022/Article2022-AzureAutomation-JobStatus.png "Job Status"){: .align-right}

When working with Runbooks and the Azure Automation Platform there are some nice visualizations showing the Runbooks Job statuses, like how many are completed, running, failed, etc. during the last 24 hours.

Unfortunately this does not show the individual Runbook - However, the Job Log does that

![Job Log](/assets/images/Articles2022/Article2022-AzureAutomation-JobLog.png "Job Log"){: .align-center}

The Job Log allows for drilling down into the individual jobs and see the content of the Job Streams for the individual Runbook Job

![Job Streams](/assets/images/Articles2022/Article2022-AzureAutomation-JobStreams.png "Job Streams"){: .align-center}

Unfortunately this is mostly bling and doesn't really scale that well.

If you remember back to the ```Demo-RunbookTemplate.ps1``` described earlier, an error that would render the Runbook unable to accomplish its purpose was a terminating error, hence the Runbook Job will fail.

This failing Runbook Job will show up in the JobLogs in Log-Analytics and can be queried

{% highlight sql linenos %}
  AzureDiagnostics
  | where ResourceProvider == "MICROSOFT.AUTOMATION"
  | where Category == "JobLogs"
  | where ResultType == "Failed"
{% endhighlight %}

This means that Azure Monitor can be used to Monitor for failed Runbook Jobs and handle alerting for these jobs, just like with any other Azure Resource.

Now we can monitor failing Runbook Jobs, so far so good - but what about other types of results that could be interesting. Like the number of items changed by the Runbook or similar tings that will let you know if the Runbook are actually doing something or just faking it.

So, lets take a look at some of the other output options that can be used in a Runbook.

## Runbook Output
The Azure Automation Platform provides a build-in option for handling output from a Runbook Job, the Job Streams. Normally I would put some information about the job in the Output stream, information like:

- Number of tags updated
- Number of emails sent
- Number of application owners validated

The list goes on and on, depending on the Runbook purpose.

As mentioned, the Job Streams are nice and shiny, but the do not scale very well. For the failing Runbook Jobs, Log-Analytics was used to provide scale - but Log-Analytics does not handle complex data from Job Streams very well. Especially not when looking at timeseries data, like number of tags updated over the last 100 jobs.

If this kind of timeseries data are required, or nice to have, I normally use an Azure Storage Account to store csv formatted "job result statistics" in blob storage and then build a PowerBI dashboard on these data.<br>
While this solution might be a bit Gyro Gearloose, it is a simple and cost effective way to achive insights into the successfull jobs.

# Sandbox vs. Hybrid Worker
So, now we have a Runbook - where are we going to execute it? In the Azure Automation Platform, basically 2 options exists:

1. Sandbox
2. Hybrid Worker

The **Sandbox** is baked into the Azure Automation Platform, which makes it easy and fast to use, initially. The flip side is that it is build into the Azure Automation Platorm, which means that it is a shared resource and that it is subject to certain limits in regards to execution time, available resources, etc.

The **Hybrid Worker** is basically a VM, which means that it has to be deployed, monitored and maintained. This makes for a bit more hassle to get started, but if done right you will have full flexibility afterwards.<br>
Nothing is baked in however, so if you need the Az modules you will have to install them, if you need a certificate for a connection, you will have to provide it in the local certificate store, etc.

I would always go for the Hybrid Worker as the flexibility provided far outweighs the complexity of operating it.

# Source Control
First a couple of questions needs answering:

1. What is Source Control?
2. Why should we use it?

Source Control is the practice of tracking and managing changes made to code, i.e. version control, almost like what you will probably be using when working with documents.<br>
Currently, Azure Automation supports three types of source control:

1. GitHub
2. Azure DevOps (Git)
3. Azure DevOps (TFVC)

When given the choice, I would always go for option 1 or 2 as they seem become the defacto standards for Source Control platforms going forward. Most likely GitHub would be the future platform of choice as it has been rumored for some time now that Azure DevOps will be deprecated.

So why should Source Control be used, apart from the obvious benefits of version control? Well, in the context of Azure Automation, some of the benefits are:
- Decoupling of your code editor from Azure Automation
  - Whatever you submit to Source Control will be synced to Azure Automation
- Allowing multible developers to develop and maintain Runbooks for the same environment
  - Using Pull Requests to manage the merging of changes
- Increased Runbook quality
  - Built in support for automatic testing and peer reviewing
- If using SCRUM like processes, PRs can be linked to features
  - Traceability of changes over time

## Branching
Now, Source Control is all well and good, but to make development using Source Control scalable to multible developers some kind of branching strategy will be needed.<br>
Over time I have seen many different attempts on implementing a branching strategy, all the way from some wildly overengineered setups, to something a lot more simple that actually works.

The common denominator for the successfull branching strategies are that they are feature based, separating the development of each feature in separate branches.

![Branching](/assets/images/Articles2022/Articles2022-Branching.png "Branching"){: .align-center}

Some developers argue that this approach potentially will generate an obsene amount of merge conflicts. It turns out however, that most features are linked to one Runbook, which in turn makes merge conflicts mostly avoidable by simple planning - no matter if you are using Scrum, KANBAN or just a plain old ToDo list.

One advantage of using the feature based branching, is that it provides a simple link between the PullRequest, the planning documentation and the release documentation, making the documentation releases closely related to the source code/Runbook releases.

## Documentation
I should probably be one of the first to admit that documentation is not necessarily top of mind when developing cool stuff - And I will never admit that I typically regret this a lot (there might even be tears), when updating some piece of code I wrote a couple of months back.

Fortunately with PowerShell there are kind of a standard that makes commenting scripts easier. It is called [Comment Based Help](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_comment_based_help) and when you get used to it, it is actually quite nice.

Only drawback is that to access the comments, you either have to have the script open and read it directly or you will have to use this:

{% highlight powershell linenos %}
  Get-Help Demo-RunbookTemplate.ps1

  NAME
      Demo-RunbookTemplate.ps1

  SYNOPSIS
      Template Runbook for use with the Azure Resource Manager API.


  SYNTAX
      Demo-RunbookTemplate.ps1 [[-ConnectionName] <String>]    
      [<CommonParameters>]


  DESCRIPTION
      Template Runbook for use with the Azure Resource Manager API.
      The included functionality is:
      - Setting the Verbose preference to "Continue", so the runbook will not squawk during normal operations.
      - Disable credential caching, avoiding potential concurrency issues.
      - Handle local and Azure execution.
      - Authenticate using Azure Service Principal, Managed Identity or if executed locally normal User Credentials.
      - Handle authentication or connection related errors as stopping errors.
{% endhighlight %}

As drawback goes it is not that big, but unfortunately the consequences seems to be that you have to know the script you are searching for before you can get confirmation from the documentation.

A better way to go would be to create something browsable like a Wiki, unfortunately nobody wants to write documentation even if it is in a Wiki.

The combination of the Wiki and the Comment Based Help however is easy to use and browseable. In the ```Lab/Demo Environment``` associated with this article, I have created a very basic example, using GitHub Workflow and the GitHub Repository Wiki. Only configuration is these files:

- .github\workflows\build.wiki.yml
- build\build.wiki.ps1

Basic idea is to use the Get-Help CmdLet to retrieve the comments from the scripts, then dump that into markdown formatted textfiles and commit these to the GitHub Repository Wiki, which is in fact also a git repository.

Looks like this:

![Runbook Wiki](/assets/images/Articles2022/Article2022-AzureAutomation-Wiki.png "Runbook Wiki"){: .align-center}

The righthand sidebar will contain a list of all Runbooks, clickable, browseable, likable.

# Key Takeaways
- Write your Runbooks to be executed in a shared environment, because they probably are.
- Keeping Runbooks silent eases debugging from the Automation Account side of things.
- The use of Source Control makes development scale.
- Comment your code using the Comment Based Code standard and generated a Wiki based on this - browsable comments/documentation is easier to use.