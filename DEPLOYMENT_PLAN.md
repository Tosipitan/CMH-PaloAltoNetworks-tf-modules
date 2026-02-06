# VM-Series ASG with GWLBE Deployment Plan

## Prerequisites

### 1. Panorama Setup
- [ ] Deploy Panorama (use `examples/panorama_standalone`)
- [ ] Create Device Group: `vmseries-asg-dg`
- [ ] Create Template Stack: `vmseries-asg-template`
- [ ] Configure Template with:
  - Network interfaces (ethernet1/1, ethernet1/2)
  - Sub-interfaces (ethernet1/1.11, .12, .20, .30)
  - Zones (trust, untrust, inbound, outbound, eastwest)
  - Virtual router with static routes + path monitoring
  - Interface management profile for health checks
  - CloudWatch plugin configuration
- [ ] Generate VM Auth Key: `Panorama > Device Deployment > Bootstrap`
- [ ] Install plugin: `sw_fw_license` (if using delicensing)
- [ ] Configure License Manager (if using delicensing)

### 2. AWS Prerequisites
- [ ] VPC with subnets (mgmt, private/trust, public/untrust) in 2+ AZs
- [ ] Security groups (mgmt, trust, untrust)
- [ ] SSH key pair
- [ ] S3 bucket for bootstrap (optional)
- [ ] SSM Parameter for delicensing credentials (optional)

### 3. Network Architecture
- [ ] Transit Gateway (if centralized design)
- [ ] TGW attachments and route tables
- [ ] VPC route tables configured

---

## Deployment Steps

### Phase 1: Bootstrap Configuration (Optional but Recommended)

**Module:** `modules/gwlb_endpoint_set/bootstrap/`

```hcl
module "bootstrap" {
  source = "./modules/gwlb_endpoint_set/bootstrap"

  bootstrap_options = {
    hostname                    = "vmseries-asg"
    mgmt-interface-swap         = "enable"
    plugin-op-commands          = "panorama-licensing-mode-on,aws-gwlb-inspect:enable,aws-gwlb-overlay-routing:enable"
    panorama-server             = "10.255.0.10"
    panorama-server-2           = "10.255.0.11"
    vm-auth-key                 = "<GENERATED_AUTH_KEY>"
    dgname                      = "vmseries-asg-dg"
    tplname                     = "vmseries-asg-template"
    dhcp-send-hostname          = "yes"
    dhcp-send-client-id         = "yes"
    dhcp-accept-server-hostname = "yes"
    dhcp-accept-server-domain   = "yes"
  }

  prefix      = "vmseries-asg-"
  global_tags = { Environment = "prod" }
}
```

**Deploy:**
```bash
terraform init
terraform plan -target=module.bootstrap
terraform apply -target=module.bootstrap
```

**Outputs needed:** `bucket_name`, `instance_profile_name`

---

### Phase 2: Gateway Load Balancer

**Module:** `modules/gwlb/`

```hcl
module "gwlb" {
  source = "./modules/gwlb"

  name         = "security-gwlb"
  vpc          = module.vpc.id
  subnet_group = module.subnet_sets["gwlb"].subnets

  health_check_port     = 80
  health_check_protocol = "HTTP"
  
  global_tags = { Environment = "prod" }
}
```

**Deploy:**
```bash
terraform plan -target=module.gwlb
terraform apply -target=module.gwlb
```

**Outputs needed:** `gwlb_arn`, `endpoint_service_name`

---

### Phase 3: GWLB Endpoints

**Module:** `modules/gwlb_endpoint_set/`

```hcl
module "gwlbe_inbound_app1" {
  source = "./modules/gwlb_endpoint_set"

  name                = "app1-inbound"
  gwlb_service_name   = module.gwlb.endpoint_service_name
  vpc                 = module.app1_vpc.id
  subnet_group        = module.subnet_sets["app1_gwlbe"].subnets
  act_as_next_hop     = true
  
  global_tags = { Environment = "prod", Traffic = "inbound" }
}

module "gwlbe_outbound" {
  source = "./modules/gwlb_endpoint_set"

  name                = "security-outbound"
  gwlb_service_name   = module.gwlb.endpoint_service_name
  vpc                 = module.security_vpc.id
  subnet_group        = module.subnet_sets["gwlbe_outbound"].subnets
  
  global_tags = { Environment = "prod", Traffic = "outbound" }
}

module "gwlbe_eastwest" {
  source = "./modules/gwlb_endpoint_set"

  name                = "security-eastwest"
  gwlb_service_name   = module.gwlb.endpoint_service_name
  vpc                 = module.security_vpc.id
  subnet_group        = module.subnet_sets["gwlbe_eastwest"].subnets
  
  global_tags = { Environment = "prod", Traffic = "eastwest" }
}
```

