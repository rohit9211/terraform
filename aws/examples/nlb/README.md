# Work with AWS NLB via terraform

A terraform module for making NLB.

## Usage
--------

Import the module and retrieve with ```terraform get``` or ```terraform get --update```. Adding a module resource to your template, e.g. `main.tf`:

```
#
# MAINTAINER Vitaliy Natarov "vitaliy.natarov@yahoo.com"
#
terraform {
  required_version = "> 0.9.0"
}
provider "aws" {
    region                  = "us-east-1"
    shared_credentials_file = "${pathexpand("~/.aws/credentials")}"    
    profile                 = "default"
}
module "iam" {
    source                          = "../../modules/iam"
    name                            = "My-Security"
    region                          = "us-east-1"
    environment                     = "PROD"

    aws_iam_role-principals         = [
        "ec2.amazonaws.com",
    ]
    aws_iam_policy-actions           = [
        "cloudwatch:GetMetricStatistics",
        "logs:DescribeLogStreams",
        "logs:GetLogEvents",
        "elasticache:Describe*",
        "rds:Describe*",
        "rds:ListTagsForResource",
        "ec2:DescribeAccountAttributes",
        "ec2:DescribeAvailabilityZones",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeVpcs",
        "ec2:Owner",
    ]
}
module "vpc" {
    source                              = "../../modules/vpc"
    name                                = "My"
    environment                         = "PROD"
    # VPC
    instance_tenancy                    = "default"
    enable_dns_support                  = "true"
    enable_dns_hostnames                = "true"
    assign_generated_ipv6_cidr_block    = "false"
    enable_classiclink                  = "false"
    vpc_cidr                            = "172.31.0.0/16"
    private_subnet_cidrs                = ["172.31.32.0/20"]
    public_subnet_cidrs                 = ["172.31.0.0/20"]
    availability_zones                  = ["us-east-1b"]
    enable_all_egress_ports             = "true"
    allowed_ports                       = ["8080", "3306", "443", "80"]

    map_public_ip_on_launch             = "true"

    #Internet-GateWay
    enable_internet_gateway             = "true"
    #NAT
    enable_nat_gateway                  = "false"
    single_nat_gateway                  = "true"
    #VPN
    enable_vpn_gateway                  = "false"
    #DHCP
    enable_dhcp_options                 = "false"
    # EIP
    enable_eip                          = "false"
}
module "ec2" {
    source                              = "../../modules/ec2"
    name                                = "TEST-Machine"
    region                              = "us-east-1"
    environment                         = "PROD"
    number_of_instances                 = 2
    ec2_instance_type                   = "t2.micro"
    enable_associate_public_ip_address  = "true"
    disk_size                           = "8"
    tenancy                             = "${module.vpc.instance_tenancy}"
    iam_instance_profile                = "${module.iam.instance_profile_id}"
    subnet_id                           = "${element(module.vpc.vpc-publicsubnet-ids, 0)}"
    #subnet_id                           = "${element(module.vpc.vpc-privatesubnet-ids, 0)}"
    #subnet_id                           = ["${element(module.vpc.vpc-privatesubnet-ids)}"]
    vpc_security_group_ids              = ["${module.vpc.security_group_id}"]

    monitoring                          = "true"
}
module "nlb" {
    source                  = "../../modules/nlb"
    name                    = "Load-Balancer"
    region                  = "us-east-1"
    environment             = "PROD"
    
    subnets                     = ["${module.vpc.vpc-privatesubnet-ids}"]
    vpc_id                      = "${module.vpc.vpc_id}"   
    enable_deletion_protection  = false

    backend_protocol    = "TCP"
    alb_protocols       = "TCP"
                                     
    target_ids          = ["${module.ec2.instance_ids}"]
}
```
Module Input Variables
----------------------
- `name` - Name to be used on all resources as prefix (`default = "TEST-ELB"`).
- `region` - The region where to deploy this code (e.g. us-east-1) (`default = "us-east-1"`).
- `environment` - Environment for service (`default = "STAGE"`).
- `orchestration` - Type of orchestration (`default = "Terraform"`).
- `subnets` - A list of subnet IDs to attach to the NLB (`efault = []`).
- `lb_internal` - If true, NLB will be an internal NLB (`default = false`).
- `name_prefix` - Creates a unique name beginning with the specified prefix. Conflicts with name (`default = "nlb"`).
- `enable_deletion_protection` - If true, deletion of the load balancer will be disabled via the AWS API. This will prevent Terraform from deleting the load balancer. Defaults to false (`default = false`).
- `load_balancer_type` - The type of load balancer to create. Possible values are application or network. The default value is application. However, we want to use network (`default  = "network"`).
- `idle_timeout` - The time in seconds that the connection is allowed to be idle. Default: 60 (`default = "60"`).
- `ip_address_type` - The type of IP addresses used by the subnets for your load balancer. The possible values are ipv4 and dualstack (`default = "ipv4"`).
- `timeouts_create` - Used for Creating LB. Default = 10mins (`default = "10m"`).
- `timeouts_update` - Used for LB modifications. Default = 10mins (`default = "10m"`).
- `timeouts_delete` - Used for LB destroying LB. Default = 10mins (`default = "10m"`).
- `vpc_id` - Set VPC ID for ?LB ().
- `alb_protocols` - A protocol the ALB accepts. (e.g.: TCP) - (`default = "TCP"`).
- `target_type` - The type of target that you must specify when registering targets with this target group. The possible values are instance (targets are specified by instance ID) or ip (targets are specified by IP address). The default is instance. Note that you can't specify targets for a target group using both instance IDs and IP addresses. If the target type is ip, specify IP addresses from the subnets of the virtual private cloud (VPC) for the target group, the RFC 1918 range (10.0.0.0/8, 172.16.0.0/12, and 192.168.0.0/16), and the RFC 6598 range (100.64.0.0/10). You can't specify publicly routable IP addresses (`default  = "instance"`).
- `deregistration_delay` - The amount time for Elastic Load Balancing to wait before changing the state of a deregistering target from draining to unused. The range is 0-3600 seconds. The default value is 300 seconds (`default = "300"`).
- `backend_port` - The port the service on the EC2 instances listen on (`default = 80`).
- `backend_protocol` - The protocol the backend service speaks. Options: HTTP, HTTPS, TCP, SSL (secure tcp) - (`default = "HTTP"`).
- `target_ids` - The ID of the target. This is the Instance ID for an instance, or the container ID for an ECS container. If the target type is ip, specify an IP address.
- `certificate_arn` - The ARN of the SSL Certificate. e.g. 'arn:aws:iam::XXXXXXXXXXX:server-certificate/ProdServerCert' (`default = ""`).
- `health_check_healthy_threshold` - Number of consecutive positive health checks before a backend instance is considered health (`default = 3`).
- `health_check_interval` - Interval in seconds on which the health check against backend hosts is tried (`default = 10`).
- `health_check_port` - The port used by the health check if different from the traffic-port (`default = "traffic-port"`).
- `health_check_unhealthy_threshold` - Number of consecutive positive health checks before a backend instance is considered unhealthy (`default  = 3`).


Authors
=======

Created and maintained by [Vitaliy Natarov](https://github.com/SebastianUA)
(vitaliy.natarov@yahoo.com).

License
=======

Apache 2 Licensed. See LICENSE for full details.
