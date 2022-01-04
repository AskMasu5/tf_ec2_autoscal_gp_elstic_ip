Create an AutoScaling and ELB:

Running a single server is a good start, but in the real world, a single server is a single point of failure. If that server crashes, or if it becomes overwhelmed by too much traffic, users can no longer access your site. The solution is to run a cluster of servers, routing around servers that go down, and adjusting the size of the cluster up or down based on traffic.
The first step is creating an ASG is to create a launch configuration, which specifies how to configure each EC2 Instance in the ASG. From deploying the single EC2 Instance earlier, you already know exactly how to configure it.
The second step is creating a load balancer that is highly available and scalable is a lot of work.
1. Create example.tf file 
2. Create variable.tf file

Now run terraform plan
$ terraform plan

Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.
aws_security_group.instance: Refreshing state... (ID: sg-54ed813c)
aws_security_group.elb: Refreshing state... (ID: sg-09ef8361)
data.aws_availability_zones.all: Refreshing state...
aws_launch_configuration.example: Refreshing state... (ID: terraform-20171010122248311700000001)
aws_instance.web: Refreshing state... (ID: i-0c4d35b212b045d78)
aws_elb.example: Refreshing state... (ID: terraform-asg-example)
aws_autoscaling_group.example: Refreshing state... (ID: tf-asg-20171010122315457300000002)
The Terraform execution plan has been generated and is shown below.
Resources are shown in alphabetical order for quick scanning. Green resources
will be created (or destroyed and then created if an existing resource
exists), yellow resources are being changed in-place, and red resources
will be destroyed. Cyan entries are data sources to be read.
Note: You didn't specify an "-out" parameter to save this plan, so when
"apply" is called, Terraform can't guarantee this is what will execute.
-/+ aws_instance.web (new resource required)
      ami:                               "ami-8da8d2e2" => "ami-8da8d2e2"
      associate_public_ip_address:       "true" => "<computed>"
      availability_zone:                 "ap-south-1a" => "<computed>"
      ebs_block_device.#:                "0" => "<computed>"
      ephemeral_block_device.#:          "0" => "<computed>"
      instance_state:                    "running" => "<computed>"
      instance_type:                     "t2.micro" => "t2.micro"
      ipv6_address_count:                "" => "<computed>"
      ipv6_addresses.#:                  "0" => "<computed>"
      key_name:                          "ec2-core-app" => "ec2-core-app"
      network_interface.#:               "0" => "<computed>"
      network_interface_id:              "eni-b523daea" => "<computed>"
      placement_group:                   "" => "<computed>"
      primary_network_interface_id:      "eni-b523daea" => "<computed>"
      private_dns:                       "ip-172-31-24-44.ap-south-1.compute.internal" => "<computed>"
      private_ip:                        "172.31.24.44" => "<computed>"
      public_dns:                        "ec2-13-126-108-226.ap-south-1.compute.amazonaws.com" => "<computed>"
      public_ip:                         "13.126.108.226" => "<computed>"
      root_block_device.#:               "1" => "<computed>"
      security_groups.#:                 "1" => "<computed>"
      source_dest_check:                 "false" => "false"
      subnet_id:                         "subnet-16814c7f" => "<computed>"
      tags.%:                            "1" => "1"
      tags.Name:                         "web-001" => "web-001"
      tenancy:                           "default" => "<computed>"
      user_data:                         "c765373c563b260626d113c4a56a46e8a8c5ca33" => "" (forces new resource)
      volume_tags.%:                     "0" => "<computed>"
      vpc_security_group_ids.#:          "0" => "1"
      vpc_security_group_ids.3652085476: "" => "sg-54ed813c"
Plan: 1 to add, 0 to change, 1 to destroy.

Run terraform plan
$ terraform plan
aws_security_group.instance: Refreshing state... (ID: sg-54ed813c)
aws_security_group.elb: Refreshing state... (ID: sg-09ef8361)
data.aws_availability_zones.all: Refreshing state...
aws_launch_configuration.example: Refreshing state... (ID: terraform-20171010122248311700000001)
aws_instance.web: Refreshing state... (ID: i-0c4d35b212b045d78)
aws_elb.example: Refreshing state... (ID: terraform-asg-example)
aws_autoscaling_group.example: Refreshing state... (ID: tf-asg-20171010122315457300000002)
aws_instance.web: Destroying... (ID: i-0c4d35b212b045d78)
aws_instance.web: Still destroying... (ID: i-0c4d35b212b045d78, 10s elapsed)
aws_instance.web: Still destroying... (ID: i-0c4d35b212b045d78, 20s elapsed)
aws_instance.web: Still destroying... (ID: i-0c4d35b212b045d78, 30s elapsed)
aws_instance.web: Still destroying... (ID: i-0c4d35b212b045d78, 40s elapsed)
aws_instance.web: Destruction complete after 47s
aws_instance.web: Creating...
  ami:                               "" => "ami-8da8d2e2"
  associate_public_ip_address:       "" => "<computed>"
  availability_zone:                 "" => "<computed>"
  ebs_block_device.#:                "" => "<computed>"
  ephemeral_block_device.#:          "" => "<computed>"
  instance_state:                    "" => "<computed>"
  instance_type:                     "" => "t2.micro"
  ipv6_address_count:                "" => "<computed>"
  ipv6_addresses.#:                  "" => "<computed>"
  key_name:                          "" => "ec2-core-app"
  network_interface.#:               "" => "<computed>"
  network_interface_id:              "" => "<computed>"
  placement_group:                   "" => "<computed>"
  primary_network_interface_id:      "" => "<computed>"
  private_dns:                       "" => "<computed>"
  private_ip:                        "" => "<computed>"
  public_dns:                        "" => "<computed>"
  public_ip:                         "" => "<computed>"
  root_block_device.#:               "" => "<computed>"
  security_groups.#:                 "" => "<computed>"
  source_dest_check:                 "" => "false"
  subnet_id:                         "" => "<computed>"
  tags.%:                            "" => "1"
  tags.Name:                         "" => "web-001"
  tenancy:                           "" => "<computed>"
  volume_tags.%:                     "" => "<computed>"
  vpc_security_group_ids.#:          "" => "1"
  vpc_security_group_ids.3652085476: "" => "sg-54ed813c"
aws_instance.web: Still creating... (10s elapsed)
aws_instance.web: Still creating... (20s elapsed)
aws_instance.web: Still creating... (30s elapsed)
aws_instance.web: Creation complete after 32s (ID: i-019e9bf03a9d3de32)
Apply complete! Resources: 1 added, 0 changed, 1 destroyed.
Outputs:
elb_dns_name = terraform-asg-example-2472445669.ap-south-1.elb.amazonaws.com
instance_ids = [
    52.66.14.13
]

The ELB is routing traffic to your EC2 Instances. Each time you hit the URL, it’ll pick a different Instance to handle the request. You now have a fully working cluster of web servers!
Copy elb_dns_name and paste in your browser