**Deploy:**
```bash
terraform plan -target=module.gwlbe_inbound_app1 -target=module.gwlbe_outbound -target=module.gwlbe_eastwest
terraform apply -target=module.gwlbe_inbound_app1 -target=module.gwlbe_outbound -target=module.gwlbe_eastwest
```

**Outputs needed:** Endpoint IDs per AZ

---

### Phase 4: Launch Template & ASG

**Module:** `modules/gwlb/asg/`

```hcl
module "vmseries_asg" {
  source = "./modules/gwlb/asg"

  name_prefix      = "vmseries-"
  region           = "us-east-1"
  ssh_key_name     = "my-key"
  vmseries_version = "11.1.4-h7"
  
  # Bootstrap
  bootstrap_options = join(",", [
    "type=dhcp-client",
    "hostname=vmseries-asg",
    "mgmt-interface-swap=enable",
    "plugin-op-commands=panorama-licensing-mode-on,aws-gwlb-inspect:enable,aws-gwlb-overlay-routing:enable",
    "panorama-server=10.255.0.10",
    "vm-auth-key=<AUTH_KEY>",
    "dgname=vmseries-asg-dg",
    "tplname=vmseries-asg-template"
  ])

  # Network Interfaces
  interfaces = {
    private = {
      device_index       = 0
      subnet_id          = {
        "us-east-1a" = "subnet-private-1a"
        "us-east-1b" = "subnet-private-1b"
      }
      security_group_ids = ["sg-private"]
      create_public_ip   = false
      source_dest_check  = false
    }
    mgmt = {
      device_index       = 1
      subnet_id          = {
        "us-east-1a" = "subnet-mgmt-1a"
        "us-east-1b" = "subnet-mgmt-1b"
      }
      security_group_ids = ["sg-mgmt"]
      create_public_ip   = true
      source_dest_check  = true
    }
    public = {
      device_index       = 2
      subnet_id          = {
        "us-east-1a" = "subnet-public-1a"
        "us-east-1b" = "subnet-public-1b"
      }
      security_group_ids = ["sg-public"]
      create_public_ip   = false
      source_dest_check  = false
    }
  }

  # ASG Configuration
  desired_capacity = 2
  min_size         = 1
  max_size         = 4
  
  # Target Group (GWLB)
  target_group_arn = module.gwlb.target_group_arn

  # Lambda Configuration
  lambda_timeout                 = 30
  reserved_concurrent_executions = 100
  
  # Delicensing (Optional)
  delicense_enabled        = true
  delicense_ssm_param_name = "/panorama/delicense-config"
  
  # VPC for Lambda (if delicensing enabled)
  subnet_ids         = ["subnet-lambda-1a", "subnet-lambda-1b"]
  security_group_ids = ["sg-lambda"]

  # Scaling Plan
  scaling_plan_enabled              = true
  scaling_metric_name               = "DataPlaneCPUUtilizationPct"
  scaling_target_value              = 75
  scaling_statistic                 = "Average"
  scaling_estimated_instance_warmup = 900
  scaling_cloudwatch_namespace      = "asg-vmseries"
  scaling_tags                      = { ManagedBy = "terraform" }

  global_tags = { Environment = "prod" }
}
```

**Deploy:**
```bash
# Install Lambda dependencies first
cd modules/gwlb/asg/scripts
pip install -r requirements.txt -t .
cd ../../../..

# Deploy ASG
terraform plan -target=module.vmseries_asg
terraform apply -target=module.vmseries_asg
```

---

### Phase 5: Sub-interface to GWLBE Mapping

**Configuration in ASG module variables:**

```hcl
# This is handled by Lambda automatically based on subinterface naming
# Lambda maps based on environment variables passed to it

# In your main.tf, configure the mapping:
locals {
  subinterface_gwlbe_mapping = {
    "ethernet1/1.11" = module.gwlbe_inbound_app1.endpoint_ids["us-east-1a"]
    "ethernet1/1.12" = module.gwlbe_inbound_app2.endpoint_ids["us-east-1a"]
    "ethernet1/1.20" = module.gwlbe_outbound.endpoint_ids["us-east-1a"]
    "ethernet1/1.30" = module.gwlbe_eastwest.endpoint_ids["us-east-1a"]
  }
}
```

