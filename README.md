# AWS VPC Peering — Project 01

**EWB Courses · AWS Solutions Architect Associate · Internship project**
Author: **Om P. Patel** · Region: **ap-south-1 (Mumbai)** · Completed: 23 October 2026

Two VPCs with non-overlapping CIDRs are joined by a VPC peering connection. Private servers in each VPC talk to each other over **private IPs only**, and are administered from a laptop through a single **bastion host** using SSH `ProxyJump`. The whole setup is defined as code in [`cloudformation/vpc-peering.yaml`](cloudformation/vpc-peering.yaml).

---

## Architecture

![Architecture diagram](architecture/vpc-peering-diagram.png)

```
                 Internet
                    │  SSH 22 (my IP /32 only)
              ┌─────▼──────┐
              │ ewb-igw-a  │
┌─────────────┴────────────┴──────────────┐        ┌───────────────────────────────┐
│ VPC A  ewb-vpc-a  10.10.0.0/16          │        │ VPC B  ewb-vpc-b  10.20.0.0/16│
│                                         │        │ (no internet gateway)         │
│  Public subnet 10.10.1.0/24             │        │                               │
│   └─ ewb-bastion 10.10.1.10 (+public IP)│◄──────►│  Private subnet 10.20.1.0/24  │
│                                         │  pcx   │   └─ ewb-app-b 10.20.1.10     │
│  Private subnet 10.10.2.0/24            │ewb-a-b │                               │
│   └─ ewb-app-a 10.10.2.10               │ -peer  │                               │
└─────────────────────────────────────────┘        └───────────────────────────────┘
```

## Resources

| Resource | Name tag | CIDR / private IP |
|---|---|---|
| VPC A | ewb-vpc-a | 10.10.0.0/16 |
| VPC B | ewb-vpc-b | 10.20.0.0/16 |
| Public subnet (VPC A) | ewb-a-public | 10.10.1.0/24 |
| Private subnet (VPC A) | ewb-a-private | 10.10.2.0/24 |
| Private subnet (VPC B) | ewb-b-private | 10.20.1.0/24 |
| Internet gateway | ewb-igw-a | VPC A only |
| Peering connection | ewb-a-b-peer | VPC A ⇄ VPC B |
| Bastion | ewb-bastion | 10.10.1.10 + public IPv4 |
| App server A | ewb-app-a | 10.10.2.10 (no public IP) |
| App server B | ewb-app-b | 10.20.1.10 (no public IP) |

All instances: `t3.micro`, Amazon Linux 2023, IMDSv2 required.

## Route tables

| Route table | Destination | Target |
|---|---|---|
| ewb-rt-a-public | 10.10.0.0/16 | local |
| ewb-rt-a-public | 0.0.0.0/0 | ewb-igw-a |
| ewb-rt-a-public | 10.20.0.0/16 | ewb-a-b-peer |
| ewb-rt-a-private | 10.10.0.0/16 | local |
| ewb-rt-a-private | 10.20.0.0/16 | ewb-a-b-peer |
| ewb-rt-b-private | 10.20.0.0/16 | local |
| ewb-rt-b-private | 10.10.0.0/16 | ewb-a-b-peer |

The public route table also needs the peering route because with `ProxyJump` the **bastion itself** opens the SSH connection to App-B.

## Security groups (least privilege — every source is a /32)

| Security group | Inbound rule | Source |
|---|---|---|
| ewb-sg-bastion | SSH / TCP 22 | My public IPv4 /32 |
| ewb-sg-app-a | SSH / TCP 22 | 10.10.1.10/32 (bastion) |
| ewb-sg-app-a | ICMP Echo request | 10.20.1.10/32 (App-B) |
| ewb-sg-app-b | SSH / TCP 22 | 10.10.1.10/32 (bastion) |
| ewb-sg-app-b | ICMP Echo request | 10.10.2.10/32 (App-A) |

Nothing is open to `0.0.0.0/0`.

---

## Deploy it yourself

