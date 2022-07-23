---
title:  'A network interface may not specify both a network interface ID and a subnet'
toc: true
date:   2022-07-23 00:00:00 +0000
---
## Overview

This post will talk about error: `Launching EC2 instance failed. Status Reason:
A network interface may not specify both a network interface ID and a subnet`
on an AWS auto scaling group (ASG), see screenshot below:

![Error-in-ASG-Activity](/assets/images/ASG-no-ENI-and-subnet/00-Error-in-ASG-Activity.png)

## What happened?

I had recently learned a you can set an
[ENI ID in a launch template](https://www.pulumi.com/registry/packages/aws/api-docs/ec2/launchtemplate/#network_interfaces_python)
(this was new information to me).

Before I became aware of this whenever I wanted my EC2 instances to keep the
same private and public IP addresses after replacing said EC2 instances I would
create the Elastic Network Interface (ENI) separately from the launch template
and ASG, allow instances to describe ENIs and attach them to themselves (you
can imagine this turns into a complex IAM policy), have scripts readily
available in the instance to find the ENI, attach it to themselves and make
this ENI the primary network interface.

All the above to say, when I learned I could set the ENI directly into the
launch template I was happy I could simplify my setup. I still needed to create
the ENI outside the launch template and ASG, which I was already doing. This
meant I could update my current stacks and simplify them.

### Lets test in a sandbox environment

I updated my pulumi modules and made the new versions available to use.

I apply the changes running `pulumi up -s <my-stack>`, I get an error saying
something like cannot use `vpc_zone_identifiers` in an ASG that uses a launch
template with assigned ENIs

I change my pulumi modules again and replace `vpc_zone_identifiers` with
`availability_zones`, made new version available to use.

I apply the changes running `pulumi up -s <my-stack>`, this time the change
went through! :D

In this scenario, I was updating an ASG that manages only 1 EC2 instance. I try
to refresh the instance in my ASG and I notice it is taking a while for a new
instance to show up, I decide to go look in `Activity` and see the error:

```text
Launching EC2 instance failed. Status Reason:
A network interface may not specify both a network interface ID and a subnet
```

![Error-in-ASG-Activity](/assets/images/ASG-no-ENI-and-subnet/00-Error-in-ASG-Activity.png)

Which reminded me of the error about not being able to use
`vpc_zone_identifiers` and I had to replace that parameter with
`availability_zones`, I looked in the `Details` tab and noticed under `Network`
I had values in both `Availability Zones` and `Subnet ID`

![With-Subnet](/assets/images/ASG-no-ENI-and-subnet/01-ASG-With-SubnetID.png)

I asked a co-worker for help and they assumed we should be able to wipe the
subnet IDs from the ASG using the `aws` cli.

We tried:

- Passing only avalability zone without any subnet id, the subnet id stayed
- Passing avalability zone and empty subnet id, threw an error
- Passing avalability zone and `,` as the value for subnet id, no error but the
  subnet id stayed
- Change subnet id to a subnet id in a different availability zone than the one
  we want to use and then pass only the availability zone we want to use, that
  threw an error

At this point, we assumed the ASG was in a state we would not be able to get it
out from, we decided to replace the ASG

```shell
pulumi up --replace '**aws:autoscaling/group:Group::my-stack' -s <my-stack>
```

New ASG had values under `Availability Zones` and no values under `Subnet ID`
and a new instance was provisioned with the same ENI we were using before

![Without-Subnet](/assets/images/ASG-no-ENI-and-subnet/02-ASG-Without-SubnetID.png)
