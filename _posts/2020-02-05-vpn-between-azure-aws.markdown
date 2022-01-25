---
title:  "How to VPN connect between Azure and AWS GovCloud Transit Gateway with Managed Services"
toc: true
date:   2020-02-05 00:00:00 +0000
permalink: /azure
tags: azure transit-gateway govcloud aws
---
I want to thank Jun Kudo for their
[post](https://hackernoon.com/how-to-connect-between-azure-and-aws-with-managed-services-4b03ec334e8a),
this all started learning from their post.

## TL; DR

If you don't want to read and just want to get it done, go here:
[janky-stuff/cloud/ipsec-between-azure-aws](https://github.com/missingcharacter/janky-stuff/tree/master/cloud/ipsec-between-azure-aws)

Follow instructions there and run:

```shell
AWS_PROFILE=your-profile ./create_ipsec.sh --azure-cidr <azure CIDR> \
  --azure-ip <public-ip1> --azure-ip <public-ip2> \
  --azure-location eastus --azure-resource-group <Resource Group> \
  --aws-cidr <aws CIDR> --aws-vpc-id <VPC ID>
```

If you want to know details continue reading

## Assumptions

### AWS Side

- You already have VPC(s)/subnets/route tables
- You already have a transit gateway setup with VPCs and/or VPNs

### Azure Side

- You already have a `Virtual Network`
- You already have a `Gateway Subnet`
- You already have a `Virtual Network Gateway` with a public IP address

*Note:* if you don't already have any resources listed in this section, I
recommend you follow
[Jun Kudo's instructions](https://hackernoon.com/how-to-connect-between-azure-and-aws-with-managed-services-4b03ec334e8a)

## The actual thing

### 1. Create customer gateway

Get the public IP addres for the Azure Virtual Network Gateway:

![Azure-Virtual-Network-GW](https://hackernoon.com/hn-images/0*hv7p3c8MsmIV8erW.jpg)

Create a static customer gateway

![01-AWS-Create-Customer-Gateway](/assets/images/azure/01-AWS-Create-Customer-Gateway.png)

### 2. Create a VPN Connection

|Parameter            |Value                                       |
|---------------------|--------------------------------------------|
|`Target Gateway Type`|`Transit Gateway` (existing transit gateway)|
|`Customer Gateway`   |The one you just created                    |
|`Routing Options`    |`Static`                                    |
|`Static IP Prefixes` |Azure's Virtual Network CIDR                |

![02-Create-VPN-Connection-part-1](/assets/images/azure/02-Create-VPN-Connection-part-1.png)

Under `Tunnel Options` select `Edit Tunnel 1 Options` and
`Edit Tunnel 2 Options` and use the options below for both tunnels:

| Parameter                     | Value    |
|-------------------------------|----------|
|`Phase 1 Encryption Algorithms`| `AES256` |
|`Phase 2 Encryption Algorithms`| `AES256` |
|`Phase 1 Integrity Algorithms` |`SHA2-256`|
|`Phase 2 Integrity Algorithms` |`SHA2-256`|
|`Phase 1 DH Group Numbers`     |  `14`    |
|`Phase 2 DH Group Numbers`     |  `14`    |
|`IkeVersion`                   | `ikev2`  |
|`Phase 1 Lifetime (seconds)`   |`<empty>` |
|`Phase 2 Lifetime (seconds)`   |`<empty>` |
|`Rekey Margin Time (seconds)`  |`<empty>` |
|`Rekey Fuzz (percentage)`      |`<empty>` |
|`Replay Window Size (packets)` |`<empty>` |
|`DPD Timeout (seconds)`        |`<empty>` |

![GovCloud-Tunnel-Options](/assets/images/azure/GovCloud-Tunnel-Options.png)

### 3. Obtain Pre-Shared Key and Public IPs

After the VPN is created, download its configuration.

![03-Download-VPN-Config](/assets/images/azure/03-Download-VPN-Config.png)

This file will have 2 Pre-Shared Keys, 1 per tunnel.

Public IPs will be in `Tunnel Details`

![05-AWS-VPN-Public-IP-Address](/assets/images/azure/05-AWS-VPN-Public-IP-Address.png)

### 4. Create Local Network Gateways

Create 1 Local Network Gateway per Tunnel, with the settings below:

<!-- markdownlint-disable MD013 -->

| Local Network Gateway | IP Address                      |                              Address Space |
| --------------------- | ------------------------------- |                              ------------- |
|       `aws-tunnel-01` | Outside IP Address for tunnel 1 | AWS CIDRs you want Azure to have access to |
|       `aws-tunnel-02` | Outside IP Address for tunnel 2 | AWS CIDRs you want Azure to have access to |

<!-- markdownlint-enable MD013 -->

![06-Azure-Create-Local-Network-Gateway](/assets/images/azure/06-Azure-Create-Local-Network-Gateway.png)

*Note*: `aws-tunnel-01` and `aws-tunnel-02` are suggested names, you may use
whatever nomenclature you prefer.

### 5. Create Connections

Go to your existing `Virtual Network Gateway` in Azure and add 1 connection per
Tunnel:

![07-Azure-Create-Connections](/assets/images/azure/07-Azure-Create-Connections.png)

Settings should be similar to the ones below:

<!-- markdownlint-disable MD013 -->

|   Connection Name | Connection Type      | Local Network Gateway | Pre-Shared Key  |
| ----------------- | -------------------- | --------------------- | --------------  |
|   `aws-tunnel-01` | Site-to-site (IPsec) |       `aws-tunnel-01` | key for tunnel1 |
|   `aws-tunnel-02` | Site-to-site (IPsec) |       `aws-tunnel-02` | key for tunnel2 |

<!-- markdownlint-enable MD013 -->

![08-Azure-Connection](/assets/images/azure/08-Azure-Connection.png)

*Note*: `aws-tunnel-01` and `aws-tunnel-02` are suggested names, you may use
whatever nomenclature you prefer.

### 6. Configure Azure Connections IPSec Policy

I couldn't find the following in the Azure Web Console, so I performed it using
the cli.

### 7. Repeat this process per connection

#### Connection has no IPSec policy

```shell
$ az network vpn-connection ipsec-policy list \
  --resource-group <Your Resource Group> --connection-name <Connection Name>
[]
```

#### Add IPSec policy to connection

```shell
az network vpn-connection ipsec-policy add \
  --resource-group <Your Resource Group> --connection-name <Connection Name> \
  --dh-group DHGroup14 --ike-encryption AES256 --ike-integrity SHA256 \
  --ipsec-encryption AES256 --ipsec-integrity SHA256 --pfs-group PFS2048 \
  --sa-lifetime 3600 --sa-max-size 1024
```

#### Verify Connection has IPSec Policy

```shell
$ az network vpn-connection ipsec-policy list \
  --resource-group <Your Resource Group> --connection-name <Connection Name>
[
  {
    "dhGroup": "DHGroup14",
    "ikeEncryption": "AES256",
    "ikeIntegrity": "SHA256",
    "ipsecEncryption": "AES256",
    "ipsecIntegrity": "SHA256",
    "pfsGroup": "PFS2048",
    "saDataSizeKilobytes": 1024,
    "saLifeTimeSeconds": 3600
  }
]
```

*Note*:
[Azure CLI page](https://docs.microsoft.com/en-us/cl/assets/images/azure/install-azure-cli?view=azure-cli-latest)

### 7. Tunnels should be up

#### AWS

![09-AWS-Tunnels-UP](/assets/images/azure/09-AWS-Tunnels-UP.png)

#### Azure

![10-Azure-Connections-UP](/assets/images/azure/10-Azure-Connections-UP.png)

### 8. Add Azure CIDR(s) to Transit Gateway Route Table

At this point Azure Resources within the `Virtual Network` associated to your
`Virtual Network Gateway` know about AWS CIDRs, thanks to
`Local Network Gateway`s.

Add Azure CIDRs to Transit Gateway Route Table:

![12-AWS-Transit-Gateway-Route-Table-AzureCIDR](/assets/images/azure/12-AWS-Transit-Gateway-Route-Table-AzureCIDR.png)

### 9. Add Azure CIDR(s) to AWS VPC Route tables

Now add Azure CIDRs to AWS VPC Route Tables and point them to the transit
gateway

### 10. You're done

Let me know if you know of a better way to do this!

## Notes

### BGP Issues

As of February 4, 2020. BGP peering seems to not be possible between AWS and
Azure due a conflict with `169.254.0.0/16`, Azure specifically states it does
not allow this range in their
[virtual network](https://docs.microsoft.com/en-u/assets/images/azure/virtual-network/virtual-networks-faq)

![00-Azure-reserved](/assets/images/azure/00-Azure-reserved.png)

On the AWS side, `169.254.0.0/16` is used as the `Inside IP CIDR` in a
`VPN Connection`.
At least someone provided feedback to Azure on how to improve their Networking,
see
[here](https://feedback.azure.com/forums/217313-networking/suggestions/38286799-ability-to-connection-azure-virtual-network-gatewa)

If someone figures out how to setup BGP between Azure and AWS, please let me
know.

### GovCloud VPN minimum requirements

You'll know them after you download the VPN's configuration:

```text
Category "VPN" connections in the GovCloud region have a minimum requirement of
AES128, SHA2, and DH Group 14.
```

![04-Category-VPN-min-reqs](/assets/images/azure/04-Category-VPN-min-reqs.png)

### Virtual Network Gateway Healthprobe

Azure offers a healthprobe for `Virtual Network Gateway`s that follows this
format `https://<Public IP Address of your Virtual Network Gateway>:8081`

![11-Azure-Virtual-Network-Gateway-Healthprobe](/assets/images/azure/11-Azure-Virtual-Network-Gateway-Healthprobe.png)

### Troubleshooting Tunnels

AWS gives no logs for their VPN Connections they only
[provide metrics](https://docs.aws.amazon.com/vpn/latest/s2svpn/monitoring-cloudwatch-vpn.html)
for them.

Azure on the other hand gives you some indicators in the connection's
`Resource health` section:

#### Mismatched IKE version showed this message on Azure

![Azure-VPN-wrong-IKE-version](/assets/images/azure/Azure-VPN-wrong-IKE-version.png)

#### Mismatched algorithms showed this message on Azure

![Azure-VPN-wrong-algorithms](/assets/images/azure/Azure-VPN-wrong-algorithms.png)
