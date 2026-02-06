# Project Summary: VM-Series Auto Scaling with GWLB Integration

## Objective
Deploy Palo Alto Networks VM-Series firewalls in AWS using Auto Scaling Groups (ASG) with Gateway Load Balancer (GWLB) integration for automated, scalable security inspection of inbound, outbound, and east-west traffic.

## What You're Building

### Architecture Components

**1. Auto Scaling VM-Series Firewalls**
- VM-Series instances that automatically scale based on traffic/CPU metrics
- Managed by AWS Auto Scaling Group (min: 1, desired: 2, max: 4)
- Automatically register to Panorama for centralized management
- Dynamic interface creation via Lambda function

**2. Gateway Load Balancer (GWLB) Integration**
- GWLB distributes traffic across VM-Series instances
- GWLB Endpoints (GWLBEs) in spoke VPCs for traffic inspection
- Transparent traffic inspection (no IP/port changes)

**3. Traffic Flows**
- **Inbound**: Internet → IGW → GWLBE → VM-Series → App
- **Outbound**: App → GWLBE → VM-Series → NAT/IGW → Internet
- **East-West**: App1 → TGW → GWLBE → VM-Series → TGW → App2

**4. Panorama Integration**
- Centralized management of all VM-Series instances
- Bootstrap configuration for automatic registration
- Template-based configuration (interfaces, zones, policies)
- Optional automated delicensing on scale-in

## Key Technical Requirements

### 1. Launch Template (LT)
**Purpose**: Define VM-Series instance configuration
- AMI selection (PAN-OS version)
- Instance type (m5.xlarge)
- First network interface (eth0/dataplane)
- Bootstrap parameters for Panorama
- IAM instance profile
- EBS encryption

### 2. Lambda Function
**Purpose**: Automate network interface management
- **On Launch**:
  - Disable source/dest check on eth0
  - Create & attach eth1 (management)
  - Create & attach eth2 (public/untrust)
  - Allocate EIPs if needed
  - Register to GWLB target group
- **On Terminate**:
  - Delicense from Panorama (optional)
  - Release EIPs
  - Deregister from target groups

### 3. Panorama Configuration
**Purpose**: Centralized firewall management
- Device Group: Groups all ASG VM-Series instances
- Template Stack: Network configuration
  - Interfaces: ethernet1/1 (trust), ethernet1/2 (untrust)
  - Sub-interfaces: .11, .12 (inbound), .20 (outbound), .30 (east-west)
  - Zones: trust, untrust, inbound, outbound, eastwest
  - Virtual Router with path monitoring per AZ
  - Security policies and NAT rules

### 4. GWLBE Mapping to Sub-interfaces
**Purpose**: Route specific traffic types to correct sub-interfaces
- **ethernet1/1.11** → App1 inbound traffic
- **ethernet1/1.12** → App2 inbound traffic
- **ethernet1/1.20** → All outbound traffic
- **ethernet1/1.30** → East-west (inter-VPC) traffic

## Required Terraform Modules

From `terraform-aws-swfw-modules` repository:

1. **`modules/gwlb/asg/`** - ASG with Launch Template & Lambda
2. **`modules/gwlb_endpoint_set/bootstrap/`** - Bootstrap S3 bucket & IAM
3. **`modules/gwlb/`** - Gateway Load Balancer
4. **`modules/gwlb_endpoint_set/`** - GWLB Endpoints
5. **`examples/centralized_design_autoscale/`** - Reference implementation

## Deployment Flow

```
1. Panorama Setup
   ↓
2. Bootstrap S3 Bucket (init-cfg.txt with Panorama credentials)
   ↓
3. Gateway Load Balancer (distributes traffic to VM-Series)
   ↓
4. GWLB Endpoints (in spoke VPCs for traffic interception)
   ↓
5. Launch Template + ASG (VM-Series instances)
   ↓
6. Lambda Execution (creates additional interfaces on launch)
   ↓
7. VM-Series Bootstrap (connects to Panorama, downloads config)
   ↓
8. Sub-interfaces Created (via Panorama template)
   ↓
9. Route Tables Updated (point traffic to GWLBEs)
   ↓
10. Traffic Flows Through VM-Series
```

## Key Benefits

**Automation**
- No manual VM-Series configuration
- Automatic scaling based on demand
- Self-healing (unhealthy instances replaced)

**High Availability**
- Multi-AZ deployment
- Path monitoring ensures correct routing per AZ
- Automatic failover

**Centralized Management**
- Single pane of glass via Panorama
- Consistent policy enforcement
- Simplified operations

**Cost Optimization**
- Scale down during low traffic
- Automated delicensing (optional)
- Pay only for what you use

## Critical Configuration Points

### Bootstrap Parameters
```
mgmt-interface-swap=enable          # Swap eth0/eth1
plugin-op-commands=...              # Enable GWLB plugins
panorama-server=<IP>                # Panorama primary
vm-auth-key=<KEY>                   # Authentication
dgname=<DEVICE_GROUP>               # Device group
tplname=<TEMPLATE_STACK>            # Template stack
```

### Interface Configuration
```
eth0 (device_index=0): Trust/Private (dataplane)
eth1 (device_index=1): Management
eth2 (device_index=2): Untrust/Public
```

### Sub-interface Mapping
```
ethernet1/1.11 → VLAN 11 → Inbound App1
ethernet1/1.12 → VLAN 12 → Inbound App2
ethernet1/1.20 → VLAN 20 → Outbound
ethernet1/1.30 → VLAN 30 → East-West
```

## Success Criteria

- [ ] VM-Series instances launch automatically
- [ ] Instances register to Panorama
- [ ] All 3 interfaces created per instance
- [ ] Sub-interfaces configured with correct VLANs
- [ ] GWLB health checks pass
- [ ] Traffic flows through VM-Series for inspection
- [ ] Auto-scaling triggers work (scale out/in)
- [ ] Delicensing works on scale-in (if enabled)

## End Result

A fully automated, scalable, highly available security architecture where:
- VM-Series firewalls automatically scale based on demand
- All traffic (inbound/outbound/east-west) is inspected
- Centralized management via Panorama
- Zero-touch provisioning of new firewall instances
- Automatic cleanup and delicensing on scale-in
