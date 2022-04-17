---
classes: wide
title: "Azure: Well Architected Resource Structure"
header:
  og_image: "/assets/images/ArticleLogos/2022-04-14-Azure-Well-Architected-Resource-Structure.jpg"
excerpt: "Enterprise Scale Landing Zone, Landing Zones, Resource Structure design patterns and how to handle connectivity - when the datacenter becomes Azure Cloud based."
categories:
  - Article
tags:
  - Architecture
  - Level 100
---
*Enterprise Scale Landing Zone, Landing Zones, Resource Structure design patterns and how to handle connectivity - when the datacenter becomes Azure Cloud based.*

If we start out from the beginning, in ancient times, around the year 2015, businesses started to talk about cloud and digitalization for real and the need for both in order for the businesses to survive in an increasingly competitive market.

At that time, the buzzwords for why cloud should be used, was:
> Agility, Flexibility and Cost

As things matured a bit and calmer minds prevailed people figured out that Agility and Flexibility really was a way of working and organizing - so the [SAFe Framework](https://www.atlassian.com/agile/agile-at-scale/what-is-safe) gained popularity. It also turned out that Cloud was not necessarily cheap to use - but if Cloud was used in the right way, it could dramatically increase the business value of IT and in turn be very much worth the effort.

At present day the buzzwords has become more like:
> Time to Market, Features, Scalability and Security

The bottom line is, Cloud took off and is fast becoming the place where all new IT platforms ends up. In other words the traditional central datacenters are moving out, looking more and more like a specialized branch office while the Cloud takes the center stage becoming the digital hub for the business.

![Datacenter Evolution](/assets/images/Articles2022/Articles2022-Datacenter.png "Datacenter Evolution")

Now, this might lead to the perception that Cloud should be handled just like the traditional central datacenter - Cloud is after all replacing it.

Nothing could be more wrong. If Cloud is handled traditionally, Cloud will just end up being a very expensive traditional datacenter which is not really where anyone should want to go. However, this is exactly the direction in which many organizations ends up moving.

The rest of this article will focus on one of the key elements for using Azure Cloud in a Secure, Scalable way and at the same time maximizing the business value: **The Well Architected Resource Structure**

Note: I am leaning heavily on the [Microsoft Cloud Adoption Framework (CAF)](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/), as this is a huge collection of recommendations and practices, tested by reality.

# Landing Zones
Like the classic datacenters were build using secure buildings and rows of racks containing physical servers, Cloud "datacenters" are build using Enterprise Scale Landing Zones and groups of Landing Zones containing resources.<br>
If used in the right way, a Landing Zone can bring freedom and flexibility to the application teams - hence contribute significantly to minimizing time to market for the individual applications and platforms.

High-level, there are 4 types of landing zones:

| | **Type** | **Characteristics** |
| ![Enterprise Scale Landing Zone](/assets/images/Articles2022/Icons/Azure.png) | Enterprise Scale Landing Zone | - Cloud Datacenter<br> - Contains the other Landing Zones<br> - Tenant wide governance |
| ![Online Landing Zone](/assets/images/Articles2022/Icons/Azure-Subscription.png) | Online Landing Zone | - Not connected to the corporate network<br> - Contained in a Subscription<br> - Contains an application or a platform<br> - Deployments are pipeline based |
| ![Connected Landing Zone](/assets/images/Articles2022/Icons/Azure-Subscription.png) | Connected Landing Zone | - Connected to the corporate network<br> - Contained in a Subscription<br> - Contains an application or a platform<br> - Deployments are pipeline based |
| ![Classic Landing Zone](/assets/images/Articles2022/Icons/Azure-ResourceGroup.png) | Classic Landing Zone | - Connected to the corporate network<br> - Contained in a Resource Group<br> - Contains many applications or platforms<br> - Deployments are traditional ITSM based |

## Enterprise Scale Landing Zone
The Enterprise Scale Landing Zone is not really a landing zone like the other three. It is more of an architectural concept containing the structure, processes and supporting services, which makes it possible to maintain a sufficient level of security and governance while supporting scalability and at the same time enable the business to maximize the value of Cloud.<br>
A central part of the Enterprise Scale Landing Zone is the resource structure, which is where the other types of landing zones resides.

A well governed Enterprise Scale Landing Zone is a prerequisite for providing safe and flexible Landing Zones for applications.

## Online Landing Zone
Typical for the Online Landing Zone is that the contained platform does not require private network access outside of the landing zone. This means no corporate network access, no RDP or SSH to VMs inside the landing zone, etc.
This type of landing zone are sometimes referred to as a "Cloud Native" landing zone and is typically seen in connection with serverless platforms, but also sometimes used by data scientists in their work.
Usually an Online Landing Zone is used in pairs or triplets, like Development, Test and Production. Deployments are usually pipeline based and fully automated.

## Connected Landing Zone
Typical for the Connected Landing Zone is that the contained platform requires access to the corporate network outside of the landing zone. This could be RDP or SSH access to servers, or it could be access to SQL backends using private endpoints or even on-premises platforms requiring private network access.
This type of landing zone typically are either not completely serverless or are using private links/endpoints bringing PaaS components into private subnets.
Usually a Connected Landing Zone is used in pairs or triplets, like Development, Test and Production. Deployments are usually pipeline based and fully automated.

## Classic Landing Zone
It is a stretch to call a Classic Landing Zone a landing zone - basically this is a Resource Group, with an associated subnet fenced off by a Network Security Group.<br>
The Classic Landing Zone is contained in a Connected Landing Zone, and are sharing the Connected Landing Zone with other Classic Landing Zones.
This type of landing zone is used to accommodate less technical savvy people, that just needs a server that someone can install a piece of software on using the traditional methods - "MSI file, next, next, finish".

# Resource Structure
As mentioned above, a central part of the Enterprise Scale Landing Zone is the resource structure - one could wonder why that is so, as the individual landing zones seems to be more or less self contained.<br>
This is where the governance, security and connectivity comes into play - those things are best handled globally, meaning Enterprise Scale and are as such dependent on the resource structure.

For the sake of simplicity I will limit the description of the resource structure to only include these components:

| | **Component** | **Type** | **Description** |
| ![Management Group](/assets/images/Articles2022/Icons/Azure-ManagementGroup.png) | Management Group | Structural | - Contains: Management Groups, Subscriptions<br> - Role: Policy Anchor, RBAC Scope |
| ![Subscription](/assets/images/Articles2022/Icons/Azure-Subscription.png) | Subscription | Structural | - Contains: Resource Groups<br> - Role: Policy Anchor, RBAC Scope, Billing Scope |
| ![Resource Group](/assets/images/Articles2022/Icons/Azure-ResourceGroup.png) | Resource Group |  Structural | - Contains: Resources<br> - Role: RBAC Scope, Contains resources with similar lifecycle and purpose |
| ![Azure Policy](/assets/images/Articles2022/Icons/Azure-Policy.png) |  Azure Policy | Supporting | Supports governance by either enforcing or auditing resource deployments |
| ![Tag](/assets/images/Articles2022/Icons/Azure-Tag.png) | Tag | Supporting | Supports governance by "adding an extra dimension" to the resource structure, like cost centers, owners, application names, etc. |
| ![Naming Convention](/assets/images/Articles2022/Icons/Azure-NamingConvention.png) | Naming Convention | Supporting | Supports governance by making navigating the resource structure easier and more "human readable" |

The Structural Components are the actual resource structure, whereas the Supporting Components supports the effort of not making a mess of things when using and scaling the resource structure over time.

Most of the successful implementations of a resource structure that I have seen, follows the basic guidelines from the CAF, with one special area deviating from the standard.<br>
The successful resource structures looks like this - with the deviating area highlighted:

![Resource Structure](/assets/images/Articles2022/Articles2022-Resource%20Structure.png)

One could argue that it is the most complex part of the resource structure that deviates from the standard - fortunately the deviations tends to fall into 3 patterns or a combination of those.

| **Pattern** | **Characteristics** |
| Geographic | - Used when scaling out geographically, when services are to be deployed in specific regions<br> - Policy limits region deployments<br> - Normally seen in the Connected Landing Zone area |
| Environment | - Used when the environment type (dev, test, prod) are the significant factor<br> - Policy used for auditing or enforcing compliance<br> - Normally seen in the Online Landing Zones area |
| Vendor | - Used when multiple vendors are used<br> - Used as RBAC Scope, for ease of IaM when isolating competing vendors<br> - Seen both in Online Landing Zones and Connected Landing Zones areas |

## Geographic Pattern
![Geographic Pattern](/assets/images/Articles2022/Articles2022-Geographic%20Pattern.png){: .align-right}

The Geographic Pattern has its strength when scaling infrastructure to more than one region across the world, which is also why it is usually seen in the Connected Landing Zone area.<br>
In most of these scenarios, Azure Policy are used to force deployments of resources to the right regions - hence making sure that all resources in a landing zone are deployed to the same geographical region.<br>
In this example, the regions West Europe (WEU), East US (EUS) and South East Asia (SEA) are visualized.

If the pattern are used in the Online Landing Zone area, it will often increase the complexity, especially if the landing zones contains web applications - these components tends to be deployed in multiple regions, e.g. web application in WEU and CDNs in other regions.

## Environment Pattern
![Environment Pattern](/assets/images/Articles2022/Articles2022-Environment%20Pattern.png){: .align-left}

The Environment Pattern is mostly used in connection with implementing compliance requirements for landing zones. In this way using Azure Policy, development and test environments can be audited on e.g. NIST or ISO 27001 while the corresponding production environments are enforced on the same standards.

This means that developers are able to verify compliance requirements in development and test environments, without getting bogged down with deployments being blocked due to policy violations - while at the same time ensuring that production environments are compliant.

## Vendor Pattern
![Vendor Pattern](/assets/images/Articles2022/Articles2022-Vendor%20Pattern.png){: .align-left}

The Vendor Pattern is used in situations where an organization uses multiple vendors to develop, build, deploy and operate different landing zones. Basically it utilizes the fact that Azure Policies and RBAC Scopes are inherited down through the resource structure.<br>
In this pattern, the "Vendor#" Management Group layer are purely used for RBAC Scoping, but as the Azure Policies are inherited from the higher layers in the resource structure each vendor are subjected to the same policies and restrictions.<br>
Another feature of this pattern is that the different vendors are not able to see each other, hence the opportunity for "friendly competitive" interactions are very limited.  

## Mixed Pattern
![Mixed Pattern](/assets/images/Articles2022/Articles2022-Mixed%20Pattern.png){: .align-right}

The Mixed Pattern is not really a single pattern, but more a mixture of 2 of the other patterns.<br>
In the example shown, the Vendor Pattern are mixed with the Geographical Pattern, allowing for multiple vendors developing, building, deploying and operating different landing zones across multiple regions in the world.

If the Vendor Pattern are mixed with the Environment Pattern, the same compliance requirements for both audit and enforce can easily be applied across vendors.<br>
In the Azure resource structure we have 6 layers of Management Groups to work with - which gives plenty of opportunities to mix patterns to achieve what is required, which should be a more simple, uniform governance and configuration across the entire resource structure.

# Connectivity
By implementing a well architected resource structure in the Enterprise Scale Landing Zone, we now have an Enterprise Scale Landing Zone that we are able to scale and maintain, by adding, removing and maintaining the Landing Zones for the individual applications and platforms.<br>
The next step is to design a way for the Landing Zones to talk to each other and the on-premises world using a private network, without compromising the capacity to scale and govern the entire thing. Landing Zones doesn't really talk much, but resources deployed within them does and that needs to be handled<br>
Networks in Azure are build using the resource type Virtual Networks also known as VNets. A VNet is scoped inside a subscription, meaning that only resources deployed in the same subscription as the VNet can use that specific VNet - in other words, a VNet is scoped to a Landing Zone.<br>
To tie the individual landing zones together, I would highly recommend using Azure Virtual WAN - which will handle most of the tedious details of networking that would otherwise increase the complexity.

The figure below zooms into a slice of the resource structure, with two landing zones (App3 and App5) connected using Azure Virtual WAN:

![Connectivity](/assets/images/Articles2022/Articles2022-Connectivity.png)

| | **Resource Type** | **Purpose** |
| ![Subscription](/assets/images/Articles2022/Icons/Azure-Subscription.png) | Subscription | Landing Zone, RBAC Scope |
| ![Resource Group](/assets/images/Articles2022/Icons/Azure-ResourceGroup.png) | Resource Group | RBAC Scope |
| ![Azure Virtual WAN](/assets/images/Articles2022/Icons/Azure-VirtualWAN.png) | Azure Virtual WAN | Azure WAN resource, providing WAN and connectivity as a service |
| ![Azure Virtual WWAN Secure Hub](/assets/images/Articles2022/Icons/Azure-VirtualWAN-SecureHub.png) | Azure Virtual WAN Secure Hub | Secure Hubs providing firewalled connectivity from the landing zones |
| ![Virtual Network](/assets/images/Articles2022/Icons/Azure-VNet.png) | Virtual Network | Azure network resource providing virtual networking in the landing zones |

The shadowed areas of the figure illustrates RBAC Scopes. As shown, the Connectivity subscription is in a separate RBAC Scope than the individual landing zones - this means that it becomes possible to delegate subnetting and things like Network Security Groups (NSGs) to the application teams, hence the application specific parts of networking becomes part of the application as well and are deployed and handled as such.<br>
There are 3 facts that makes this possible, without loosing control with the corporate network:
1. When a VNet is connected to Azure Virtual WAN, the "dangerous" configurations becomes locked so things like IP scopes cannot be changed - hence configuration errors on networks does not propagate outside of the landing zones.
2. The connections to Azure Virtual WAN are configured in the Connectivity RBAC scope, this is where the network team resides and have control.
3. All traffic that leaves the connected landing zones are firewalled before entering the Azure Virtual WAN backend - the firewall also resides in the Connectivity RBAC scope.

This means that all the staying in control are left with the Connectivity team, all the flexibility are handed over to the landing zone/application teams. If Azure Firewall Premium is used to secure the Virtual Hubs, IDS/IPS with full Sentinel integration can be implemented to monitor all traffic leaving a landing zone, in a sort of "a modern micro-segmentation".

The beauty of it all is that Azure Virtual WAN integrates Azure Firewall directly in the Virtual Hubs as well as Virtual Network Gateways (VPN/IPSec) and Express Route Gateways (MPLS) for connectivity outside of Azure.<br>
This makes branch offices, on-premises datacenters and other external connected networks handle almost like just another landing zone in Azure - and remember a landing zone can exist across the world - so this allows for branch office connectivity across the world using Azure as backbone.

# Governance
If we again take a look at the "reason for cloud" buzzwords mentioned in the beginning: Time to Market, Features, Scalability and Security - Ideally we would like to meet these expectations, without loosing control, which puts some requirements in the direction of the governance.<br>

The best way to give freedom, is to delegate responsibility - as much as possible - which leads to these principles:
1. You own it, You build it, You deploy it
2. If you own the landing zone, you own the cost

If we put the principles in context of the real world, it means that a Product Owner assisted by an Architect owns a landing zone and the platform/application that resides in the landing zone. These two are also responsible for building and deploying the platform, the full stack of the platform including any network, servers and other infrastructure components required.<br>
Guidelines for how to architect a Cloud based platform/application can be found in the [Well Architected Framework](https://docs.microsoft.com/en-us/azure/architecture/framework/)

If responsibility is just delegated without any principles or guardrails, the 2 first buzzwords (Time to Market and Features) would be achieved, potentially also Scalablity, but Security probably not so much. And that is even before someone has to pay for the consumption.<br>

This is where the Supporting Components mentioned earlier comes into play:

| | **Component** | **Type** | **Description** |
| ![Azure Policy](/assets/images/Articles2022/Icons/Azure-Policy.png) |  Azure Policy | Supporting | Supports governance by either enforcing or auditing resource deployments |
| ![Tag](/assets/images/Articles2022/Icons/Azure-Tag.png) | Tag | Supporting | Supports governance by "adding an extra dimension" to the resource structure, like cost centers, owners, application names, etc. |
| ![Naming Convention](/assets/images/Articles2022/Icons/Azure-NamingConvention.png) | Naming Convention | Supporting | Supports governance by making navigating the resource structure easier and more "human readable" |

## Azure Policy
Azure Policy is a very large topic, so I will focus on how they can support maintenance of the resource structure.<br>
The recommended way of structuring policies is to group policies with similar purpose and scope into initiatives which is then assigned to a place (scope) in the resource structure. An initiative can be assigned to Management Groups, Subscriptions and Resource Groups.

At the top of the resource structure the policies assigned should be general policies, with a broad scope - this could be logging policies, ensuring that relevant logs are collected across the resource structure. There is no need to assign this kind of policies to all subscriptions, if they can be assigned to the Tenant Root Group management group - generally speaking.

The further down the resource structure we move, the more specialized the policies should become. This could be allowed location policies, ensuring that resources in specific parts of the resource structure are deployed in a specific location, e.g. when the Geographical Pattern are used. If this kind of policy is assigned at the top of the resource structure, a lot of exceptions would probably have to be implemented - hence increasing the complexity and minimizing the readability of the resource structure.

If policies are used correctly, you can to a great extend control which resources are deployed where and ensure that possible compliance requirements are met - and at the same time give the Application/Product owners extended freedom to develop their platforms with a minimal time to market.

## Tag
A Tag is a key/value pair that can be used to label resources, resource groups and subscriptions - and they can be force inherited down through the resource structure using Azure Policy.<br>
Two tags that are almost always recommended are: ProductOwner and ApplicationName. These tags will add information to the resource structure, about who is owning the specific parts like landing zones and what the application inside that landing zone is called. If these tags are force inherited all the way to the resource level, each and every single resource will be labelled with the tags - hence being easier to identify, which in turn is very useful when doing Cost Management and Automation.

Inheritance and enforced tagging are configured using Azure Policy - which means that all deployments of resources and resource groups will inherit tags from their respective parents at deployment time.

## Naming Convention
In general, I am not a very big fan of naming conventions. They tends to require a lot of effort to define and over a fairly short amount of time the conventions will become less and less fit for purpose. I have also seen some horrific examples of naming conventions designed to "look great when listing resources in Excel".

A few rules can, however, be a great support to the governance of an Enterprise Scale Landing Zone:

| **#** | **Rule** | **Reason** |
| **1** | A VNet inside a Connected Landing Zone are given the same name as the Subscription containing that specific landing zone | This minimizes confusion when the landing are to be handled, Application owners typically knows the Subscription name but do not care about the network - opposite is true for the network team |
| **2** | A Resource Group associated with a Subnet are given the same name (same goes for a potential NSG on the subnet), apart from the prefix rg- for Resource Group, snet- for Subnet and nsg- for the Network Security Group | This minimizes confusion when deploying into landing zones with many subnets and resource groups, also makes it easier to identify the correct NSGs when investigating connectivity problems |
| **3** | An Azure Policy Initiative Assignment are named after the pattern apia-[AssignmentScope]-[Purpose] | Makes it easy to identify assignments on different scopes, hence making maintenance easy |
| **4** | A Virtual Machine resource are named like to operating system hostname that are deployed in the Virtual Machine | Avoids confusion when server administrators talk to Azure engineers - they need to have the same name to identify the resource/vm |

These basic rules, are just that - basic - and they are focused on the resource structure and Enterprise Scale governance of said resource structure. Depending on the situation the naming convention should be extended to cover the requirements. However, there is no need to overdo the naming convention - it will just have to be maintained and enforced, which potentially uses a lot of time that could be better spend in other places.

## Governance Owner (CCoE)
During a Cloud Journey - which will never end because technology keeps evolving - differences of opinions across the organization will arise as well as different maturity levels will be encountered. To avoid getting stuck in committee discussions, an organization needs one place to go, a CCoE with a strong mandate.

As the name implies - CCoE = Cloud Center of Excellence - this will be the go to place for direction, advice and knowledge, the CCoE can even act as arbiter when technical disputes needs resolving.

Primary responsibility for the CCoE are to support the entire organization in implementing the digitalization strategy and adopting new technologies. This means that the CCoE owns the Enterprise Scale Landing Zone and the surrounding governance and maintenance.

Specific areas like Identity and Access Management can be delegated to other teams, but in my mind the CCoE is responsible for IaM to be implemented in the Enterprise Scale Landing Zone and that it is fit for purpose.

# Key Takeaways
- A well governed Enterprise Scale Landing Zone is a prerequisite for providing safe and flexible Landing Zones for applications
- A Well Architected Resource Structure forms the foundation for the Enterprise Scale Landing Zone, Governance and Connectivity
- An Enterprise Scale Landing Zone is not a Landing Zone, it contains Landing Zones and may as such be considered "a Cloud based datacenter"
- A Cloud Center of Excellence (CCoE) should own the Enterprise Scale Landing Zone, hence the CCoE should also own the resource structure


