# Deploying a Secure and Scalable Azure Virtual Network for Production

In this detailed project, I walk through building a real-time Azure Virtual Network (VNet) suitable for production environments. This project leverages various Azure services — VM Scale Sets, Application Gateway, NAT Gateway, and Azure Bastion — to build a highly resilient and secure network architecture.

## Project Overview

The goal of this project is to design and deploy an Azure VNet with the following features:

- High availability across two Availability Zones
- VM Scale Set for dynamic scalability, with scale-out and scale-in rules
- Application Gateway for traffic distribution
- Enhanced security with a private application subnet and NAT Gateway
- Secure access via Azure Bastion

## Architecture

The architecture implemented in this project:

- **Resource Group** — the main container for all resources
- **Virtual Network** — one Application Subnet and one `AzureBastionSubnet`
- **Network Security Groups** — one for the Application Subnet, one for the Application Gateway subnet
- **NAT Gateway** — deployed on the Application Subnet to enable outbound internet access for instances with no public IP
- **VM Scale Set** — ensures the right number of VM instances are running to handle the load, spread across two Availability Zones
- **Application Gateway** — distributes incoming traffic across VM Scale Set instances
- **Azure Bastion** — provides secure access to instances in the private Application Subnet

This is the architecture followed throughout this project.

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/28-landing-zone.png" alt="Landing Zone" width="800">
</p>

## Step-by-Step Implementation

### 1. Creating the Resource Group

- Navigate to Resource Groups in the Azure Portal and click **Create**.
- Set the resource group name and select the region to deploy into.
- Click **Review + Create**, then **Create**. Every resource built for the rest of this project — VNet, VM Scale Set, Application Gateway, NAT Gateway, Bastion — lives inside this single Resource Group.

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/1-rg-review.png" alt="Resource Group Review" width="800">
</p>

*Reviewing the resource group before creation.*

### 2. Setting Up the Virtual Network

- Navigate to the Virtual Networks service in the Azure Portal and click **Create**.
- Set the VNet name and specify the IPv4 address space (e.g., `10.0.0.0/16`).

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/2-vnet-name.png" alt="Virtual Network Name" width="800">
</p>

*Naming the VNet and setting its address space.*

- Add two subnets: an **Application Subnet** and an **AzureBastionSubnet**. Azure requires the Bastion subnet to be named exactly `AzureBastionSubnet`.

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/3-subnets-creation.png" alt="Subnets Creation" width="800">
</p>

*Adding the Application Subnet and the mandatory AzureBastionSubnet.*

- Click **Review + Create**, then **Create**. The VNet along with its subnets is successfully created.

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/4-vnet-created.png" alt="Virtual Network Created" width="800">
</p>

*The VNet and its two subnets, successfully provisioned.*

### 3. Configuring Network Security Groups

Two separate NSGs are needed here — one for the Application Gateway subnet, and one for the Application Subnet where the VM Scale Set instances live.

- Create an NSG for the Application Gateway subnet.

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/5-nsg-appgateway-creation.png" alt="NSG for Application Gateway Subnet" width="800">
</p>

*Creating the NSG for the Application Gateway subnet.*

- Configure its inbound rules. Application Gateway needs specific management ports (`65200–65535`) open in addition to HTTP (port 80), or its internal health signalling won't work.

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/6-app-gateway-inbound-rules.png" alt="Application Gateway NSG Inbound Rules" width="800">
</p>

*Inbound rules opening HTTP and the required management port range.*

- Create a second NSG for the Application Subnet, where the VM Scale Set instances will sit.

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/7-nsg-appsub-creation.png" alt="NSG for Application Subnet" width="800">
</p>

*Creating the NSG for the Application Subnet.*

- Configure its inbound rules to allow HTTP traffic only from the Application Gateway subnet, and deny everything else by default.

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/8-nsg-appsub-inbound-rules.png" alt="Application Subnet NSG Inbound Rules" width="800">
</p>

*Restricting inbound traffic to only the Application Gateway subnet.*