**Note:** Lambda creates interfaces dynamically. Sub-interface to GWLBE mapping is configured in Panorama template and applied via bootstrap.

---

### Phase 6: Route Tables

Update VPC route tables to point to GWLB endpoints:

```hcl
# Inbound traffic (IGW → GWLBE → VM-Series → App)
resource "aws_route" "igw_to_gwlbe" {
  route_table_id         = aws_route_table.igw.id
  destination_cidr_block = "10.104.0.0/16"
  vpc_endpoint_id        = module.gwlbe_inbound_app1.endpoint_ids["us-east-1a"]
}

# Outbound traffic (App → GWLBE → VM-Series → NAT/IGW)
resource "aws_route" "app_to_gwlbe" {
  route_table_id         = aws_route_table.app1.id
  destination_cidr_block = "0.0.0.0/0"
  vpc_endpoint_id        = module.gwlbe_outbound.endpoint_ids["us-east-1a"]
}

# East-West traffic (App1 → TGW → GWLBE → VM-Series → TGW → App2)
resource "aws_ec2_transit_gateway_route" "to_gwlbe" {
  destination_cidr_block         = "10.105.0.0/16"
  transit_gateway_attachment_id  = module.tgw_attachment_security.id
  transit_gateway_route_table_id = module.tgw.route_table_ids["from_spokes"]
}
```

**Deploy:**
```bash
terraform plan
terraform apply
```

---

## Verification Steps

### 1. Verify ASG
```bash
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names vmseries-asg
```

### 2. Verify VM-Series Instances
```bash
aws ec2 describe-instances --filters "Name=tag:Name,Values=vmseries*"
```

### 3. Verify Lambda Execution
```bash
aws logs tail /aws/lambda/vmseries-asg_actions --follow
```

### 4. Verify GWLB Target Health
```bash
aws elbv2 describe-target-health --target-group-arn <GWLB_TG_ARN>
```

### 5. Verify Panorama Registration
- Login to Panorama
- Navigate to `Panorama > Managed Devices`
- Verify VM-Series instances are connected
- Check device group membership

### 6. Verify Sub-interfaces
- SSH to VM-Series: `ssh admin@<MGMT_IP>`
- Run: `show interface all`
- Verify: ethernet1/1.11, .12, .20, .30 are up with DHCP IPs

### 7. Test Traffic Flow
```bash
# From spoke VPC instance
curl http://example.com
# Check VM-Series logs for traffic
```

---

## Rollback Plan

### If deployment fails:

1. **Destroy ASG first:**
   ```bash
   terraform destroy -target=module.vmseries_asg
   ```

2. **Destroy GWLBEs:**
   ```bash
   terraform destroy -target=module.gwlbe_inbound_app1
   terraform destroy -target=module.gwlbe_outbound
   terraform destroy -target=module.gwlbe_eastwest
   ```

3. **Destroy GWLB:**
   ```bash
   terraform destroy -target=module.gwlb
   ```

4. **Destroy Bootstrap:**
   ```bash
   terraform destroy -target=module.bootstrap
   ```

---

## Troubleshooting

### Issue: VM-Series not registering to Panorama
- Check bootstrap S3 bucket access
- Verify VM Auth Key is correct
- Check Panorama connectivity from mgmt subnet
- Review `/var/log/pan/bootstrap.log` on VM-Series

### Issue: Lambda timeout
- Increase `lambda_timeout` variable
- Check Lambda VPC connectivity
- Review CloudWatch logs

### Issue: GWLB health checks failing
- Verify interface management profile allows HTTP/ping
- Check security group rules on trust interface
- Verify GWLB target group health check settings

### Issue: Sub-interfaces not created
- Check Lambda logs for errors
- Verify interface configuration in variables
- Ensure subnet IDs are correct for each AZ

### Issue: Traffic not flowing through VM-Series
- Verify route tables point to GWLB endpoints
- Check VM-Series security policies
- Verify NAT rules configured
- Check path monitoring status: `show routing route`

---

## Post-Deployment

- [ ] Configure security policies in Panorama
- [ ] Configure NAT rules
- [ ] Enable logging and monitoring
- [ ] Set up CloudWatch alarms for scaling metrics
- [ ] Test failover scenarios
- [ ] Document custom configurations
- [ ] Schedule regular backups of Panorama configuration
