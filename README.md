<div align="center"> <h1> AWS-UseCase </h1> </div>
<div align="center"> <h3> Task: Have to create/launch Application using Terraform </h3> </div>

1. Create the key and security group which allow the port 80.
2. Launch EC2 instance.
3. In this Ec2 instance use the key and security group which we have created in step 1.
4. Launch one Volume (EBS) and mount that volume into /var/www/html
5. Developer have uploded the code into github repo also the repo has some images.
6. Copy the github repo code into /var/www/html
7. Create S3 bucket, and copy/deploy the images from github repo into the s3 bucket and change the permission to public readable.

8 Create a Cloudfront using s3 bucket(which contains images) and use the Cloudfront URL to  update in code in /var/www/html

**Step-1**
* First we have to connect to the aws using terraform 
* Here The code
```
provider "aws" {
  region = "ap-south-1"
  profile = "sumit"
```
* Just Understand The Code - we are using the Profile "sumit" Here. profile means . we are storing the **AWS Access Key ID**
,**AWS Secret Access Key**,**Default region name**,**Default output format**.
* For Creating the Profile Use the command 
```
aws configure --profile < Profile-Name>
```
![job2](https://github.com/Sumit-Rasal/aws-usecase-1/blob/master/Screenshot/Screenshot%20from%202020-06-14%2008-35-11.png)
**Step-2**
* create the Key Pair For The EC2 instance
```
resource "tls_private_key" "webserver_private_key" {
 algorithm = "RSA"
 rsa_bits = 4096
}
resource "local_file" "private_key" {
 content = tls_private_key.webserver_private_key.private_key_pem
 filename = "webserver_key.pem"
 file_permission = 0400
}

resource "aws_key_pair" "webserver_key" {
 key_name = "webserver"
 public_key = tls_private_key.webserver_private_key.public_key_openssh
}
```
**Step-3**
* security group which allow the port 80
```
resource "aws_security_group" "allow_http_ssh" {
  name        = "allow_http"
  description = "Allow http inbound traffic"
ingress {
    description = "http"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]

  }
ingress {
    description = "ssh"
    from_port   = 22
    to_port     = 22
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
    Name = "allow_http_ssh"
  }
}

```
**Step-4**
* Launch The EC2 instance 
* In this Ec2 instance use the key and security group which we have created in step 2,3.
```
resource "aws_instance" "webserver" {
  ami           = "ami-0447a12f28fddb066"
  instance_type = "t2.micro"
  key_name  = aws_key_pair.webserver_key.key_name
  security_groups=[aws_security_group.allow_http_ssh.name]
tags = {
    Name = "webserver_task1"
  }
  connection {
        type    = "ssh"
        user    = "ec2-user"
        host    = aws_instance.webserver.public_ip
        port    = 22
        private_key = tls_private_key.webserver_private_key.private_key_pem
    }
  provisioner "remote-exec" {
        inline = [
        "sudo yum install httpd php git -y",
        "sudo systemctl start httpd",
        "sudo systemctl enable httpd",
        ]
    }
}

```
**Step-5**
* Creat The ebs volume.
```
resource "aws_ebs_volume" "my_volume" {
    availability_zone = aws_instance.webserver.availability_zone
    size              = 1
    tags = {
        Name = "webserver-pd"
    }
}
```
**Step-6**
* attach the ebc volume.
```
resource "aws_volume_attachment" "ebs_attachment" {
    device_name = "/dev/xvdf"
    volume_id   =  aws_ebs_volume.my_volume.id
    instance_id = aws_instance.webserver.id
    force_detach =true
   depends_on=[ aws_ebs_volume.my_volume,aws_ebs_volume.my_volume]
}
```
**Step-7**
* Creating the S3 bucket
```
resource "aws_s3_bucket" "task1_s3bucket" {
  bucket = "website-images-res"
  acl    = "public-read"
  tags = {
    Name        = "My bucket"
    Environment = "Dev"
  }
}
```
**Step-8**
* Downloading The Github repository Data and upload in the s3
```
resource "null_resource" "images_repo" {
  provisioner "local-exec" {
    command = "git clone https://github.com/Sumit-Rasal/aws-usecase-1.git myImage"
  }
  provisioner "local-exec"{
  when        =   destroy
        command     =   "rm -rf my_images"
    }
}
resource "aws_s3_bucket_object" "myImage-1" {
  bucket = aws_s3_bucket.task1_s3bucket.bucket
  key    = "myImage"
  source = "myImage/myImage"
  acl="public-read"
   depends_on = [aws_s3_bucket.task1_s3bucket,null_resource.images_repo]
}
```
**Step-9**
* Create the cloud Front for the s3.
```
resource "aws_cloudfront_distribution" "s3_distribution" {
  origin {
    domain_name = aws_s3_bucket.task1_s3bucket.bucket_regional_domain_name
    origin_id   = aws_s3_bucket.task1_s3bucket.id

     custom_origin_config {
            http_port = 80
            https_port = 443
            origin_protocol_policy = "match-viewer"
            origin_ssl_protocols = ["TLSv1", "TLSv1.1", "TLSv1.2"]
        }
  }
  enabled             = true
  is_ipv6_enabled     = true
  comment             = "Some comment"
default_cache_behavior {
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = aws_s3_bucket.task1_s3bucket.id
forwarded_values {
      query_string = false
cookies {
        forward = "none"
      }
    }
   viewer_protocol_policy = "allow-all"
  }
 price_class = "PriceClass_200"
restrictions {
        geo_restriction {
        restriction_type = "none"
        }
    }
 viewer_certificate {
    cloudfront_default_certificate = true
  }
 depends_on = [aws_s3_bucket.task1_s3bucket]
}
```
**Step-10**
* attached The ebc volume to instance 
```
resource "null_resource" "nullremote"  {
depends_on = [  aws_volume_attachment.ebs_attachment,aws_cloudfront_distribution.s3_distribution ]
    connection {
        type    = "ssh"
        user    = "ec2-user"
        host    = aws_instance.webserver.public_ip
        port    = 22
        private_key = tls_private_key.webserver_private_key.private_key_pem
    }
   provisioner "remote-exec" {
        inline  = [
     "sudo mkfs.ext4 /dev/xvdf",
     "sudo mount /dev/xvdf /var/www/html",
     "sudo rm -rf /var/www/html/*",
     "sudo git clone https://github.com/Sumit-Rasal/aws-usecase-1.git /var/www/html/",
     "sudo su << EOF",
            "echo \"${aws_cloudfront_distribution.s3_distribution.domain_name}\" >> /var/www/html/path.txt",
            "EOF",
     "sudo systemctl restart httpd"
 ]
    }
}
```
**Step-11**
* displaying the pulic ip of instance
```
output "IP"{
 value=aws_instance.webserver.public_ip
}
```
![job-1](https://github.com/Sumit-Rasal/aws-usecase-1/blob/master/Screenshot/Screenshot%20from%202020-06-13%2022-42-57.png)


















