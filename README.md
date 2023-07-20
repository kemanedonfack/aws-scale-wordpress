# AWS Scale WordpPress

## Introduction

In today's article, we will explore the scalability possibilities of deploying WordPress on AWS. Building upon our previous article on deploying WordPress on a 2-Tier AWS architecture with Terraform, we will focus on utilizing the Auto Scaling Group (ASG) feature, along with leveraging Amazon S3 for media storage and CloudFront for caching. These enhancements will enable us to scale our WordPress deployment effectively and handle increasing traffic demands. So let's dive in!

If you haven't read the previous article, "Deploying WordPress on a 2-Tier AWS Architecture with Terraform," we highly recommend checking it out first. It provides a comprehensive guide on setting up the initial 2-Tier architecture, which forms the foundation for this scalability enhancement.

[Link to the previous article: Deploying WordPress on a 2-Tier AWS Architecture with Terraform]


## Prerequisites

Before we proceed with scaling our WordPress deployment on AWS, make sure you have the following prerequisites in place:
- An AWS account with appropriate permissions to create resources.
- Basic knowledge of Terraform and its concepts.
- Familiarity with WordPress and its deployment on AWS, as discussed in our previous article.

## Using Auto Scaling Group for Scalability

Auto Scaling Group (ASG) is a powerful AWS feature that allows you to automatically adjust the number of instances in your application fleet based on demand. By using ASG, we can ensure our WordPress deployment can scale horizontally to handle varying loads.

In this section, we will explain the modifications made to the existing Terraform files to incorporate ASG into our WordPress deployment.

If you would like to delve deeper into the concept of Auto Scaling Group and its benefits, I have a dedicated article that covers it in detail. It provides valuable insights into how ASG works, and its configuration options.

[Link to the article: Understanding Auto Scaling Group in AWS]

### Step 1: Modifying main.tf

In the main.tf file, we need to remove our 2 aws_instance resources and add some new resources : aws_launch_template to define the configuration for our EC2 instances The template specifies the AMI, instance type, security groups, and other parameters. 

```
resource "aws_launch_template" "instances_configuration" {
  name_prefix            = "asg-instance"
  image_id               = var.ami
  key_name               = aws_key_pair.aws_ec2_access_key.id
  instance_type          = var.instance_type
  vpc_security_group_ids = [aws_security_group.production-instance-sg.id]

  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Name = "asg-instance"
  }
}
```

In the same main.tf file, we will create the Autoscaling Group (aws_autoscaling_group) resource. The ASG will be responsible for managing the number of instances and ensuring they match the desired capacity.

`main.tf`
```
resource "aws_autoscaling_group" "asg" {
  name                      = "asg"
  min_size                  = 2
  max_size                  = 4
  desired_capacity          = 2
  health_check_grace_period = 150
  health_check_type         = "ELB"
  vpc_zone_identifier       = [aws_subnet.ec2_1_public_subnet.id, aws_subnet.ec2_2_public_subnet.id]

  launch_template {
    id      = aws_launch_template.instances_configuration.id
    version = "$Latest"
  }

  tag {
    key                 = "Name"
    value               = "production-instance"
    propagate_at_launch = true
  }

  depends_on = [
    aws_db_instance.rds_master,
  ]
}
```

Next, we'll define an Auto Scaling policy that adjusts the number of instances based on CPU utilization.

`main.tf`

```
resource "aws_autoscaling_policy" "avg_cpu_policy_greater" {
  name                   = "avg-cpu-policy-greater"
  policy_type            = "TargetTrackingScaling"
  autoscaling_group_name = aws_autoscaling_group.asg.id

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value = 50.0
  }
}
```

### Step 2: Modifying loadbalancer.tf

we modified the loadbalancer.tf to ensure that the ALB is properly configured to handle incoming HTTP traffic on port 80 and forward it to the instances registered with the ASG. The ALB acts as the entry point for client requests and distributes the traffic evenly across the instances in the ASG.

```
resource "aws_lb" "alb" {
  name               = "asg-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb_sg.id]
  subnets            = [aws_subnet.ec2_1_public_subnet.id, aws_subnet.ec2_2_public_subnet.id]
}

resource "aws_lb_listener" "alb_listener" {
  load_balancer_arn = aws_lb.alb.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.alb_target_group.arn
  }
}

resource "aws_lb_target_group" "alb_target_group" {
  name     = "asg-target-group"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.infrastructure_vpc.id
}
```

`security_group.tf`

we have also created a security group for the ALB

```
resource "aws_security_group" "alb_sg" {
  name = "asg-alb-sg"
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

  vpc_id = aws_vpc.infrastructure_vpc.id
}
```

### Step 3: Modifying efs.tf