### 4. Reserving Public IP Addresses

Three resources in this architecture need a public IP: Application Gateway, Azure Bastion, and NAT Gateway. All three public IP addresses were provisioned up front so they were ready to attach as each resource was created.

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/10-total-3-public-ip-created.png" alt="Three Public IPs Created" width="800">
</p>

*All three public IPs reserved ahead of time.*

### 5. Creating Azure Bastion

Now it's time to deploy Azure Bastion, which lets us securely connect to the VM Scale Set instances inside the private Application Subnet, without ever opening SSH to the internet.

- Navigate to Bastions in the Azure Portal and click **Create**.
- Select the VNet created earlier — Azure automatically uses the `AzureBastionSubnet` configured earlier — and attach one of the public IPs reserved in the previous step.
- Click **Review + Create**, then **Create**. The Bastion resource is successfully deployed.

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/11-bastion-created.png" alt="Bastion Created" width="800">
</p>

*Azure Bastion successfully deployed.*

### 6. Provisioning VM Scale Set Instances

- Navigate to Virtual Machine Scale Sets in the Azure Portal and click **Create**.
- Specify the image (Ubuntu Server 24.04 LTS), size, and the basic configuration for the Scale Set.

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/12-vmss-basic-page-creation.png" alt="VM Scale Set Basics Page" width="800">
</p>

*Basic configuration for the VM Scale Set.*

- Configure networking to use the VNet and Application Subnet created earlier, with **Public IP per instance** set to **Disabled** — this is what keeps every instance private.
- Under Availability Zones, select **Zone 1** and **Zone 2**, so instances are spread across two independent physical locations. Set the initial instance count to 2.
- Click **Review + Create**, then **Create**.

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/13-vmss-created-successful.png" alt="VM Scale Set Created Successfully" width="800">
</p>

*Two instances now running — one in Zone 1, one in Zone 2.*

### 7. Connecting to the VM Instances via Bastion

With no public IPs on the instances, Azure Bastion is how we actually get in to configure them.

- Select an instance in the Portal, click **Connect**, and choose **Bastion**. No key files need to be copied anywhere — Bastion handles the tunnel entirely through the browser.

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/14-access-vm-through-bastion.png" alt="Accessing VM Through Bastion" width="800">
</p>

*Connecting to a private instance via Bastion, entirely through the browser.*

- Once connected, you're logged into the instance directly in the browser.

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/15-entered-vm.png" alt="Logged Into the VM" width="800">
</p>

*Logged into the instance with no public IP or SSH exposure.*

### 8. Creating the NAT Gateway

The VM Scale Set instances have no public IP, but they still need a safe path for outbound traffic — OS updates, package installs, and so on. That's what NAT Gateway provides.

- Navigate to NAT Gateways in the Azure Portal and click **Create**.
- Attach it to the Application Subnet, and use one of the public IPs reserved earlier.

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/16-nat-gate-creation.png" alt="NAT Gateway Creation" width="800">
</p>

*Configuring the NAT Gateway for outbound-only internet access.*

- Click **Review + Create**, then **Create**.

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/17-nat-review-create.png" alt="NAT Gateway Review + Create" width="800">
</p>

*Reviewing and creating the NAT Gateway.*

### 9. Deploying the Application

Back on the VM instance (connected via Bastion), install and start Nginx:

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

Edit the default web page so you can confirm which instance is serving a given request later:

```bash
sudo nano /var/www/html/index.html
```

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/19-index-file.png" alt="Editing the Index File" width="800">
</p>

*Editing the default Nginx page to identify the serving instance.*

```bash
sudo systemctl restart nginx
```

Repeat this for the second instance so both are ready to serve traffic.

### 10. Setting Up Application Gateway

Now for the final core piece — an Application Gateway that fronts both VM Scale Set instances.

- Navigate to Application Gateways in the Azure Portal and click **Create**.
- Set the name, select the **v2 SKU** (for zone redundancy and autoscaling of the gateway itself), choose the same Virtual Network, and attach the public IP reserved earlier.

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/20-appgate-create.png" alt="Application Gateway Creation" width="800">
</p>

