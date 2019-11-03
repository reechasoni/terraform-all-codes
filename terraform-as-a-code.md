## Creating an AWS Infrastructure for production like - Network setup, Application server setup, Creating and configuring Autosacling and Cloudwatch for application monitoring 

provider "aws" {
  region                  = "eu-west-1"
  profile                 = "terraform-user"
}

variable "VPCID" {
  default = "vpc-99465bfe"
}

variable "ExternalElbSGId" {
  default = "sg-7f7f4a04"
}

variable "AMIID" {
  default = "ami-1a962263"
}

variable "publicSG" {
  default = "sg-a4c7ecdf"
}

variable "userdatapath" {
  default = "C:\\Users\\saikiran\\Documents\\udemy\\userdata.txt"
}

variable "ASGAZs" {
    default = ["eu-west-1b", "eu-west-1a"]
}


variable "publicsubnetids" {
    default = ["subnet-e4b4bbad", "subnet-60aa9007"]
}


variable "DMZSubnetIds" {
  default = ["subnet-f2b48e95", "subnet-2bb6b962"]
}

variable "autoscaling_notification_arn"{
    default = "arn:aws:sns:eu-west-1:556788094947:AutoscalingActivity_Notification"
}


resource "aws_lb" "prodapp-alb" {
  name            = "App-TF-ALB-Prod"
  internal        = false
  security_groups = ["${var.ExternalElbSGId}"]
  subnets         = "${var.DMZSubnetIds}"

  enable_deletion_protection = true

  tags {
    Environment = "production"
  }
}

resource "aws_lb_target_group" "APP-TF-TG-Prod" {
  name     = "APP-TF-TG-Prod"
  port     = 80
  protocol = "HTTP"
  vpc_id   = "${var.VPCID}"

  health_check {
      interval = "30"
      path = "/"
      protocol = "HTTP"
      healthy_threshold = "2"
      unhealthy_threshold = "5"
      timeout = "28"
      matcher = "200"
  }
}

resource "aws_lb_listener" "APP-HTTP-Listener" {
  load_balancer_arn = "${aws_lb.prodapp-alb.arn}"
  port              = "80"
  protocol          = "HTTP"

  default_action {
    target_group_arn = "${aws_lb_target_group.APP-TF-TG-Prod.arn}"
    type             = "forward"
  }
}

resource "aws_lb_listener" "APP-HTTPS-Listener" {
  load_balancer_arn = "${aws_lb.prodapp-alb.arn}"
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-2016-08"
  certificate_arn   = "arn:aws:acm:eu-west-1:556788094947:certificate/6b7cd6f3-bf23-4047-81a9-586a916f5d7f"

  default_action {
    target_group_arn = "${aws_lb_target_group.APP-TF-TG-Prod.arn}"
    type             = "forward"
  }
}

resource "aws_launch_configuration" "App_LC_01-07-2018" {
  name          = "App_LC_01-07-2018"
  image_id      = "${var.AMIID}"
  instance_type = "t2.micro"
  iam_instance_profile = "Instance_Role_Prod"
  key_name = "vpctestkp"
  security_groups = ["${var.publicSG}"]
  user_data = "${file("${var.userdatapath}")}"
}

resource "aws_autoscaling_group" "app_asg_prod" {
  availability_zones        = "${var.ASGAZs}"
  name                      = "App_ASG_TF_Prod"
  max_size                  = 5
  min_size                  = 2
  health_check_grace_period = 300
  health_check_type         = "ELB"
  desired_capacity          = 2
  launch_configuration      = "${aws_launch_configuration.App_LC_01-07-2018.name}"
  target_group_arns         = ["${aws_lb_target_group.APP-TF-TG-Prod.arn}"]
  termination_policies      = ["OldestInstance"]
  vpc_zone_identifier        = "${var.publicsubnetids}"

  tag {
    key                 = "Environment"
    value               = "production"
    propagate_at_launch = true
  }

    tag {
    key                 = "Name"
    value               = "ApplicationProduction"
    propagate_at_launch = true
  }
}

resource "aws_autoscaling_notification" "asg_activity_notification" {
  group_names = [
    "${aws_autoscaling_group.app_asg_prod.name}"
  ]

  notifications = [
    "autoscaling:EC2_INSTANCE_LAUNCH",
    "autoscaling:EC2_INSTANCE_TERMINATE",
    "autoscaling:EC2_INSTANCE_LAUNCH_ERROR",
    "autoscaling:EC2_INSTANCE_TERMINATE_ERROR"
  ]

  topic_arn = "${var.autoscaling_notification_arn}"
}


resource "aws_cloudwatch_metric_alarm" "High_Cpu_Alarm" {
  alarm_name                = "App_High_CPU_Alarm"
  comparison_operator       = "GreaterThanOrEqualToThreshold"
  evaluation_periods        = "5"
  metric_name               = "CPUUtilization"
  namespace                 = "AWS/EC2"
  period                    = "60"
  statistic                 = "Average"
  threshold                 = "80"
  alarm_description         = "This metric monitors ec2 cpu utilization"
    
    dimensions {
        AutoScalingGroupName = "${aws_autoscaling_group.app_asg_prod.name}"
  }

  alarm_actions     = ["${aws_autoscaling_policy.app_scaleup_policy.arn}"]
}

resource "aws_autoscaling_policy" "app_scaleup_policy" {
  name                   = "app_scaleup_policy"
  scaling_adjustment     = 1
  adjustment_type        = "ChangeInCapacity"
  cooldown               = 300
  autoscaling_group_name = "${aws_autoscaling_group.app_asg_prod.name}"
}

resource "aws_cloudwatch_metric_alarm" "Low_Cpu_Alarm" {
  alarm_name                = "App_Low_CPU_Alarm"
  comparison_operator       = "LessThanOrEqualToThreshold"
  evaluation_periods        = "5"
  metric_name               = "CPUUtilization"
  namespace                 = "AWS/EC2"
  period                    = "60"
  statistic                 = "Average"
  threshold                 = "30"
  alarm_description         = "This metric monitors ec2 cpu utilization"
    
    dimensions {
        AutoScalingGroupName = "${aws_autoscaling_group.app_asg_prod.name}"
  }

    alarm_actions     = ["${aws_autoscaling_policy.app_scaledown_policy.arn}"]
}

resource "aws_autoscaling_policy" "app_scaledown_policy" {
  name                   = "app_scaledown_policy"
  scaling_adjustment     = -1
  adjustment_type        = "ChangeInCapacity"
  cooldown               = 300
  autoscaling_group_name = "${aws_autoscaling_group.app_asg_prod.name}"
}
