# Terraform

resource "aws_vpc" "MyVPC" {
  cidr_block = var.MyVPC_cidr
  tags = {
    "Name" = "my_vpc"
  }
}


resource "aws_subnet" "Subnet1" {
  vpc_id                  = aws_vpc.MyVPC.id
  cidr_block              = var.Subnet1_cidr
  availability_zone       = var.Subnet1_AZ
  map_public_ip_on_launch = true
  tags = {
    "Name" = "my_subnet1"
  }
}


resource "aws_subnet" "Subnet2" {
  vpc_id                  = aws_vpc.MyVPC.id
  cidr_block              = var.Subnet2_cidr
  availability_zone       = var.Subnet2_AZ
  map_public_ip_on_launch = true
  tags = {
    "Name" = "my_subnet2"
  }
}


resource "aws_internet_gateway" "myIGW" {
  vpc_id = aws_vpc.MyVPC.id
  tags = {
    Name = "my_igw"
  }
}
# "keyname" or keyname both are valid in Terraform. HCL allows keys in maps to be written without quotes if the key does not contain special characters (like spaces or hyphens) and follows a simple naming convention.
# But recommended best practice is "keyname"


resource "aws_route_table" "myRT" {
  vpc_id = aws_vpc.MyVPC.id


  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.myIGW.id
  }
  tags = {
    Name = "my_rt"
  }
}


resource "aws_route_table_association" "rtA1" {
  route_table_id = aws_route_table.myRT.id
  subnet_id      = aws_subnet.Subnet1.id
}


resource "aws_route_table_association" "rtA2" {
  route_table_id = aws_route_table.myRT.id
  subnet_id      = aws_subnet.Subnet2.id
}


resource "aws_security_group" "mySG" {
  name        = "my_SG"
  description = "Allow SSH and HTTP inbound traffic and all outbound traffic"
  vpc_id      = aws_vpc.MyVPC.id
}


resource "aws_vpc_security_group_ingress_rule" "allowSSH" {
  security_group_id = aws_security_group.mySG.id


  ip_protocol = "tcp"
  from_port   = 22
  to_port     = 22
  cidr_ipv4   = "0.0.0.0/0"
}


resource "aws_vpc_security_group_ingress_rule" "allowHTTP" {
  security_group_id = aws_security_group.mySG.id


  ip_protocol = "tcp"
  from_port   = 80
  to_port     = 80
  cidr_ipv4   = "0.0.0.0/0"
}


resource "aws_vpc_security_group_egress_rule" "allowEverything" {
  security_group_id = aws_security_group.mySG.id


  ip_protocol = "-1"
  cidr_ipv4   = "0.0.0.0/0"
}


#localVAR is a local variable for this terraform file, to reference it later in the code
resource "aws_s3_bucket" "localVAR" {
  # Can't use both bucket and bucket_prefix (aws will append random suffix in bucket name while creating) at the same time.    
  #bucket_prefix = "tf-project1-AC"
  bucket = "tf-project1-ac-s3bucket"
}


resource "aws_s3_bucket_ownership_controls" "localVAR1" {
  bucket = aws_s3_bucket.localVAR.id


  rule {
    object_ownership = "BucketOwnerPreferred"
  }
}


resource "aws_s3_bucket_public_access_block" "localVAR1" {
  bucket = aws_s3_bucket.localVAR.id


  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}


resource "aws_s3_bucket_acl" "localVAR1" {
  depends_on = [
    aws_s3_bucket_ownership_controls.localVAR1,
    aws_s3_bucket_public_access_block.localVAR1
  ]


  bucket = aws_s3_bucket.localVAR.id
  acl    = "public-read"
}




resource "aws_instance" "instance1" {
  ami                    = "ami-05576a079321f21f8"
  instance_type          = "t2.micro"
  subnet_id              = aws_subnet.Subnet1.id
  vpc_security_group_ids = [aws_security_group.mySG.id]
  user_data              = base64encode(file("userdata1.sh"))
  tags = {
    "Name" = "Web-Server-1"
    "Env"  = "Dev"
  }
}


resource "aws_instance" "instance2" {
  ami                    = "ami-05576a079321f21f8"
  instance_type          = "t2.micro"
  subnet_id              = aws_subnet.Subnet2.id
  vpc_security_group_ids = [aws_security_group.mySG.id]
  user_data              = base64encode(file("userdata2.sh"))
  tags = {
    "Name" = "Web-Server-2"
    "Env"  = "Dev"
  }
}




resource "aws_lb" "myLB" {
  name               = "my-lb"
  internal           = false # external/publib LB
  load_balancer_type = "application"
  security_groups    = [aws_security_group.mySG.id]
  subnets            = [aws_subnet.Subnet1.id, aws_subnet.Subnet2.id]


  #enable_deletion_protection = true
  /*
  access_logs {
    bucket  = aws_s3_bucket.lb_logs.id
    prefix  = "test-lb"
    enabled = false
  }
*/
}


resource "aws_lb_target_group" "myTG" {
  name     = "my-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.MyVPC.id
  health_check {
    path     = "/"
    protocol = "HTTP"
  }
}


resource "aws_lb_target_group_attachment" "tgattach1" {
  target_group_arn = aws_lb_target_group.myTG.arn
  target_id        = aws_instance.instance1.id
  port             = 80
}


resource "aws_lb_target_group_attachment" "tgattach2" {
  target_group_arn = aws_lb_target_group.myTG.arn
  target_id        = aws_instance.instance2.id
  port             = 80
}


resource "aws_lb_listener" "listener" {
  load_balancer_arn = aws_lb.myLB.arn
  port              = 80
  protocol          = "HTTP"


  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.myTG.arn
  }
}


output "loadbalancerdns" {
  value = aws_lb.myLB.dns_name # print the value on the terminal
}





main.tf




—--------------------------------------------------------------------------------------------------------------------------
variable.tf



variable "MyVPC_cidr" {
  default = "10.0.0.0/16"
}


variable "Subnet1_cidr" {
  default = "10.0.0.0/24"
}


variable "Subnet2_cidr" {
  default = "10.0.1.0/24"
}


variable "Subnet1_AZ" {
  default = "us-east-1a"
  #default = "ap-south-1a"
}


variable "Subnet2_AZ" {
  default = "us-east-1b"
  #default = "ap-south-1b"
}



—----------------------------------------------------------------------------------------------------------------------------
provider.tf

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}


# Configure the AWS Provider
provider "aws" {
  region = "us-east-1"
  #region = "ap-south-1"
}

