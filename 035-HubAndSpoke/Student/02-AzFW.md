# Module 2: Firewalling Traffic

[< Previous Module](./01-HubNSpoke-basic.md) - **[Home](../README.md)** - [Next Module >](./04-AppGW.md)

## Introduction

In this module you will be fine-tuning your routing design to send VM traffic through the firewall.

## Description

In this module we will deploy an Azure Firewall to the hub VNet, so that you have the topology described here:

![hubnspoke basic](media/hubnspoke-01.png)

We will make sure that the firewall is inspecting all outbound Internet traffic from the Virtual Machines, as well as traffic going from Azure to onprem. Finally, we will direct traffic from the Internet through the firewall for inspection and redirection to the web servers installed on each cloud VM in the last module.

## Method

**_IMPORTANT: Be sure to create ALL resources in Canada Central region._**

**_IMPORTANT: If a parameter is not specified, it should not be set OR the default is acceptable._**

1. Modify the `hub` VNet so that it contains a new subnet for the firewall per the table below:

   | Name  | Address Space | Subnets<br>Name: Address Space                                                               |
   | ----- | ------------- | -------------------------------------------------------------------------------------------- |
   | `Hub` | 10.0.0.0/16   | GatewaySubnet: 10.0.0.0/24 <br>default: 10.0.10.0/24<br>**AzureFirewallSubnet: 10.0.1.0/24** |

1. Create a new Azure Firewall with the following settings:

   - Name: `hub-fw`
   - Tier: `standard`
   - Firewall management: `Use policy`
   - Create a new policy named: `demo-policy`
   - VNet: `hub`
   - Create a new public IP named: `fw-pip`

   The firewall will take between 5-10 minutes to deploy. Once the PIP has deployed, gather the IP and record it.

1. While the firewall is deploying, create four route tables:
   1. Name: `hub-gateway`
   1. Propagate gateway routes: `yes`
   ***
   1. Name: `hub`
   1. Propagate gateway routes: `no`
   ***
   1. Name: `spoke1`
   1. Propagate gateway routes: `no`
   ***
   1. Name: `spoke2`
   1. Propagate gateway routes: `no`
1. Once created, associate the route tables as follows:

   | Route Table   | VNet   | Subnet        |
   | ------------- | ------ | ------------- |
   | `hub-gateway` | hub    | GatewaySubnet |
   | `hub`         | hub    | default       |
   | `spoke1`      | spoke1 | default       |
   | `spoke2`      | spoke2 | default       |

1. Create the following routes in the correct route table:

   `hub-gateway` Route Table

   | Name   | Address prefix | Next hop type     | Next hop IP address |
   | ------ | -------------- | ----------------- | ------------------- |
   | hub    | 10.0.0.0/16    | Virtual appliance | 10.0.1.4            |
   | spoke1 | 10.1.0.0/16    | Virtual appliance | 10.0.1.4            |
   | spoke2 | 10.2.0.0/16    | Virtual appliance | 10.0.1.4            |

   **_Note_**: At this point, traffic from on-premises to the cloud and back will break.

   `hub` Route Table

   | Name    | Address prefix   | Next hop type     | Next hop IP address |
   | ------- | ---------------- | ----------------- | ------------------- |
   | default | 0.0.0.0/0        | Virtual appliance | 10.0.1.4            |
   | spoke1  | 10.1.0.0/16      | Virtual appliance | 10.0.1.4            |
   | spoke2  | 10.2.0.0/16      | Virtual appliance | 10.0.1.4            |
   | fw-pip  | \<PIP of FW\>/32 | Internet          |                     |

   **_Note_**: At this point, traffic in the hub to anywhere will break.

   `spoke1` Route Table

   | Name    | Address prefix   | Next hop type     | Next hop IP address |
   | ------- | ---------------- | ----------------- | ------------------- |
   | default | 0.0.0.0/0        | Virtual appliance | 10.0.1.4            |
   | hub     | 10.0.0.0/16      | Virtual appliance | 10.0.1.4            |
   | fw-pip  | \<PIP of FW\>/32 | Internet          |                     |

   `spoke2` Route Table

   | Name    | Address prefix   | Next hop type     | Next hop IP address |
   | ------- | ---------------- | ----------------- | ------------------- |
   | default | 0.0.0.0/0        | Virtual appliance | 10.0.1.4            |
   | hub     | 10.0.0.0/16      | Virtual appliance | 10.0.1.4            |
   | fw-pip  | \<PIP of FW\>/32 | Internet          |                     |

   **_Note_**: At this point, traffic in the cloud is broken, because the firewall has no rules and is deny all by default.

