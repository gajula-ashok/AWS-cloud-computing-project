# AWS VPC Peering Project

A hands-on AWS networking project that builds two VPCs, connects them with VPC peering, and exposes one EC2 instance to the internet through an Internet Gateway. All resources are created through the AWS Management Console.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Network Plan](#network-plan)
- [Prerequisites](#prerequisites)
- [Implementation Steps](#implementation-steps)
- [Connectivity Tests](#connectivity-tests)
- [Screenshots](#screenshots)
- [Troubleshooting](#troubleshooting)
- [Cleanup](#cleanup)
- [Key Learnings](#key-learnings)

## Overview

The project demonstrates:

- Creating custom VPCs and subnets with non-overlapping CIDR blocks
- Making a subnet public with an Internet Gateway and route table
- Keeping a subnet private (no direct internet route)
- Connecting two VPCs privately with a VPC peering connection
- Controlling traffic with security groups
- Verifying connectivity with SSH and ping

## Architecture

```
 Client (C)
     |
 Internet Gateway
     |
+----------------------- VPC-1 (28.0.0.0/16) ------------------+        +--- VPC-2 (28.200.0.0/16) ---+
|                                                              |        |                             |
|  Public Subnet 28.0.1.0/24     Private Subnet 28.0.2.0/24    |  Peer  |  Private Subnet 28.200.0.0/16|
|  [ EC2-Public ] -------------> [ EC2-Private ] <-------------+--------+-> [ EC2-Peer ]              |
|                                                              |        |                             |
+--------------------------------------------------------------+        +-----------------------------+
```

Traffic flow:

1. The client connects to `EC2-Public` over the Internet Gateway.
2. `EC2-Public` reaches `EC2-Private` inside VPC-1.
3. `EC2-Private` and `EC2-Peer` communicate in both directions across the peering connection.

## Network Plan

| Resource | Name | CIDR / Details |
|---|---|---|
| VPC | `VPC-1` | `28.0.0.0/16` |
| VPC | `VPC-2` | `28.200.0.0/16` |
| Public subnet | `public-subnet` | `28.0.1.0/24` (VPC-1) |
| Private subnet | `private-subnet` | `28.0.2.0/24` (VPC-1) |
| Private subnet | `private-subnet-2` | `28.200.0.0/16` (VPC-2) |
| Internet Gateway | `IGW-1` | Attached to VPC-1 |
| Peering connection | `VPC1-VPC2-peer` | VPC-1 (requester) to VPC-2 (accepter) |
| Security group | `vpc1-sg` | VPC-1 |
| Security group | `vpc2-sg` | VPC-2 |

### Route tables

| Route table | Associated subnet | Routes |
|---|---|---|
| `public-rt` | `public-subnet` | `28.0.0.0/16` local, `0.0.0.0/0` to `IGW-1` |
| `private-rt` | `private-subnet` | `28.0.0.0/16` local, `28.200.0.0/16` to peering connection |
| VPC-2 route table | `private-subnet-2` | `28.200.0.0/16` local, `28.0.0.0/16` to peering connection |

### Security group rules (inbound)

**`vpc1-sg`**

| Type | Source | Purpose |
|---|---|---|
| SSH (22) | My IP | Access from the client to the public instance |
| SSH (22) | `28.0.0.0/16` | SSH between instances in VPC-1 |
| All ICMP - IPv4 | `28.0.0.0/16` | Ping within VPC-1 |
| All ICMP - IPv4 | `28.200.0.0/16` | Ping from VPC-2 |

**`vpc2-sg`**

| Type | Source | Purpose |
|---|---|---|
| SSH (22) | `28.0.0.0/16` | SSH from VPC-1 |
| All ICMP - IPv4 | `28.0.0.0/16` | Ping from VPC-1 |

### EC2 instances

| Name | VPC | Subnet | Public IP | Security group |
|---|---|---|---|---|
| `EC2-Public` | VPC-1 | `public-subnet` | Yes | `vpc1-sg` |
| `EC2-Private` | VPC-1 | `private-subnet` | No | `vpc1-sg` |
| `EC2-Peer` | VPC-2 | `private-subnet-2` | No | `vpc2-sg` |

All instances use Amazon Linux, `t2.micro` (free tier), and one shared key pair (`project-key.pem`).

## Prerequisites

- An AWS account with permission to manage VPC and EC2 resources
- A single AWS region used throughout (for example `ap-south-1`)
- An SSH client (or EC2 Instance Connect in the console)

## Implementation Steps

### 1. Create the VPCs

VPC console, **Your VPCs**, **Create VPC**, choose **VPC only**.

- `VPC-1` with `28.0.0.0/16`
- `VPC-2` with `28.200.0.0/16`

### 2. Create the subnets

**Subnets**, **Create subnet**.

- In `VPC-1`: `public-subnet` (`28.0.1.0/24`) and `private-subnet` (`28.0.2.0/24`), same Availability Zone
- In `VPC-2`: `private-subnet-2` (`28.200.0.0/16`)
- Select `public-subnet`, then **Actions**, **Edit subnet settings**, and enable **auto-assign public IPv4 address**

### 3. Create and attach the Internet Gateway

**Internet gateways**, **Create internet gateway** (`IGW-1`), then **Actions**, **Attach to VPC**, and choose `VPC-1`.

### 4. Create route tables for VPC-1

- Create `public-rt` in `VPC-1`, add route `0.0.0.0/0` to `IGW-1`, and associate it with `public-subnet`
- Create `private-rt` in `VPC-1` and associate it with `private-subnet`

### 5. Create the VPC peering connection

**Peering connections**, **Create peering connection**.

- Requester: `VPC-1`
- Accepter: My account, this Region, `VPC-2`
- Select the new connection, then **Actions**, **Accept request**. The status must show **Active**.

### 6. Add the peering routes (both sides)

- `private-rt`: destination `28.200.0.0/16`, target the peering connection (`pcx-...`)
- VPC-2 route table (associated with `private-subnet-2`): destination `28.0.0.0/16`, target the same `pcx-...`

### 7. Create the security groups

Security group names cannot start with `sg-`, so use `vpc1-sg` and `vpc2-sg`. A description is required. Add the inbound rules listed in the [Network Plan](#security-group-rules-inbound).

### 8. Launch the EC2 instances

**EC2**, **Launch instances**. Create the key pair `project-key` (RSA, `.pem`) on the first launch, then reuse it. In **Network settings**, choose **Select existing security group** (do not create a new one), and use the values from the [EC2 instances table](#ec2-instances).

## Connectivity Tests

Private instances have no public IP, so they are reached by hopping through `EC2-Public` using SSH agent forwarding.

**1. Client to `EC2-Public`**

```bash
chmod 400 project-key.pem
ssh -i project-key.pem ec2-user@<public-ip>
```

**2. `EC2-Public` to `EC2-Private`**

On your computer:

```bash
ssh-add project-key.pem
ssh -A ec2-user@<public-ip>
```

Inside `EC2-Public`:

```bash
ping -c 4 28.0.2.x
ssh ec2-user@28.0.2.x
```

**3. `EC2-Private` to `EC2-Peer`** (VPC-1 to VPC-2)

```bash
ping -c 4 28.200.x.x
```

**4. `EC2-Peer` to `EC2-Private`** (VPC-2 to VPC-1)

```bash
ssh ec2-user@28.200.x.x
ping -c 4 28.0.2.x
```

Expected result: all pings return replies with 0% packet loss.

## Screenshots

Place screenshots in a `screenshots/` folder and reference them here.

| # | Screenshot | File |
|---|---|---|
| 1 | VPC list with both CIDRs | `screenshots/01-vpcs.png` |
| 2 | Subnets list with VPC ID and CIDR | `screenshots/02-subnets.png` |
| 3 | Internet Gateway attached to VPC-1 | `screenshots/03-igw.png` |
| 4a | `public-rt` routes | `screenshots/04a-public-rt.png` |
| 4b | `private-rt` routes | `screenshots/04b-private-rt.png` |
| 4c | VPC-2 route table routes | `screenshots/04c-vpc2-rt.png` |
| 5 | Subnet associations | `screenshots/05-associations.png` |
| 6 | Peering connection (Active) | `screenshots/06-peering.png` |
| 7 | EC2 instances list | `screenshots/07-instances.png` |
| 8 | Security group inbound rules | `screenshots/08-security-groups.png` |
| 9a | SSH into `EC2-Public` | `screenshots/09a-ssh-public.png` |
| 9b | Public to private | `screenshots/09b-public-to-private.png` |
| 9c | Private to peer | `screenshots/09c-private-to-peer.png` |
| 9d | Peer to private | `screenshots/09d-peer-to-private.png` |
| 10 | VPC Resource map (optional) | `screenshots/10-resource-map.png` |

Example embed:

```markdown
![VPC list](screenshots/01-vpcs.png)
```

Before submitting, make sure the region is visible, and crop or blur your account ID and key pair names.

## Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| Instance launch fails: security group already exists | The wizard tried to create a group with an existing name. Choose **Select existing security group** instead. |
| Security group name rejected | Names cannot begin with `sg-`. Use `vpc1-sg` and `vpc2-sg`. |
| Cannot SSH to `EC2-Public` | Check the instance has a public IP, `public-rt` has the `0.0.0.0/0` route to `IGW-1`, and `vpc1-sg` allows SSH from your IP. |
| `Permission denied (publickey)` on the hop | Agent forwarding is not active. Run `ssh-add project-key.pem` and reconnect with `-A`. |
| Ping across the peering times out | Confirm the peering status is Active, both route tables have the peering route, and both security groups allow ICMP from the other VPC's CIDR. |
| SSH to `EC2-Peer` fails but ping works | `vpc2-sg` is missing SSH from `28.0.0.0/16`. |

## Cleanup

To avoid charges, delete resources in this order:

1. Terminate all three EC2 instances
2. Delete the peering connection
3. Detach and delete `IGW-1`
4. Delete the custom route tables and subnets
5. Delete the security groups and both VPCs

## Key Learnings

- A subnet is "public" because its route table points to an Internet Gateway, not because of its name.
- VPC peering needs routes on **both** sides plus matching security group rules.
- VPC peering is not transitive, and the two VPCs must not have overlapping CIDR blocks.
- Security groups are stateful, so reply traffic is allowed automatically.
- Note: `28.0.0.0/8` is public address space. It works inside AWS for this lab, but the private ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`) are the recommended choice for real projects.

## Author

GAJULA ASHOK
AWS Cloud Computing
