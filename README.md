# How to manage Auto Scaling Group and Load Balancer with Terraform?

### What is Terraform?

Terraform is an open-source infrastructure as a code (IAC) tool that allows to create, manage & deploy the production-ready environment. Terraform codifies cloud APIs into declarative configuration files. Terraform can manage both existing service providers and custom in-house solutions.

Read more about Terraform from [here](https://www.terraform.io/docs/index.html)

### Prerequisite:

* Basic understanding of **AWS & Terraform**
* A server with **Terraform** pre-installed
* An **access key & secret key** created the AWS
* The **SSH key**

In this tutorial, I will be going to create various resources like **VPC, EC2, SG,** etc & will manage **ASG & ELB** using terraform. So, let’s begin the fun.

![Setup](https://github.com/DhruvinSoni30/Terrafrom-ELB-ASG/blob/main/1.png)


> We will use separate file for creating all the resources & a separate file for variables also. At the end I will discuss about variable file.

**Step 1:- Create Provider block**

* Create `provider.tf` file and add the below content to it

  ```
  provider "aws" {
    region = "us-east-1"
    access_key = "{}"
    secret_key = "{}"
    version = "v2.70.0"
  }
  ```

* Here I am using the 2.70.0 version of AWS because the entire code is written in the terraform 11 version.

**Step 2:- Create AWS VPC**

* Create `vpc.tf` file and add the below code to it

  ```
  resource "aws_vpc" "demovpc" {
     cidr_block       = "${var.vpc_cidr}"
     instance_tenancy = "default"
  tags = {
     Name = "Demo VPC"
   }
  }
  ``` 
  
**Step 3:- Create AWS Internet Gateway**

* Create `igw.tf` file and add the below code to it

  ```
  resource "aws_internet_gateway" "demogateway" {
    vpc_id = "${aws_vpc.demovpc.id}"
  }
  ``` 

* Here I am creating the Internet Gateway in the newly created VPC

**Step 4:- Create AWS Subnets**

* Create `subnet.tf` file and add the below code to it

  ```
  # Creating 1st subnet 
  resource "aws_subnet" "demosubnet" {
    vpc_id                  = "${aws_vpc.demovpc.id}"
    cidr_block             = "${var.subnet_cidr}"
    map_public_ip_on_launch = true
    availability_zone = "us-east-1a"
    tags = {
      Name = "Demo subnet"
    }
  }
  # Creating 2nd subnet 
  resource "aws_subnet" "demosubnet1" {
    vpc_id                  = "${aws_vpc.demovpc.id}"
    cidr_block             = "${var.subnet1_cidr}"
    map_public_ip_on_launch = true
    availability_zone = "us-east-1b"
    tags = {
      Name = "Demo subnet 1"
    }
  }
  ```
  
* Here I am creating 2 subnets and both will act as the public subnet

**Step 5:- Create AWS Route Table**

* Create `route_table.tf` file and add the below code to it

  ```
  #Creating Route Table
  resource "aws_route_table" "route" {
    vpc_id = "${aws_vpc.demovpc.id}"
    route {
        cidr_block = "0.0.0.0/0"
        gateway_id = "${aws_internet_gateway.demogateway.id}"
      }
    tags = {
        Name = "Route to internet"
      } 
  }
  resource "aws_route_table_association" "rt1" {
    subnet_id = "${aws_subnet.demosubnet.id}"
    route_table_id = "${aws_route_table.route.id}"
  }
  resource "aws_route_table_association" "rt2" {
    subnet_id = "${aws_subnet.demosubnet1.id}"
    route_table_id = "${aws_route_table.route.id}"
  }
  ```
  
* Here I am creating a new route table and associating the route table to the newly created subnets. Both newly created subnets will work as the public subnet.

**Step 6:- Create AWS Security Group for Load Balancer**

* Create `sg_elb.tf` file and add the below code to it
  
  ```
  # Creating Security Group for ELB
  resource "aws_security_group" "demosg1" {
    name        = "Demo Security Group"
    description = "Demo Module"
    vpc_id      = "${aws_vpc.demovpc.id}"
  
  # Inbound Rules
  # HTTP access from anywhere
    ingress {
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  # HTTPS access from anywhere
    ingress {
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  # SSH access from anywhere
    ingress {
      from_port   = 22
      to_port     = 22
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  # Outbound Rules
  # Internet access to anywhere
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  }
  ```
* Here I am creating inbound rules for ports 22,80 & 443 and opening outbound connection for all the ports for all the IPs.

**Step 7:- Create AWS Load Balancer**

* Create `elb.tf` file and add the below code to it
  
  ```
  resource "aws_elb" "web_elb" {
  name = "web-elb"
  security_groups = [
    "${aws_security_group.demosg1.id}"
  ]
  subnets = [
    "${aws_subnet.demosubnet.id}",
    "${aws_subnet.demosubnet1.id}"
  ]
  cross_zone_load_balancing   = true
  health_check {
    healthy_threshold = 2
    unhealthy_threshold = 2
    timeout = 3
    interval = 30
    target = "HTTP:80/"
  }
  listener {
    lb_port = 80
    lb_protocol = "http"
    instance_port = "80"
    instance_protocol = "http"
  }
  }
  ```
* The newly created application load balancer will require at least 2 subnets so, I am attaching both the subnets
* I have enabled cross-zone load balancing
* I have defined the health check policy so that I will always have healthy instances associated with my load balancer

**Step 8:- Create AWS Launch configuration**

* Create `launch_config.tf` file and add the below code to it

  ```
  resource "aws_launch_configuration" "web" {
    name_prefix = "web-"
    image_id = "ami-087c17d1fe0178315" 
    instance_type = "t2.micro"
    key_name = "tests"
    security_groups = [ "${aws_security_group.demosg.id}" ]
    associate_public_ip_address = true
    user_data = "${file("data.sh")}"
  lifecycle {
    create_before_destroy = true
  }
  }
  ```
* Here I am using AWS Linux 2 as the AMI instance and using user data for configuring the newly created instances. I will discuss the user data part later in the article.
* Key pair has already existed in the region
* I am using create_before_destroy here to create new instances from a new launch configuration before destroying the old ones.

**Step 9:- Create AWS Security group for EC2 instances**

* Create `sg_ec2.tf` file and add the below code to it

  ```
  # Creating Security Group for ELB
  resource "aws_security_group" "demosg1" {
    name        = "Demo Security Group"
    description = "Demo Module"
    vpc_id      = "${aws_vpc.demovpc.id}"
  # Inbound Rules
  # HTTP access from anywhere
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  # HTTPS access from anywhere
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  # SSH access from anywhere
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  # Outbound Rules
  # Internet access to anywhere
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  }
  ```
* Here I am creating inbound rules for ports **22,80 & 443** and opening outbound connection for all the ports for all the IPs.

**Step 10:- Create AWS Auto Scaling Group**

* Create `asg.tf` file and add the below code to it

  ```
  resource "aws_autoscaling_group" "web" {
    name = "${aws_launch_configuration.web.name}-asg"
    min_size             = 1
    desired_capacity     = 1
    max_size             = 2
  
    health_check_type    = "ELB"
    load_balancers = [
      "${aws_elb.web_elb.id}"
    ]
  launch_configuration = "${aws_launch_configuration.web.name}"
  enabled_metrics = [
    "GroupMinSize",
    "GroupMaxSize",
    "GroupDesiredCapacity",
    "GroupInServiceInstances",
    "GroupTotalInstances"
  ]
  metrics_granularity = "1Minute"
  vpc_zone_identifier  = [
    "${aws_subnet.demosubnet.id}",
    "${aws_subnet.demosubnet1.id}"
  ]
  # Required to redeploy without an outage.
  lifecycle {
    create_before_destroy = true
  }
  tag {
    key                 = "Name"
    value               = "web"
    propagate_at_launch = true
  }
  }
  ```
  
* There will be a minimum of 1 instance to serve the traffic.
* There will be at max 2 instancess to serve the traffic.
* Auto Scaling Group will be launched with 1instance
* Auto Scaling Group will get information about instance availability from the ELB
* I have set up a collection for some Cloud Watch metrics to monitor the Auto Scaling Group state.
* Each instance launched from this Auto Scaling Group will have Name a tag set to web.

**Step 11:- Create AWS Auto Scaling Policy**

* Create `asg_policy.tf` file and add the below code to it.

  ```
  resource "aws_autoscaling_policy" "web_policy_up" {
    name = "web_policy_up"
    scaling_adjustment = 1
    adjustment_type = "ChangeInCapacity"
    cooldown = 300
    autoscaling_group_name = "${aws_autoscaling_group.web.name}"
  }
  resource "aws_cloudwatch_metric_alarm" "web_cpu_alarm_up" {
    alarm_name = "web_cpu_alarm_up"
    comparison_operator = "GreaterThanOrEqualToThreshold"
    evaluation_periods = "2"
    metric_name = "CPUUtilization"
    namespace = "AWS/EC2"
    period = "120"
    statistic = "Average"
    threshold = "70"
  dimensions = {
    AutoScalingGroupName = "${aws_autoscaling_group.web.name}"
  }
  alarm_description = "This metric monitor EC2 instance CPU utilization"
  alarm_actions = [ "${aws_autoscaling_policy.web_policy_up.arn}" ]
  }
  resource "aws_autoscaling_policy" "web_policy_down" {
    name = "web_policy_down"
    scaling_adjustment = -1
    adjustment_type = "ChangeInCapacity"
    cooldown = 300
    autoscaling_group_name = "${aws_autoscaling_group.web.name}"
  }
  resource "aws_cloudwatch_metric_alarm" "web_cpu_alarm_down" {
    alarm_name = "web_cpu_alarm_down"
    comparison_operator = "LessThanOrEqualToThreshold"
    evaluation_periods = "2"
    metric_name = "CPUUtilization"
    namespace = "AWS/EC2"
    period = "120"
    statistic = "Average"
    threshold = "30"
  dimensions = {
    AutoScalingGroupName = "${aws_autoscaling_group.web.name}"
  }
  alarm_description = "This metric monitor EC2 instance CPU utilization"
  alarm_actions = [ "${aws_autoscaling_policy.web_policy_down.arn}" ]
  }
  ```
  
* `aws_autoscaling_policy` declares how AWS should change Auto Scaling Group instances count in when `aws_cloudwatch_metric_alarm` trigger.
* cooldown option will wait for 300 seconds before increasing Auto Scaling Group again.
* `aws_cloudwatch_metric_alarm` is an alarm, which will be fired, if the total CPU utilization of all instances in our Auto Scaling Group will be the greater or equal threshold value which is 70% during 120 seconds.
* `aws_cloudwatch_metric_alarm` is an alarm, which also will be fired, if the total CPU utilization of all instances in our Auto Scaling Group will be the lesser or equal threshold value which is 30% during 120 seconds.
 
**Step 12:- Create terraform variable file**

* Create `vars.tf` file and add the below code to it

  ```
  # Defining Public Key
  variable "public_key" {
    default = "tests.pub"
  }
  # Defining Private Key
  variable "private_key" {
    default = "tests.pem"
  }
  # Definign Key Name for connection
  variable "key_name" {
   default = "tests"
  }
  # Defining CIDR Block for VPC
  variable "vpc_cidr" {
    default = "10.0.0.0/16"
  }
  # Defining CIDR Block for Subnet
  variable "subnet_cidr" {
    default = "10.0.1.0/24"
  }
  # Defining CIDR Block for 2d Subnet
  variable "subnet1_cidr" {
    default = "10.0.2.0/24"
  }
  ```

**Step 13:- Create a user data file**

* Create `data.sh` file and add the below code to it

  ```
  sudo yum update -y
  sudo amazon-linux-extras install docker -y
  sudo service docker start
  sudo usermod -a -G docker ec2-user
  sudo chkconfig docker on
  sudo chmod 666 /var/run/docker.sock
  docker pull dhruvin30/dhsoniweb:v1
  docker run -d -p 80:80 dhruvin30/dhsoniweb:latest
  ```
* Here I am installing docker and running my portfolio website’s docker image

So, now our entire code is ready. We need to run the below steps to create the infrastructure.

* `terraform init` is to initialize the working directory and downloading plugins of the AWS provider
* `terraform plan` is to create the execution plan for our code
* `terraform apply` is to create the actual infrastructure. It will ask you to provide the Access Key and Secret Key in order to create the infrastructure. So,     instead of hardcoding the Access Key and Secret Key, it is better to apply at the run time.


After terraform apply completes you can verify the resources on the AWS console. Terraform will create the below resources.
* VPC
* Auto Scaling Group
* Launch Configurationn
* Auto Scaling Policy
* Load Balancer
* Internet Gateway
* Route Table
* Security Groups
* Subnets
* Cloud Watch Alarm

After the infrastructure is ready you can verify the output by navigating `http://DNS-of-Load-Balancer` you should see the below output.

![Output](https://github.com/DhruvinSoni30/Terrafrom-ELB-ASG/blob/main/2.png)

That’s it now, you have learned how to set up dynamic Auto Scaling Group and Load Balancer to distribute traffic to your instances in multiple Availability Zones. 