1. Create a Log Analytics workspace in your resource group called `la-hackathon`.
1. Once the firewall has deployed, navigate to it in the Portal. Click on `Diagnostic Settings` in the left-hand blade, and then click on `+ Add Diagnostic Setting`. Name it `Log Analytics`, then enable `allLogs`, and `Send to the Log Analytics workspace` created above.
1. Go to Azure Firewall Manager, and locate the `demo-policy`.

   Although it is not germane to this exercise, it is worth knowing that Azure Firewall has 3 types of rule collections:

   - Network Rules: These are standard Layer-4 (source/dest/port) rules
   - Application rules: These are Layer-7 aware rules that permit FQDN filtering as well as using Layer 4 flows.
   - DNAT rules: These are better thought of as _inbound_ rules, from the Internet to something in the protected virtual networks

1. Create the following rule collections:

   - Name: `Internal`
   - Collection type: `Network`
   - Priority: `100`
   - Rule collection action: `Allow`
   - Rule collection group: `DefaultNetworkRuleCollectionGroup`
   - Rules:

   | Name           | Source type | Source                              | Protocol | Destination Ports | Destination Type | Destination                         |
   | -------------- | ----------- | ----------------------------------- | -------- | ----------------- | ---------------- | ----------------------------------- |
   | Cloud-ToOnPrem | IP Address  | 10.0.0.0/16,10.1.0.0/16,10.2.0.0/16 | Any      | \*                | IP Address       | 172.16.0.0/16                       |
   | OnPrem-ToCloud | IP Address  | 172.16.0.0/16                       | Any      | \*                | IP Address       | 10.0.0.0/16,10.1.0.0/16,10.2.0.0/16 |
   | Gateways       | IP Address  | 10.0.0.0/24                         | Any      | \*                | IP Address       | 10.0.0.0/24                         |
   | Hub-To-Spoke1  | IP Address  | 10.0.0.0/16                         | Any      | \*                | IP Address       | 10.1.0.0/16                         |
   | Spoke1-To-Hub  | IP Address  | 10.1.0.0/16                         | Any      | \*                | IP Address       | 10.0.0.0/16                         |
   | Hub-To-Spoke2  | IP Address  | 10.0.0.0/16                         | Any      | \*                | IP Address       | 10.2.0.0/16                         |
   | Spoke2-To-Hub  | IP Address  | 10.2.0.0/16                         | Any      | \*                | IP Address       | 10.1.0.0/16                         |

1. At this point, traffic flows should be working correctly and identically from before we put in the firewall. Verify it by:

   - From `onprem-vm`, ping 10.0.10.4 (`hub-vm`), 10.1.10.4 (`spoke1-vm`), and 10.2.10.4 (`spoke2-vm`)
   - From `onprem-vm`, ssh to 10.1.10.4 (`spoke1-vm`). Attempt to ping 10.2.10.4 (`spoke2-vm`)
   - From `spoke1-vm`, `traceroute` to 172.16.10.4 (`onprem-vm`) and observe the path.
   - From one of the cloud VMs, attempt to `curl google.com`.

1. Add the following rules to the `Internal` firewall ruleset above:
   | Name | Source type | Source | Protocol | Destination Ports | Destination Type | Destination |
   | -------------- | ----------- | ----------------------------------- | -------- | ----------------- | ---------------- | ----------------------------------- |
   | Spoke1-to-Spoke2 | IP Address | 10.1.0.0/16 | Any | \* | IP Address | 10.2.0.0/16 |
   | Spoke2-to-Spoke1 | IP Address | 10.2.0.0/16 | Any | \* | IP Address | 10.1.0.0/16 |
1. Once the ruleset has saved, attempt to ping from `spoke1-vm`, attempt to ping 10.2.10.4 (`spoke2-vm`). **Does it work? Why?**
1. Navigate to the Application rules blade of the firewall policy. Add the following rule:

   | Name      | Source type | Source                                | Protocol   | TLS inspection | Destination Type | Destination |
   | --------- | ----------- | ------------------------------------- | ---------- | -------------- | ---------------- | ----------- |
   | Allow-Web | IP Address  | 10.0.0.0/16, 10.1.0.0/16, 10.2.0.0/16 | http,https | No             | FQDN             | \*          |