In this step, we make several modifications to the efs.tf file to ensure compatibility with the latest version of WordPress and integrate it with the Auto Scaling Group (ASG) setup.

Firstly, we update the installation script to use the latest version of WordPress. This update is necessary because the WP Offload S3 Lite plugin, which we will be using, requires the latest version of WordPress. Here is the modified installation script:

```
resource "null_resource" "install_script" {
  count = 2

  depends_on = [
    aws_db_instance.rds_master,
    local_file.private_key,
    aws_efs_mount_target.efs_mount_target_1,
    aws_efs_mount_target.efs_mount_target_2,
    aws_autoscaling_group.asg
  ]

  connection {
    type        = "ssh"
    host        = data.aws_instances.production_instances.public_ips[count.index]
    user        = "ec2-user"
    private_key = file(var.private_key_location)
  }

  provisioner "remote-exec" {
    inline = [
      "sudo yum update -y",
      "sudo yum install docker -y",
      "wget https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)",
      "sudo mv docker-compose-$(uname -s)-$(uname -m) /usr/local/bin/docker-compose",
      "sudo chmod -v +x /usr/local/bin/docker-compose",
      "sudo systemctl enable docker.service",
      "sudo systemctl start docker.service",
      "sudo yum -y install amazon-efs-utils",
      "sudo mkdir -p ${var.mount_directory}",
      "sudo mount -t efs -o tls ${aws_efs_file_system.efs_volume.id}:/ ${var.mount_directory}",
      "sudo docker run --restart=always --name wordpress-docker -e WORDPRESS_DB_USER=${aws_db_instance.rds_master.username} -e WORDPRESS_DB_HOST=${aws_db_instance.rds_master.endpoint} -e WORDPRESS_DB_PASSWORD=${aws_db_instance.rds_master.password} -e WORDPRESS_DB_NAME=${aws_db_instance.rds_master.db_name} -v ${var.mount_directory}:${var.mount_directory} -p 80:80 -d wordpress",
    ]
  }
}

```

Additionally, we replace the reference to the previous EC2 instances with the Auto Scaling Group in the following data block:

```
data "aws_instances" "production_instances" {
  instance_tags = {
    "Name" = "production-instance"
  }
  depends_on = [
    aws_autoscaling_group.asg
  ]
}
```

## Creating S3 Bucket and CloudFront distribution




### Step 7: Deployment

To start the deployment process, we need to initialize Terraform in our project directory. This step ensures that Terraform downloads the necessary providers and sets up the backend configuration. Run the following command in your terminal:

```
terraform init
```

![terraform-init](./images/terraform-init.png)

After initializing Terraform, we can generate an execution plan to preview the changes that will be made to our AWS infrastructure. The plan provides a detailed overview of the resources that will be created, modified, or destroyed.

To generate the plan, execute the following command:
```
terraform plan
```

![terraform-plan](./images/terraform-plan-2.png)

![terraform-plan](./images/terraform-plan-1.png)

Once we are satisfied with the execution plan, we can proceed with deploying our infrastructure on AWS. Terraform will provide the necessary resources and configure them according to our specifications.

To deploy the infrastructure, run the following command:

```
terraform apply
```

Terraform will prompt for confirmation before proceeding with the deployment. Type `yes` and press Enter to continue.


![terraform-apply](./images/terraform-apply-1.png)

The deployment process will take some time. Terraform will display the progress and status of each resource being created.

![terraform-apply](./images/terraform-apply-2.png)

Once the deployment is complete, Terraform will output the information about the provisioned resources, such as the IP addresses, DNS names, RDS endpoint, user and database name.

![outputs](./images/outputs.png)

Finally, let's check our infrastructure from the AWS Console:

![vpc](./images/vpc.png)

![instance](./images/instance.png)

![database](./images/databases.png)

![security](./images/security.png)

![loadbalancer](./images/loadbalancer.png)

The **WordPress** installation is available via the generated Load Balancer domain:

![wordpress](./images/wordpress.png)

**Important**: Let's note that we can link it to a custom domain along with an HTTP certificate by using AWS ACM service.

**Congratulations! You have successfully deployed your WordPress application on a 2-tier AWS architecture using Terraform**. You can now access your WordPress website and start customizing it to suit your needs.

The complete source code of the project is available on [GitHub]().

## Conclusion

To conclude, deploying WordPress on a 2-Tier AWS architecture with Terraform offers a reliable and scalable solution for hosting your website. This approach allows any team, regardless of their level of Cloud experience, to successfully set up and manage their WordPress application. By leveraging Terraform's infrastructure-as-code capabilities, the deployment process becomes streamlined and repeatable.

However, it is important to emphasize the significance of conducting thorough research and due diligence when selecting the appropriate AWS services and configurations for your specific requirements. Ensuring high availability, fault tolerance, and security should be prioritized during the deployment process.
