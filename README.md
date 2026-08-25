# Azure Enterprise Network Lab: Multi-Region Deployment (Manual + Terraform) (In Progress)

This project simulates a small enterprise network with two sites in different Azure regions. The goal was to build the same type of network foundation two different ways and compare the experience, then work through a real deployment failure to practice troubleshooting. This project is still in progress. More phases will be added as they're completed.

## Table of Contents

| Section | Description |
|---|---|
| [Project Overview](#project-overview) | Bit more explanation about the project |
| [Architecture](#architecture) | Side by side comparison of both sites |
| [Phase 1: Site 1, US West (Manual Deployment)](#phase-1-site-1-us-west-manual-deployment) | Manual Configuration |
| [Phase 2: Site 2, UK West (Terraform Deployment)](#phase-2-site-2-uk-west-terraform-deployment) | Automated via Terraform and PowerShell |
| [Scenario #1: EU Admin Can't Reach the US Site](#scenario-1-eu-admin-cant-reach-the-us-site) | A simulated connectivity issue between the two sites |
| [Troubleshooting](#troubleshooting) | Issues run into during the project and how they were resolved |

## Project Overview

The idea behind this lab was to first stand up two separate network sites in Azure, each in its own region, and connect the deployment process to real-world practices. Site 1 was deployed through the Azure Portal to understand what each resource does and how they connect. Site 2 was deployed with Terraform to practice infrastructure as code and see how the same resources look when they're automated. After both sites are created, I'll go back in and intentionally break or add stuff: misconfigured NSG rules, bad routes, a new VM, etc. Then I'll work through diagnosing, fixing/creating, verifying, and documenting each issue. I'll also document any genuine issues that come up when doing this and show what I learned and how I fixed it.

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

Created the VNET successfully in West US under the rg-network-lab resource group.

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

Attached both nsg-us-hq and rtable-us-hq to subnet-us-hq so traffic in the subnet is governed by them.

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
- **`azurerm_subnet_network_security_group_association.eu_subnet_nsg`:** the part that's easy to forget. An NSG doesn't do anything on its own; it has to be attached to the subnet, and this is what does that.
- **`azurerm_route_table.eu_route_table`:** creates the route table.
- **`azurerm_subnet_route_table_association.eu_subnet_route_table`:** same idea as the NSG association; attaches the route table to the subnet.
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
 
An EU admin is trying to access the US branch (vm-us-hq) via SSH from the EU VM (vm-eu-network-lab) and can't get through.

### Confirm The Issue
 
First, i'll try a ping from the EU VM to the US VM's private IP (10.20.1.4) to see if there's any connectivity
 
![Ping fails](screenshots/scenario1-01-ping-fails.png)

Completely timed out, and without connectivity, SSH won't work either.

![SSH fails](screenshots/scenario1-02-ssh-fails.png)

### Diagnose and Fix
 
Since these two sites are split across different VNets, I went to check if both were peered. Found out they were not, so it's time to create a peer. 
 
![Peering EU to US](screenshots/scenario1-03-peering-eu-to-us.png)

Since the two VNets had never been peered, there was no path between them at all.

![Peering US to EU](screenshots/scenario1-04-peering-us-to-eu.png)

And the peer is created.

![Peering created](screenshots/scenario1-05-peering-created.png)

### Verify
 
With peering set up in both directions, ping started working right away.
 
![Ping succeeds](screenshots/scenario1-06-ping-succeeds.png)
 
SSH, though, still wasn't going through. Since it isn't going through at all, it's most likely an NSG rule blocking it and not an SSH misconfig.
 
![SSH still fails](screenshots/scenario1-07-ssh-still-fails.png)

### Diagnose and Fix
 
Checked the inbound rules on nsg-us-hq and found a Deny-SSH-FromEU rule sitting at priority 200, blocking port 22 from the entire EU subnet (10.30.1.0/24).
 
![NSG deny rule found](screenshots/scenario1-08-nsg-deny-rule-found.png)
 
Added a narrower allow rule above it, priority 190, allowing SSH from just the one EU admin VM's IP (10.30.1.4) and not the entire branch.
 
![Allow rule config](screenshots/scenario1-09-allow-rule-config.png)
![Allow rule created](screenshots/scenario1-10-allow-rule-created.png)

### Verify
 
SSH from the EU VM to the US VM worked! Problem fixed!
 
![SSH succeeds](screenshots/scenario1-11-ssh-succeeds.png)

## Troubleshooting

This section covers issues run into during the project and how they were resolved. More will be added here as they come up.

### Issue: VM SKU Not Available in Region

**What happened:** While applying the Terraform configuration for Site 2, the VM deployment failed with a 409 Conflict error. The error indicated that Standard_B1s was not available in UK West due to capacity restrictions on that SKU in that region.

![SKU error](screenshots/troubleshooting-01-vm-sku-error.png)

**Resolution:** Changed the VM size in the Terraform configuration from Standard_B1s to Standard_D2als_v6, which was available in UK West. Re-ran `terraform apply` and the deployment completed successfully.


