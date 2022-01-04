3. Create an Instance
Create a directory for isolating terraform files.
$ mkdir ~/terraform && cd ~/terraform
Terraform code is written in a language called HCL in files with the extension “.tf”. It is a declarative language. The first step to using Terraform is typically to configure the provider you want to use. Create a file called “example.tf” and put the following code in it:
provider “aws” {
 access_key = “aws_access_key_id”
 secret_key = “aws_secret_access_key_id”
 region = “ap-south-1”
}
In here, we’re going to be using the AWS provider and that you wish to deploy your infrastructure in the “ap-south-1” region. For each provider, there are many different kinds of “resources” you can create, such as servers, databases, and load balancers.
resource “aws_instance” “web” {
 ami = “${lookup(var.amis,var.region)}”
 count = “${var.count}”
 key_name = “${var.key_name}”
 vpc_security_group_ids = [“${aws_security_group.instance.id}”]
 source_dest_check = false
 instance_type = “t2.micro”
tags {
 Name = “${format(“web-%03d”, count.index + 1)}”
 }
}
Create another file named “variables.tf”
variable “count” {
 default = 1
 }
variable “region” {
 description = “AWS region for hosting our your network”
 default = “ap-south-1”
}
variable “public_key_path” {
 description = “Enter the path to the SSH Public Key to add to AWS.”
 default = “/path_to_keyfile/keypair_name.pem”
}
variable “key_name” {
 description = “Key name for SSHing into EC2”
 default = “kaypair_name”
}
variable “amis” {
 description = “Base AMI to launch the instances”
 default = {
 ap-south-1 = “ami-8da8d2e2”
 }
}
NOTE: you need to create and download keypair using management console
In a terminal, go into the folder where you created example.tf, and run the “terraform plan” command:
$ terraform plan
Refreshing Terraform state in-memory prior to plan…
(...)
+ aws_instance.example
    ami:                      "ami-2d39803a"
    availability_zone:        "<computed>"
    ebs_block_device.#:       "<computed>"
    ephemeral_block_device.#: "<computed>"
    instance_state:           "<computed>"
    instance_type:            "t2.micro"
    key_name:                 "<computed>"
    network_interface_id:     "<computed>"
    placement_group:          "<computed>"
    private_dns:              "<computed>"
    private_ip:               "<computed>"
    public_dns:               "<computed>"
    public_ip:                "<computed>"
    root_block_device.#:      "<computed>"
    security_groups.#:        "<computed>"
    source_dest_check:        "true"
    subnet_id:                "<computed>"
    tenancy:                  "<computed>"
    vpc_security_group_ids.#: "<computed>"
Plan: 1 to add, 0 to change, 0 to destroy.
To actually create the instance, run the “terraform apply” command:
$ terraform apply
aws_instance.example: Creating…
…………
………. . . .
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
You can see your created instance in aws management console is up and running.
4. Create an AutoScaling and ELB
Running a single server is a good start, but in the real world, a single server is a single point of failure. If that server crashes, or if it becomes overwhelmed by too much traffic, users can no longer access your site. The solution is to run a cluster of servers, routing around servers that go down, and adjusting the size of the cluster up or down based on traffic.
The first step is creating an ASG is to create a launch configuration, which specifies how to configure each EC2 Instance in the ASG. From deploying the single EC2 Instance earlier, you already know exactly how to configure it.
The second step is creating a load balancer that is highly available and scalable is a lot of work.
Now, reopen your example.tf file and copy the following
provider "aws" {
  access_key = “aws_access_key_id”
  secret_key = “aws_secret_access_key_id”
  region     = "ap-south-1"
}
data "aws_availability_zones" "all" {}
### Creating EC2 instance
resource "aws_instance" "web" {
  ami               = "${lookup(var.amis,var.region)}"
  count             = "${var.count}"
  key_name               = "${var.key_name}"
  vpc_security_group_ids = ["${aws_security_group.instance.id}"]
  source_dest_check = false
  instance_type = "t2.micro"
tags {
    Name = "${format("web-%03d", count.index + 1)}"
  }
}
### Creating Security Group for EC2
resource "aws_security_group" "instance" {
  name = "terraform-example-instance"
  ingress {
    from_port = 8080
    to_port = 8080
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
## Creating Launch Configuration
resource "aws_launch_configuration" "example" {
  image_id               = "${lookup(var.amis,var.region)}"
  instance_type          = "t2.micro"
  security_groups        = ["${aws_security_group.instance.id}"]
  key_name               = "${var.key_name}"
  user_data = <<-EOF
              #!/bin/bash
              echo "Hello, World" > index.html
              nohup busybox httpd -f -p 8080 &
              EOF
  lifecycle {
    create_before_destroy = true
  }
}
## Creating AutoScaling Group
resource "aws_autoscaling_group" "example" {
  launch_configuration = "${aws_launch_configuration.example.id}"
  availability_zones = ["${data.aws_availability_zones.all.names}"]
  min_size = 2
  max_size = 10
  load_balancers = ["${aws_elb.example.name}"]
  health_check_type = "ELB"
  tag {
    key = "Name"
    value = "terraform-asg-example"
    propagate_at_launch = true
  }
}
## Security Group for ELB
resource "aws_security_group" "elb" {
  name = "terraform-example-elb"
  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port = 80
    to_port = 80
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
### Creating ELB
resource "aws_elb" "example" {
  name = "terraform-asg-example"
  security_groups = ["${aws_security_group.elb.id}"]
  availability_zones = ["${data.aws_availability_zones.all.names}"]
  health_check {
    healthy_threshold = 2
    unhealthy_threshold = 2
    timeout = 3
    interval = 30
    target = "HTTP:8080/"
  }
  listener {
    lb_port = 80
    lb_protocol = "http"
    instance_port = "8080"
    instance_protocol = "http"
  }
}
In variable.tf file
variable "count" {
    default = 1
  }
variable "region" {
  description = "AWS region for hosting our your network"
  default = "ap-south-1"
}
variable "public_key_path" {
  description = "Enter the path to the SSH Public Key to add to AWS."
  default = "/home/ratul/developments/devops/keyfile/ec2-core-app.pem"
}
variable "key_name" {
  description = "Key name for SSHing into EC2"
  default = "ec2-core-app"
}
variable "amis" {
  description = "Base AMI to launch the instances"
  default = {
  ap-south-1 = "ami-8da8d2e2"
  }
}
Also create a file named output.tf and copy the following
output "instance_ids" {
    value = ["${aws_instance.web.*.public_ip}"]
}
output "elb_dns_name" {
  value = "${aws_elb.example.dns_name}"
}
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