*Creating the Application Gateway with the v2 SKU.*

- Configure the backend pool to point at the VM Scale Set, add an HTTP listener on port 80, and connect it via a routing rule.

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/21-routing-rule.png" alt="Routing Rule" width="800">
</p>

*Connecting the listener to the backend pool via a routing rule.*

- Review the full configuration and click **Create**.

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/22-app-gate%20review.png" alt="Application Gateway Review + Create" width="800">
</p>

*Final review before creating the Application Gateway.*

### 11. Accessing the Application

Once the Application Gateway finishes provisioning, copy its public IP address and paste it into the browser.

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/23-domain-search-screen.png" alt="Accessing the Application via Public IP" width="800">
</p>

*The application reachable through the Application Gateway's public IP.*

The application is now reachable — served through Application Gateway, load-balanced across two VM Scale Set instances in two separate Availability Zones, with neither instance ever exposing a public IP of its own.

### 12. Configuring Autoscale (Scale Out and Scale In)

To make this genuinely production-ready rather than just "working," the VM Scale Set needs to grow and shrink on its own based on real load, instead of running a fixed instance count forever.

- Navigate to the VM Scale Set, and under Settings click **Scaling**. Switch from **Manual scale** to **Custom autoscale**, and set the instance limits: **Minimum = 2**, **Maximum = 4**, **Default = 2**.

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/24-auto-scale-create-page.png" alt="Autoscale Configuration Page" width="800">
</p>

*Setting up custom autoscale with min/max/default instance counts.*

**Scale-Out Rule:** metric = Percentage CPU, operator = Greater than, threshold = 70%, action = Increase count by 1, cooldown = 5 minutes.

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/25-scaleout-rule.png" alt="Scale-Out Rule" width="800">
</p>

*The scale-out rule triggered when CPU exceeds 70%.*

**Scale-In Rule:** metric = Percentage CPU, operator = Less than, threshold = 30%, action = Decrease count by 1, cooldown = 5 minutes.

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/26-scalein-rule.png" alt="Scale-In Rule" width="800">
</p>

*The scale-in rule triggered when CPU drops below 30%.*

To confirm scale-out actually works, artificial CPU load was generated on one of the instances through Bastion:

```bash
sudo apt install stress -y
stress --cpu 2 --timeout 600
```

Within a few minutes, average CPU crossed the 70% threshold and the Scale Set automatically provisioned a third instance to absorb the load. Once the load stopped and CPU dropped below 30% past the cooldown window, the Scale Set brought the instance count back down to 2 on its own — no manual intervention either way.

> **Note:** Saving the autoscale configuration initially failed with a `MissingSubscriptionRegistration` error for `Microsoft.Insights`. This happens because autoscale depends on the Insights resource provider being registered on the subscription. Running the following from the CLI and retrying resolved it:
> ```bash
> az provider register --namespace Microsoft.Insights
> ```

## Conclusion

By following these steps, this sets up a production-grade Azure Virtual Network with high availability, scalability, and enhanced security. Application Gateway sits as the only public entry point, VM Scale Set instances stay entirely private across two Availability Zones, NAT Gateway handles outbound traffic, and Azure Bastion provides admin access without ever exposing SSH to the internet. On top of that, the scale-out and scale-in rules mean the Scale Set actually grows and shrinks with real demand instead of running a fixed instance count around the clock.

<p align="center">
  <img src="https://raw.githubusercontent.com/Rishi1306/secure-azure-vnet/main/work-images/27-whole-rg.png" alt="Final Resource Group" width="800">
</p>

*The complete resource group with every component in place.*

This setup ensures the application is resilient, cost-efficient, and secure — suitable for real-time deployment in production environments.

## Tech Stack

- **Azure Virtual Network (VNet)**
- **VM Scale Sets**
- **Application Gateway (v2 SKU)**
- **NAT Gateway**
- **Azure Bastion**
- **Network Security Groups**
- **Nginx**