### Prerequisites
- AWS CLI v2 configured (`aws configure`)
- An existing EC2 key pair in the target region

### Create the stack

```bash
aws cloudformation deploy \
  --stack-name ewb-vpc-peering \
  --template-file cloudformation/vpc-peering.yaml \
  --region ap-south-1 \
  --parameter-overrides \
      KeyName=<your-key-pair-name> \
      MyIp=$(curl -s https://checkip.amazonaws.com)/32

# Get the bastion's public IP
aws cloudformation describe-stacks --stack-name ewb-vpc-peering --region ap-south-1 \
  --query "Stacks[0].Outputs[?OutputKey=='BastionPublicIp'].OutputValue" --output text
```

You can also upload the template in the CloudFormation console (**Create stack → Upload a template file**).

### SSH config

Copy [`ssh/config.example`](ssh/config.example) into `~/.ssh/config`, replace the bastion IP and key path, then:

```bash
ssh ewb-bastion    # bastion
ssh ewb-app-a      # App-A through the bastion
ssh ewb-app-b      # App-B through the bastion (crosses the peering)
```

The private key stays on the laptop; it is never copied to any instance.

---

## Test results

| # | Test | Where | Command | Expected | Result |
|---|---|---|---|---|---|
| 01 | Baseline (before peering routes) | App-A | `ping -c 4 10.20.1.10` | No replies | 4 sent, 0 received, 100% loss ✅ |
| 02 | Forward path | App-A | `ping -c 4 10.20.1.10` | 4 replies | 4 received, 0% loss ✅ |
| 03 | Reverse path | App-B | `ping -c 4 10.10.2.10` | 4 replies | 4 received, 0% loss ✅ |
| 04 | Private SSH | Laptop | `ssh ewb-app-b` | Shell on App-B | Connected through bastion ✅ |
| 05 | Isolation | EC2 console | Check public IPv4 | None on App-A / App-B | Empty on both ✅ |

Screenshots for every test are in [`screenshots/`](screenshots/) and the full write-up is in [`docs/project-report.pdf`](docs/project-report.pdf).

### Controlled failure and recovery

1. Removed **only** the ICMP Echo rule (source 10.10.2.10/32) from `ewb-sg-app-b`.
2. Ping from App-A → App-B: **100% packet loss**, even though peering and routes were unchanged.
3. Re-added the exact same rule → ping: **4 received, 0% loss**.

**Takeaway:** routing and security groups are separate layers, and both must allow the traffic.

---

## What I learned

1. **Peering is only the link.** An *Active* peering connection carries nothing until route tables point to it.
2. **Routes are needed in both directions.** Without the return route the reply has no path back.
3. **Security groups are a separate layer.** Correct routes still fail if the SG drops the packet.
4. **Least privilege.** /32 sources everywhere. App-B sees the bastion's private IP (10.10.1.10) as the SSH source because ProxyJump makes the bastion open its own connection.
5. **Private servers + bastion.** App servers have no public IP; VPC B has no IGW, and peering is not transitive, so VPC B gets no internet access through VPC A.
6. **Plan CIDRs up front.** Peering requires non-overlapping ranges. Troubleshoot in order: instance state → destination IP → route table → security group.

---

## Cleanup

```bash
aws cloudformation delete-stack --stack-name ewb-vpc-peering --region ap-south-1
aws cloudformation wait stack-delete-complete --stack-name ewb-vpc-peering --region ap-south-1
```

Deleting the stack removes every resource it created, in the correct dependency order. No NAT gateway is used, so the lab costs only a few cents per hour while running.

## Repository layout

```
aws-vpc-peering/
├── README.md
├── .gitignore
├── cloudformation/
│   └── vpc-peering.yaml     # Infrastructure as Code
├── ssh/
│   └── config.example       # ProxyJump SSH config
├── architecture/
│   └── vpc-peering-diagram.png
├── screenshots/             # Console + terminal evidence
└── docs/
    └── project-report.pdf   # Full project report
```
