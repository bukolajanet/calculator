
#Virtual Private Cloud
resource "aws_vpc" "And_Digital_vpc" {
  cidr_block           = var.vpc_cidr
  instance_tenancy     = "default"
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = {
    Name        = "${var.environment}-igw"
    Environment = var.environment
  }
}


#Internet Gateway
resource "aws_internet_gateway" "internet_gateway" {
  vpc_id = aws_vpc.And_vpc.id
  tags = {
    Name        = "${var.environment}-igw"
    Environment = var.environment
  }
}

#Elastic IP
resource "aws_elastic_ip" "nat_elastic_ip" {
  vpc        = true
  depends_on = [aws_internet_gateway.igw]
}

#NAT Gateway
resource "aws_nat_gateway" "nat_gateway" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id     = element(aws_subnet.public_subnet.*.id, 0)
  tags = {
    Name        = "${var.environment}-natgw"
    Environment = var.environment
  }
}

#Public subnet
resource "aws_subnet" "public_subnet" {
  vpc_id                  = aws_vpc.And_vpc.id
  count                   = length(var.public_subnets_cidr)
  cidr_block              = element(var.public_subnets_cidr, count.index)
  availability_zone       = element(var.availability_zones, count.index)
  map_public_ip_on_launch = true
  tags = {
    Name        = "${var.environment} -${element(var.availability_zones, count.index)}-public-subnet"
    Environment = var.environment
  }
}


#Private subnet
resource "aws_subnet" "private_subnet" {
  vpc_id                  = aws_vpc.And_vpc.id
  count                   = length(var.private_subnets_cidr)
  cidr_block              = element(var.private_subnets_cidr, count.index)
  availability_zone       = element(var.availability_zones, count.index)
  map_public_ip_on_launch = false
  tags = {
    Name        = "${var.environment}-${element(var.availability_zones, count.index)}-private-subnet"
    Environment = var.environment
  }
}


#Public subnet route table
resource "aws_route_table" "public_Route_table" {
  vpc_id = aws_vpc.And_vpc.id
  tags = {
    Name        = "${var.environment}-public_Route_table"
    Environment = var.environment
  }
}

resource "aws_route" "public_internet_gateway" {
  route_table_id         = aws_route_table.publicRT.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.igw.id
}

resource "aws_route_table_association" "public" {
  count          = length(var.public_subnets_cidr)
  subnet_id      = element(aws_subnet.public_subnet.*.id, count.index)
  route_table_id = aws_route_table.publicRT.id
}

#Private subnet route table
resource "aws_route_table" "private_Route_Table" {
  vpc_id = aws_vpc.And_vpc.id
  tags = {
    Name        = "${var.environment} -private_Route_Table"
    Environment = var.environment
  }
}

resource "aws_route" "private_nat_gateway" {
  route_table_id         = aws_route_table.privateRT.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.natgw.id
}

resource "aws_route_table_association" "private" {
  count          = length(var.private_subnets_cidr)
  subnet_id      = element(aws_subnet.private_subnet.*.id, count.index)
  route_table_id = aws_route_table.privateRT.id
}



#Elastic Load balancer
resource "aws_security_group" "Elb_security_group" {
  name        = "Elb_security_group"
  description = "Allow HTTP/HTTPS traffic to instances through Elastic Load Balancer"
  vpc_id      = aws_vpc.And_vpc.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.environment}-Elb_security_group"
    Environment = var.environment
  }
}

resource "aws_load_balancer" "EC2_Elastic_load-balancer" {
  name               = "EC2_Elastic_load_balancer"
  load_balancer_type = "application"
  security_groups    = [aws_security_group.Elb_security-group.id]
  subnets            = aws_subnet.public_subnet.*.id

}


resource "aws_lb_listener" "Listener_ELB" {
  load_balancer_arn = aws_lb.EC2_ELB.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-2022-02"
  certificate_arn   = aws_acm_certificate.janebuk.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.target_groupELB.arn
  }

}

resource "aws_lb_listener" "https_redirect" {
  load_balancer_arn = aws_lb.EC2_ELB.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"

    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

resource "aws_lb_target_group" "target_groupELB" {
  port     = 443
  protocol = "HTTPS"
  vpc_id   = aws_vpc.And_vpc.id

  health_check {
    healthy_threshold   = 2
    protocol            = "HTTPS"
    unhealthy_threshold = 2
  }

  lifecycle {
    create_before_destroy = true
  }
}




#Security Group
resource "aws_security_group" "EC2_Security_group" {
  name        = "${var.environment}-default-sg"
  description = "Allows SSH,HTTP, HTTPS "
  vpc_id      = aws_vpc.And_vpc.id
  ingress {
    from_port   = "0"
    to_port     = "0"
    protocol    = "tcp"
    cidr_blocks = [aws_security_group.ElbSG.id]
  }

  egress {
    from_port   = "0"
    to_port     = "0"
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.environment}-EC2_Security_group"
    Environment = var.environment
  }
}



#Auto scaling  Launch configuration
resource "aws_launch_configuration" "EC2_Launch_Configuration" {
  name_prefix = "test-"
  image_id      = var.EC2_ami
  instance_type = var.EC2_instance_type
  key_name      = "Sysops"
  security_groups             = [aws_security_group.EC2.id]
  associate_public_ip_address = false

  lifecycle {
    create_before_destroy = true
  }
}

#Auto Scaling Group
resource "aws_auto_scaling_group" "EC2_Auto_scaling_group" {
  name                 = "${aws_launch_configuration.EC2_LC.name}-asg"
  min_size             = 2
  desired_capacity     = 2
  max_size             = 4
  health_check_type    = "ELB"
  load_balancers       = [aws_lb.EC2_ELB.id]
  launch_configuration = aws_launch_configuration.EC2_LC.name
  vpc_zone_identifier  = aws_subnet.private_subnet.*.id
  target_group_arns    = [aws_lb_target_group.target_groupELB.arn]


  lifecycle {
    create_before_destroy = true
  }

}

#Auto scaling Policy
resource "aws_autoscaling_policy" "scale_up_policy" {
  name                   = "scale_up_policy"
  scaling_adjustment     = 2
  adjustment_type        = "ChangeInCapacity"
  cooldown               = var.cool_down
  autoscaling_group_name = aws_autoscaling_group.EC2_ASG.name
}

resource "aws_cloudwatch_metric_alarm" "scale_up_alarm" {
  alarm_name          = "scale_up_alarm"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = "2"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = var.scale_up_period
  statistic           = "Average"
  threshold           = var.GreaterThanOrEqualToThreshold

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.EC2_ASG.name
  }

  alarm_description = "This metric monitor EC2 instance CPU utilization"
  alarm_actions     = [aws_autoscaling_policy.scale_up_policy.arn]
}

resource "aws_autoscaling_policy" "scale_down_policy" {
  name                   = "scale_down_policy"
  scaling_adjustment     = -2
  adjustment_type        = "ChangeInCapacity"
  cooldown               = 300
  autoscaling_group_name = aws_autoscaling_group.EC2_ASG.name
}

resource "aws_cloudwatch_metric_alarm" "scale_down_alarm" {
  alarm_name          = "scale_down_alarm"
  comparison_operator = "LessThanOrEqualToThreshold"
  evaluation_periods  = "2"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = var.scale_down_period
  statistic           = "Average"
  threshold           = var.LessThanOrEqualToThreshold

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.EC2_ASG.name
  }

  alarm_description = "This metric monitor EC2 instance CPU utilization"
  alarm_actions     = [aws_autoscaling_policy.scale_down_policy.arn]
}