1. Deploy a fifth VM with the following configuration:
   1. Name: `spoke1-vm2`
   1. Image: Ubuntu Server 20.04 LTS - Gen 2
   1. Size: `Standard_B2s`
   1. Authentication type: Password
      - Username: `azureuser`
      - Password: `P@$$w0rd12300`
   1. Public inbound ports: None
   1. VNet/subnet: `spoke1/default`
   1. Public IP: None
   1. NIC security group: `None`
   1. Public inbound ports: None
   1. Set up auto shutdown
   1. Input the following cloud-init config:
1. While it is deploying, edit the `spoke1` routing table and add the following rules:

   | Name   | Address prefix | Next hop type     | Next hop IP address |
   | ------ | -------------- | ----------------- | ------------------- |
   | spoke1 | 10.1.0.0/16    | Virtual appliance | 10.0.1.4            |

1. Wait a minute or two, and then go examine the effective routes on the `spoke1-vm` NIC (or the new `spoke1-vm2` NIC).
1. Once the new VM has deployed, `traceroute` from `spoke1-vm` to `spoke1-vm2` and examine the path.
1. Navigate to the Firewall resource, and click on Logs in the Monitoring section of the left-hand blade (**Note**: this is a shortcut to the Log Analytics resource created above, that also restricts the scope of the found logs to just those from the firewall resource, _and_ provides some example queries relevant to the specific resource type (in the case, Azure Firewall). The same set of logs can be viewed by going directly to the Log Analytics workspace, but you will need to ensure that your query (KQL) contains a `where` clause that restricts logs to just those of the firewall resource to achieve the same results.)
1. Load the example query called "Network rule log data" and run it. Find the log data the corresponds to the `traceroute` done between `spoke1-vm` and `spoke1-vm2` above.
1. Using a new browser tab (or a second person), edit the `spoke1` routing table again and remove the `spoke1` rule created above.
1. Wait a minute or two, and then go examine the effective routes on the `spoke1-vm` NIC (or the new `spoke1-vm2` NIC).
1. `traceroute` again from `spoke1-vm` to `spoke1-vm2` and examine the path.
1. Re-run the "Network rule log data" query and examine the data (**Note**: data ingestion into Log Analytics can take a few minutes. In this case, you should never see data about the `traceroute` done above, but to be sure, wait a minute or two and re-run the query.)
1. Open a web browser and navigate to the public IP address of `spoke1-vm`. Your connection should time out. **Why?**
1. Browse to the firewall policy, and add a DNAT rule:

   - Name: `Dnat`
   - Priority: 100
   - Rule collection group: `DefaultDnatRuleCollectionGroup`
   - Rules:

   | Rule name | Source | Port | Protocol | Destination               | Translated Address or FQDN | Translated Port |
   | --------- | ------ | ---- | -------- | ------------------------- | -------------------------- | --------------- |
   | Spoke1-vm | \*     | 80   | TCP      | \<public IP of firewall\> | \<public IP of spoke1-vm\> | 80              |
   | Spoke2-vm | \*     | 81   | TCP      | \<public IP of firewall\> | \<public IP of spoke2-vm\> | 80              |
   | hub-vm    | \*     | 82   | TCP      | \<public IP of firewall\> | \<public IP of hub-vm\>    | 80              |

1. Browse to the public IP of the firewall. You should see the `spoke1-vm` web page.
1. Browse to the public IP of the firewall on port 81. You should see the `spoke2-vm` web page.
1. Browse to the public IP of the firewall on port 82. You should see the `hub-vm` web page.
1. SSH to the `spoke1-vm` and issue the command: `cat /var/log/apache2/access.log`. Examine the logs. Pay attention to the first "column" and explain what you see.

---

## Discussion:

1. Can you explain why we put the firewall's PIP/32 routing to the Internet in the `hub`, `spoke1`, and `spoke2` route tables?
1. How can we protect multiple sites with multiple servers serving them behind a single firewall?

---

## Success Criteria

1. The spoke VMs can reach each other.
1. The Azure Firewall should inspect traffic from any VM in the hub or the spoke going to the public Internet or to onprem.
1. The Azure Firewall should inspect traffic from any on-premise client going to the hub or any spokes.
1. The solution should be independent of network administrators adding, changing or removing prefixes in the on-premises network in the future.
1. A web server should be installed on each VM, and it should be reachable from the public Internet.
1. The participants should be able to show Azure Firewall logs, to demonstrate that traffic traverses the firewall.
1. The participants should be able to identify the client source IP as seen by each web server and explain it.

## Related documentation

- [What is Azure Firewall](https://docs.microsoft.com/azure/firewall/overview)
