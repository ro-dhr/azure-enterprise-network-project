# Azure Enterprise Network Lab: Multi-Region Deployment (Manual + Terraform) (In Progress)

This project simulates a small enterprise network with two sites in different Azure regions. The goal was to build the same type of network foundation two different ways and compare the experience, then work through a real deployment failure to practice troubleshooting. This project is still in progress. More phases will be added as they're completed.

## Table of Contents

| Section | Description |
|---|---|
| [Project Overview](#project-overview) | What this project is and why it's set up this way |
| [Architecture](#architecture) | Side by side comparison of both sites |
| [Phase 1: Site 1, US West (Manual Deployment)](#phase-1-site-1-us-west-manual-deployment) | Building the network foundation and VM through the Azure Portal |
| [Phase 2: Site 2, UK West (Terraform Deployment)](#phase-2-site-2-uk-west-terraform-deployment) | Building the same network foundation with Terraform and PowerShell |
| [Scenario #1: EU Admin Can't Reach the US Site](#scenario-1-eu-admin-cant-reach-the-us-site) | A simulated connectivity issue between the two sites |
| [Scenario #2: US VM Suddenly Can't Reach the EU VM](#scenario-2-us-vm-suddenly-cant-reach-the-eu-vm) | A route table rule silently blocking traffic between sites |
| [Scenario #3: New EU App Subnet](#scenario-3-new-eu-app-subnet-requested-by-the-network-manager) | Standing up a new isolated app subnet on request |
| [Scenario #4: Setting up Azure Firewall](#scenario-4-locking-down-outbound-traffic-with-azure-firewall) | Adding Azure Firewall for traffic control and visibility |
| [Scenario #5: External User Needs Access to the App VM](#scenario-5-external-user-needs-access-to-the-app-vm) | External user needs access to EU-APP site |
| [Troubleshooting](#troubleshooting) | Issues run into during the project and how they were resolved |

## Project Overview

The idea behind this lab was to first stand up two separate network sites in Azure, each in its own region, and connect the deployment process to real-world practices. Site 1 was deployed through the Azure Portal to understand what each resource does and how they connect. Site 2 was deployed with Terraform to practice infrastructure as code and see how the same resources look when they're automated. After both sites are created, I'll go back in and intentionally break or add stuff: misconfigured NSG rules, bad routes, a new VM, etc. Then I'll work through diagnosing, fixing/creating, verifying, and documenting each issue. I'll also document any genuine issues that come up when doing this and show what I learned and how I fixed it

## Architecture

| | Site 1 (Manual) | Site 2 (Terraform) |
|---|---|---|
| Region | West US | UK West |
| Resource Group | rg-network-lab | rg-eu-network-lab |
| VNet | vnet-us-hq (10.20.0.0/16) | vnet-eu-network-lab (10.30.0.0/16) |
| Subnet | subnet-us-hq (10.20.1.0/24) | subnet-eu-vm (10.30.1.0/24) |
| NSG | nsg-us-hq | nsg-eu-network-lab |
| Route Table | rtable-us-hq | rt-eu-network-lab |
| VM | vm-us-hq (Ubuntu 24.04, Standard_D2als_v7) | vm-eu-network-lab (Ubuntu 24.04, Standard_D2als_v6) |
| Deployment Method | Azure Portal | Terraform + PowerShell |

Both sites use a private subnet with no default outbound internet access, and both restrict inbound SSH (port 22) to a single admin IP through the NSG.

## Phase 1: Site 1, US West (Manual Deployment)

This phase covers building the network foundation and VM for the US site entirely through the Azure Portal.

### Create the Virtual Network

Reviewed the VNet settings before creating it. Address space set to 10.20.0.0/16, with a subnet carved out at 10.20.1.0/24.

![VNet review](screenshots/site1-01-vnet-review.png)

VNet created successfully in West US under the rg-network-lab resource group.

![VNet created](screenshots/site1-02-vnet-created.png)

### Create the Network Security Group

Created nsg-us-hq to control traffic in and out of the subnet.

![NSG created](screenshots/site1-03-nsg-created.png)

Added an inbound rule allowing SSH (port 22) from a single admin IP only, keeping the NSG locked down instead of leaving management access open to the internet.

![NSG SSH rule](screenshots/site1-04-nsg-ssh-rule.png)

### Create the Route Table

Reviewed the route table settings before creating it.

![Route table review](screenshots/site1-05-route-table-review.png)

Route table rtable-us-hq created in West US.

![Route table created](screenshots/site1-06-route-table-created.png)

### Associate the NSG and Route Table with the Subnet

Attached both nsg-us-hq and rtable-us-hq to subnet-us-hq so traffic in the subnet would actually be governed by them.

![Associating subnet](screenshots/site1-07-associating-subnet.png)

Confirmed the association went through. The subnet now shows both the security group and route table attached.

![Association complete](screenshots/site1-08-association-complete.png)

### Deploy the Virtual Machine

Configured the VM: Ubuntu Server 24.04 LTS, Standard_D2als_v7, SSH key authentication with a dedicated key pair, placed inside vnet-us-hq / subnet-us-hq with nsg-us-hq attached.

![VM configuration](screenshots/site1-09-vm-create-config.png)

VM deployed and running.

![VM deployed](screenshots/site1-10-vm-deployed.png)

### Verify the Deployment

Confirmed the hostname matched what was configured.

![Hostname check](screenshots/site1-11-hostname.png)

Connected to the VM over SSH using the key pair generated at creation.

![SSH login](screenshots/site1-12-ssh-login.png)

Checked the network interface with `ip addr` to confirm the VM picked up the expected private IP in the 10.20.1.0/24 range.

![ip addr output](screenshots/site1-13-ip-addr.png)

Reviewed the routing table on the VM itself with `ip route` to see how it was reaching the default gateway.

![ip route output](screenshots/site1-14-ip-route.png)

Ran a ping test to 8.8.8.8 to confirm outbound connectivity.

![Ping test](screenshots/site1-15-ping-test.png)

### Basic Server Prep

Ran an update to make sure the package index was current.

![apt update](screenshots/site1-16-apt-update.png)

Installed a few standard networking utilities (traceroute, tcpdump, dnsutils, net-tools) to have on hand for future troubleshooting.

![Installing net-tools](screenshots/site1-17-net-tools-install.png)

## Phase 2: Site 2, UK West (Terraform Deployment)

This phase covers building the same type of network foundation for the UK site, but this time using Terraform instead of clicking through the portal. Changes were applied through PowerShell.

### Confirm Terraform and Azure CLI Setup

Checked the installed Terraform version and confirmed the Azure CLI was authenticated against the correct subscription before writing any config.

![Terraform and az account check](screenshots/site2-01-terraform-az-account.png)

### Set Up the Project Directory

Created a working directory for the project and a subfolder to hold the Terraform configuration.

![Creating directories](screenshots/site2-02-creating-directories.png)

### Initialize Terraform

Created main.tf and ran `terraform init`, which pulled down the AzureRM provider.

![Terraform init](screenshots/site2-03-terraform-init.png)

### Define Variables

Set up variables.tf with the region and resource group name for the automated site, defaulting to UK West and rg-eu-network-lab.

![Variables file](screenshots/site2-04-variables-file.png)

Re-ran `terraform init` from inside the project's terraform directory to confirm everything was wired up correctly before moving on to planning.

![Terraform init complete](screenshots/site2-05-terraform-init-complete.png)

### Apply the Configuration

Ran `terraform plan` and `terraform apply`. Terraform provisioned all 11 resources in one pass: the resource group, NSG, NSG rule, route table, VNet, subnet, subnet associations, public IP, network interface, and the VM itself.

![Terraform apply](screenshots/site2-06-terraform-apply.png)

### Infrastructure as Code (main.tf)

Here's the full main.tf that made this happen. It's everything from the [Architecture](#architecture) table above written out as code: the resource group, VNet, subnet, NSG and its rule, route table, public IP, network interface, and the VM.

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
}
provider "azurerm" {
  features {}
}
resource "azurerm_resource_group" "eu_site" {
  name     = var.resource_group_name
  location = var.location
}
resource "azurerm_virtual_network" "eu_vnet" {
  name                = "vnet-eu-network-lab"
  location            = azurerm_resource_group.eu_site.location
  resource_group_name = azurerm_resource_group.eu_site.name
  address_space       = ["10.30.0.0/16"]
}
resource "azurerm_subnet" "eu_subnet" {
  name                 = "subnet-eu-vm"
  resource_group_name  = azurerm_resource_group.eu_site.name
  virtual_network_name = azurerm_virtual_network.eu_vnet.name
  address_prefixes     = ["10.30.1.0/24"]
}
resource "azurerm_network_security_group" "eu_nsg" {
  name                = "nsg-eu-network-lab"
  location            = azurerm_resource_group.eu_site.location
  resource_group_name = azurerm_resource_group.eu_site.name
}
resource "azurerm_network_security_rule" "allow_ssh" {
  name                        = "Allow-SSH-From-Admin-IP"
  priority                    = 100
  direction                   = "Inbound"
  access                      = "Allow"
  protocol                    = "Tcp"
  source_port_range           = "*"
  destination_port_range      = "22"
  source_address_prefix       = "<your-admin-ip>/32"
  destination_address_prefix  = "*"
  resource_group_name         = azurerm_resource_group.eu_site.name
  network_security_group_name = azurerm_network_security_group.eu_nsg.name
}
resource "azurerm_subnet_network_security_group_association" "eu_subnet_nsg" {
  subnet_id                 = azurerm_subnet.eu_subnet.id
  network_security_group_id = azurerm_network_security_group.eu_nsg.id
}
resource "azurerm_route_table" "eu_route_table" {
  name                = "rt-eu-network-lab"
  location            = azurerm_resource_group.eu_site.location
  resource_group_name = azurerm_resource_group.eu_site.name
}
resource "azurerm_subnet_route_table_association" "eu_subnet_route_table" {
  subnet_id      = azurerm_subnet.eu_subnet.id
  route_table_id = azurerm_route_table.eu_route_table.id
}
resource "azurerm_public_ip" "eu_vm_public_ip" {
  name                = "pip-eu-vm"
  location            = azurerm_resource_group.eu_site.location
  resource_group_name = azurerm_resource_group.eu_site.name
  allocation_method   = "Static"
  sku                 = "Standard"
}
resource "azurerm_network_interface" "eu_vm_nic" {
  name                = "nic-eu-vm"
  location            = azurerm_resource_group.eu_site.location
  resource_group_name = azurerm_resource_group.eu_site.name
  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.eu_subnet.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.eu_vm_public_ip.id
  }
}
resource "azurerm_linux_virtual_machine" "eu_vm" {
  name                = "vm-eu-network-lab"
  resource_group_name = azurerm_resource_group.eu_site.name
  location            = azurerm_resource_group.eu_site.location
  size                = "Standard_D2als_v6"
  admin_username      = "azureadmin"
  network_interface_ids = [
    azurerm_network_interface.eu_vm_nic.id
  ]
  admin_ssh_key {
    username   = "azureadmin"
    public_key = file("~/.ssh/id_ed25519.pub")
  }
  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }
  source_image_reference {
    publisher = "Canonical"
    offer     = "ubuntu-24_04-lts"
    sku       = "server"
    version   = "latest"
  }
}
```

Walking through it block by block:

- **`terraform` / `provider "azurerm"`:** just getting Terraform ready to talk to Azure. The `terraform` block locks in AzureRM version 4.x so the config doesn't quietly break on some future provider update, and `provider "azurerm"` is what actually connects everything to the subscription.
- **`azurerm_resource_group.eu_site`:** lays down rg-eu-network-lab first, since everything else needs somewhere to live.
- **`azurerm_virtual_network.eu_vnet`:** creates the VNet itself with the 10.30.0.0/16 range.
- **`azurerm_subnet.eu_subnet`:** carves out the 10.30.1.0/24 subnet inside that VNet.
- **`azurerm_network_security_group.eu_nsg`:** creates an empty NSG, just the container for whatever rules go inside it.
- **`azurerm_network_security_rule.allow_ssh`:** the actual rule, only letting SSH in from one admin IP.
- **`azurerm_subnet_network_security_group_association.eu_subnet_nsg`:** the part that's easy to forget. An NSG doesn't do anything on its own, it has to be attached to the subnet, and this is what does that.
- **`azurerm_route_table.eu_route_table`:** creates the route table.
- **`azurerm_subnet_route_table_association.eu_subnet_route_table`:** same idea as the NSG association, attaches the route table to the subnet so it's actually in play.
- **`azurerm_public_ip.eu_vm_public_ip`:** grabs a static public IP for the VM to use.
- **`azurerm_network_interface.eu_vm_nic`:** builds the NIC and hooks that public IP up to it.
- **`azurerm_linux_virtual_machine.eu_vm`:** the VM itself, an Ubuntu 24.04 box on a Standard_D2als_v6, using SSH key auth instead of a password, and finally connected to the NIC from the step above.

### Verify the Resources in Azure

Confirmed the VNet was created in UK West with the expected address space (10.30.0.0/16).

![VNet created](screenshots/site2-07-vnet-created.png)

Confirmed the subnet was created and correctly associated with both the NSG and route table.

![Subnet created](screenshots/site2-08-subnet-created.png)

Reviewed the NSG. Same approach as Site 1: inbound SSH locked to a single admin IP, everything else default deny.

![NSG created](screenshots/site2-09-nsg-created.png)

Confirmed the route table was created and associated with the subnet.

![Route table created](screenshots/site2-10-route-table-created.png)

Confirmed the VM was up and running.

![VM created](screenshots/site2-11-vm-created.png)

Pulled up the full resource group to see everything Terraform had created in one view: the NIC, NSG, public IP, route table, VM, OS disk, and VNet.

![Resource group overview](screenshots/site2-12-resource-group-overview.png)

### Verify Connectivity

Connected to the VM over SSH using an ED25519 key pair.

![SSH login](screenshots/site2-13-ssh-login.png)

Checked hostname, network interface, routing table, and ran a ping test, same verification steps as Site 1, to confirm the automated deployment produced a working VM with the expected configuration.

![SSH verification](screenshots/site2-14-ssh-more-info.png)

## Scenario #1: EU Admin Can't Reach the US Site

This one was set up on purpose, a simulated scenario to practice diagnosing a connectivity issue across the two sites rather than a real deployment problem. An EU admin is trying to access the US branch (vm-us-hq) from the EU VM (vm-eu-network-lab) and can't get through.

A ping from the EU VM to the US VM's private IP (10.20.1.4) timed out completely, and an SSH attempt just hung with no response.

![Ping fails](screenshots/scenario1-01-ping-fails.png)
![SSH fails](screenshots/scenario1-02-ssh-fails.png)

The two VNets, vnet-eu-network-lab and vnet-us-hq, had never been peered, so there was no path between them at all.

![Peering EU to US](screenshots/scenario1-03-peering-eu-to-us.png)
![Peering US to EU](screenshots/scenario1-04-peering-us-to-eu.png)
![Peering created](screenshots/scenario1-05-peering-created.png)

With peering set up in both directions, ping started working right away.

![Ping succeeds](screenshots/scenario1-06-ping-succeeds.png)

SSH, though, still wasn't going through.

![SSH still fails](screenshots/scenario1-07-ssh-still-fails.png)

Checked the inbound rules on nsg-us-hq and found a Deny-SSH-FromEU rule sitting at priority 200, explicitly blocking port 22 from the entire EU subnet (10.30.1.0/24).

![NSG deny rule found](screenshots/scenario1-08-nsg-deny-rule-found.png)

Added a narrower allow rule above it, priority 190, allowing SSH from just the one EU admin VM's IP (10.30.1.4).

![Allow rule config](screenshots/scenario1-09-allow-rule-config.png)
![Allow rule created](screenshots/scenario1-10-allow-rule-created.png)

SSH from the EU VM to the US VM worked.

![SSH succeeds](screenshots/scenario1-11-ssh-succeeds.png)

## Scenario #2: US VM Suddenly Can't Reach the EU VM

Connectivity between the two sites was working fine, and then it just wasn't. The US VM (vm-us-hq) can't reach the EU VM (vm-eu-network-lab) anymore, and I need to figure out where the traffic is actually getting dropped.

### Confirm the Issue

A ping from the US VM to the EU VM's private IP (10.30.1.4) times out completely.

![Ping fails](screenshots/scenario2-01-ping-fails.png)

### Diagnose and Fix

First instinct is an NSG rule, so I check the outbound rules on nsg-us-hq. Everything there looks normal: default allow rules only, nothing blocking outbound traffic to the EU subnet.

![US NSG outbound rules look fine](screenshots/scenario2-02-us-nsg-outbound.png)

Just to be safe, I check the inbound rules on nsg-eu-network-lab too. Also fine; nothing there would be dropping this traffic either.

![EU NSG inbound rules look fine](screenshots/scenario2-03-eu-nsg-inbound.png)

With both NSGs ruled out, I go into Network Watcher and use the Next Hop tool to see how traffic from the US VM is actually being routed toward the EU VM's IP.

![Network Watcher next hop tool](screenshots/scenario2-04-next-hop-tool.png)

The result comes back as Next Hop Type: None. That means this traffic has no route at all, it's being dropped before it can even leave the subnet.

![Next hop result comes back None](screenshots/scenario2-05-next-hop-none.png)

That points straight at the route table, so I check rtable-us-hq and find a route called Block-Eu-Test, targeting 10.30.1.0/24 with a next hop type of None. That's effectively a black hole route for anything headed to the EU subnet, and it has no business being there.

![Route table shows the blocking rule](screenshots/scenario2-06-blocking-route.png)

I delete it.

![Deleting the route](screenshots/scenario2-07-deleting-route.png)

And it's gone.

![Route deleted](screenshots/scenario2-08-route-deleted.png)

### Verify

Running the Next Hop tool again, the result now comes back as VirtualNetworkPeering instead of None, meaning traffic has a real path to the EU VM through the peering connection.

![Next hop now shows VirtualNetworkPeering](screenshots/scenario2-09-next-hop-peering.png)

Ping confirms it, connectivity between the two VMs is fully restored.

![Ping succeeds](screenshots/scenario2-10-ping-succeeds.png)

## Scenario #3: New EU App Subnet, Requested by the Network Manager

The network manager requests a second VM for the EU site to act as an internal application server, in its own subnet. They give me the parameters to use: subnet-eu-app on 10.30.2.0/24, its own NSG with SSH locked to the admin IP, and once it's up, she wants that new app subnet fully isolated from the US site as a segmentation requirement. Everything here goes in through Terraform.

### Building the App Subnet and VM

Starting with the subnet itself, subnet-eu-app, carved out of the same vnet-eu-network-lab.

![Adding the app subnet](screenshots/scenario3-01-app-subnet.png)

Adding the route table association so the new subnet inherits the same routing behaviour as the rest of the site.

![Route table association](screenshots/scenario3-02-route-table-association.png)

Then a dedicated NSG for the app subnet, nsg-eu-app, with its own SSH rule scoped to the admin IP, plus the association tying it to the subnet.

![New NSG, rule, and association](screenshots/scenario3-03-nsg-rule-association.png)

And the VM itself needs a public IP and a NIC.

![Public IP and NIC](screenshots/scenario3-04-nic-public-ip.png)

Running a plan first to see exactly what Terraform is about to create. Eight new resources in total as expected.

![Terraform plan](screenshots/scenario3-05-terraform-plan.png)

Applying it.

![Terraform apply](screenshots/scenario3-06-terraform-apply.png)

Confirming the subnet is in place with the right NSG and route table attached.

![Subnet verified](screenshots/scenario3-07-subnet-verify.png)

And the NSG has the rules I'd expect.

![NSG rules](screenshots/scenario3-08-nsg-rules.png)

The VM itself is up and running in UK West, sitting on 10.30.2.4.

![VM created](screenshots/scenario3-09-vm-created.png)

SSH into it to confirm it's reachable.

![SSH into vm-eu-app](screenshots/scenario3-10-ssh-login.png)

Hostname, network interface, and routing table all look correct.

![ip addr, hostname, routing table](screenshots/scenario3-11-ip-info.png)

Before touching anything else, I confirm vm-eu-app can reach both the original EU VM and the US VM, full connectivity across the whole environment as a baseline.

![Connectivity to both EU and US VMs](screenshots/scenario3-12-connectivity-baseline.png)

### Locking It Down

Now for the part the network manager actually asked for: this new app subnet needs to be isolated from the US site entirely. I add two rules to nsg-eu-app, one denying all inbound traffic from 10.20.1.0/24, one denying all outbound traffic to it.

![Deny rules for US traffic](screenshots/scenario3-13-deny-rules.png)

Plan first to double-check exactly what's about to change.

![Terraform plan for the deny rules](screenshots/scenario3-14-terraform-plan-deny.png)

Then apply it.

![Rules pushed](screenshots/scenario3-15-terraform-apply-deny.png)

### Verify

From vm-eu-app, a ping to the US VM now fails completely.

![EU app VM can't reach US VM](screenshots/scenario3-16-eu-app-to-us-fails.png)

And it holds in the other direction too, the US VM can no longer reach vm-eu-app either.

![US VM can't reach EU app VM](screenshots/scenario3-17-us-to-eu-app-fails.png)

Segmentation is working exactly as requested, vm-eu-app is fully isolated from the US site while still able to talk to the rest of the EU network.

## Scenario #4: Locking Down Outbound Traffic with Azure Firewall

The network manager wants more visibility and control over what leaves the EU site, not just NSG allow/deny rules but actual traffic inspection.

### Building the Firewall

First, a dedicated subnet for it. Azure Firewall needs its own subnet, named exactly AzureFirewallSubnet, carved out at 10.30.0.0/24.

![Creating the firewall subnet](screenshots/scenario4-01-firewall-subnet.png)

It also needs a static public IP to use as its frontend.

![Public IP for the firewall](screenshots/scenario4-02-firewall-public-ip.png)

Reviewing the configuration before deploying.

![Firewall review](screenshots/scenario4-03-firewall-review.png)

Deployment completed.

![Firewall created](screenshots/scenario4-04-firewall-created.png)

### Writing the Rules

The goal here is straightforward: block outbound HTTP, allow outbound HTTPS. First, a network rule denying port 80 from both EU subnets.

![Deny HTTP rule](screenshots/scenario4-05-deny-http-rule.png)

And an allow rule for port 443 from the app subnet.

![Allow HTTPS rule](screenshots/scenario4-06-allowed-https-rule.png)

### Routing Traffic Through the Firewall

Rules alone don't do anything unless traffic actually passes through the firewall, so I update rt-eu-network-lab with a default route (0.0.0.0/0) pointing at the firewall's private IP.

![Route pointing to the firewall](screenshots/scenario4-07-route-to-firewall.png)

### Verify

From the original EU VM, testing both ports against a public IP. HTTPS connects successfully, HTTP times out exactly as expected.

![EU VM firewall test](screenshots/scenario4-08-eu-vm-firewall-test.png)

Same test from vm-eu-app, same result.

![EU app VM firewall test](screenshots/scenario4-09-eu-app-firewall-test.png)

### Adding Visibility

Since the whole point of this was better visibility, I turn on diagnostic logging for the firewall, sending network and application rule logs to a Log Analytics workspace so denied and allowed traffic is actually queryable.

![Setting up monitoring](screenshots/scenario4-10-diagnostic-settings.png)

To generate some traffic to look at, I send a few HTTP requests from the EU VM that I know will get blocked.

![Sending HTTP requests](screenshots/scenario4-11-curl-requests.png)

Then query the logs for denied HTTP attempts, grouped by source IP. Both EU VMs show up, exactly the ones that just tried.

![Denied HTTP attempts in the logs](screenshots/scenario4-12-log-query-results.png)

## Scenario #5: External User Needs Access to the App VM

A new request: someone outside the organization, a contractor, needs SSH access to vm-eu-app specifically. Handing out the admin credentials or opening the app subnet up broadly isn't an option, so this needs to go through the firewall with a dedicated, scoped-down account on the VM itself.

### Setting Up an External Test Client

To simulate this properly, I spin up a separate VM, externalVM, in its own resource group and region (Spain Central), completely outside the lab's environment, just to act as an outside user hitting the firewall from the internet.

![External test VM](screenshots/scenario5-01-external-vm.png)

It's sitting in its own isolated VNet with no connection to anything in the lab.

![External VM's VNet](screenshots/scenario5-02-external-vnet.png)

With a DNAT rule already set up on the firewall forwarding port 44001 to vm-eu-app's SSH port, I test from externalVM whether that port is even reachable from outside.

![Testing the port from outside](screenshots/scenario5-03-port-test.png)

### Preparing the App VM

On vm-eu-app, I create a separate account, externaladmin, specifically for this kind of outside access instead of handing over anything tied to the admin account.

![Creating the external account](screenshots/scenario5-04-external-account.png)

### Verify

From externalVM, SSHing to the firewall's public IP on port 44001, which gets DNAT'd through to port 22 on vm-eu-app. It connects, and whoami confirms I'm logged in as externaladmin, not the admin account.

![SSH from outside Azure succeeds](screenshots/scenario5-05-ssh-success.png)

## Troubleshooting

This section covers issues run into during the project and how they were resolved. More will be added here as they come up.

### Issue: VM SKU Not Available in Region

**What happened:** While applying the Terraform configuration for Site 2, the VM deployment failed with a 409 Conflict error. The error indicated that Standard_B1s was not available in UK West due to capacity restrictions on that SKU in that region.

![SKU error](screenshots/troubleshooting-01-vm-sku-error.png)

**Resolution:** Changed the VM size in the Terraform configuration from Standard_B1s to Standard_D2als_v6, which was available in UK West. Re-ran `terraform apply` and the deployment completed successfully.

**Takeaway:** Not every VM size is available in every region, and capacity can vary even within a region depending on current demand. When a SKU fails with an availability error, checking Azure's SKU availability for that specific region (or just trying a comparable size) is usually faster than trying to force the original one through.


