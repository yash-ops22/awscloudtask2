# Task Overview

1. Create Security group which allow the port 80.
2. Launch EC2 instance.
3. In this Ec2 instance use the existing key or provided 
  key and security group which we have created in step 1.
4. Launch one Volume using the EFS service and attach it 
  in your vpc, then mount that volume into /var/www/html
5. Developer have uploded the code into github repo also 
   the repo has some images.
6. Copy the github repo code into /var/www/html
7. Create S3 bucket, and copy/deploy the images from github 
  repo into the s3 bucket and change the permission to
  public readable.
8. Create a Cloudfront using s3 bucket(which contains images)
and use the Cloudfront URL to  update in code in /var/www/html

# Solution

 Before creating any Resources, we have to configure our AWS Profile:
 Here we have already configure it

           
           AWS Access Key ID [****************LELI]:
           AWS Secret Access Key [****************Av0P]:
           Default region name [ap-south-1]:
           Default output format [json]:


Then in our terraform code specifying the provider for creating
 the resources into the specifide provider...

         provider "aws" {
            region     = "ap-south-1"
            profile    = "Yashu"
             }


# Step 1:

First as mentioned we have to create a Security Group which
will allow port 80, also we have included port 22 to login into 
our instance.

    resource "aws_security_group" "sgcloud2" {

                name        = "sgcloud2"
           

                ingress {

                  from_port   = 80
                  to_port     = 80
                  protocol    = "tcp"
                  cidr_blocks = [ "0.0.0.0/0"]

                }

              
                ingress {

                  from_port   = 22
                  to_port     = 22
                  protocol    = "tcp"
                  cidr_blocks = [ "0.0.0.0/0"]

                }




                egress {

                  from_port   = 0
                  to_port     = 0
                  protocol    = "-1"
                  cidr_blocks = ["0.0.0.0/0"]
                }


                tags = {

                  Name = "sgcloud2"
                }
              }



<img src="sgroup.png">




# Step 2:

Launching our instance using the above created security
group and using the key we had created earlier. Also, installing 
the httpd server into it so that we can launch our site....


      resource "aws_instance" "aws-os" {
                        ami             =  "ami-0732b62d310b80e97"
                        instance_type   =  "t2.micro"
                        key_name        =  "cloudkey2"
                        subnet_id       =  "subnet-d26d68ba"
                        security_groups = ["${aws_security_group.sgcloud2.id}"]


                       connection {
                          type     = "ssh"
                          user     = "ec2-user"
                          private_key = file("C:/Users/win 10/Downloads/cloudkey2.pem")
                          host     = aws_instance.aws-os.public_ip
                        }

                        provisioner "remote-exec" {
                          inline = [
                            "sudo yum install amazon-efs-utils -y",
                            "sudo yum install httpd  php git -y",
                            "sudo systemctl restart httpd",
                            "sudo systemctl enable httpd",
                            "sudo setenforce 0",
                            "sudo yum -y install nfs-utils"
                          ]
                        }

                        tags = {
                          Name = "osaws"
                        }
                      }

     

# Step 3:

We have to create an volume or storage,  In our first
aws task we had used EBS, here we are using EFS.
Creating the EFS Volume......

     resource "aws_efs_file_system" "amazonefs" {
       creation_token = "my-cloudefs"

       tags = {
         Name = "amazonefs"
       }
     }

     resource "aws_efs_mount_target" "efsmount" {
        file_system_id = aws_efs_file_system.amazonefs.id
        security_groups = [aws_security_group.sgcloud2.id]
        subnet_id      =   "subnet-d26d68ba"

     }



<img src="efs.png">







We cannot use this storage directly, so Mounting this EFS to our 
instance that we have launched, mounting this EFS into directory
/var/www/html of the httpd server.
Also, cloning the gihthub repo into the /var/www/html folder
and downloading the files into it.......

     
    resource "null_resource" "mount"  {
                depends_on = [aws_efs_mount_target.efsmount]
                connection {
                type     = "ssh"
                user     = "ec2-user"
                private_key = file("C:/Users/win 10/Downloads/cloudkey2.pem")
                host     = aws_instance.aws-os.public_ip
              }
              provisioner "remote-exec" {
                  inline = [
                    "sudo mount -t nfs -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport ${aws_efs_file_system.amazonefs.id}.efs.ap-south-         1.amazonaws.com:/ /var/www/html",
                       "sudo rm -rf /var/www/html/*",
                       "sudo git clone https://github.com/yash-ops22/awscloud2.git /var/www/html/",
                       "sudo sed -i 's/url/${aws_cloudfront_distribution.amazoncloudfront.domain_name}/g' /var/www/html/index.html"
                        ]
                    }
                  }


# Step 4:


Creating the S3 bucket, so that we can upload files or
images into it.......

     resource "aws_s3_bucket" "buckets3yashucloud22" {
        bucket = "yashu22"
        acl    = "private"

        tags = {
          Name        = "buckets3"
        }
      }
       locals {
          s3_origin_id = "myS3Origin"
        }
  
  
  
   <img src="buckets3.png">     
  
  
  
  



Uploading the images into our S3 bucket and making it
publically accesible....
     
     resource "aws_s3_bucket_object" "objectawsbucket" {
          bucket = "${aws_s3_bucket.buckets3yashucloud22.id}"
        key    = "picture"
          source = "C:/Users/win 10/Downloads/aws1.jpg"
          acl    = "public-read"
        } 



# Step 5:



Creating the Cloudfront from the S3 bucket...
  
  
    resource "aws_cloudfront_distribution"  "amazoncloudfront" {
    origin {
    domain_name = "${aws_s3_bucket.buckets3yashucloud22.bucket_regional_domain_name}"
    origin_id   = "${local.s3_origin_id}"
   

     custom_origin_config {
            http_port = 80
            https_port = 443
            origin_protocol_policy = "match-viewer"
            origin_ssl_protocols = ["TLSv1", "TLSv1.1", "TLSv1.2"]
           }
        }
  

    enabled             = true
    is_ipv6_enabled     = true  

    default_cache_behavior {
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "${local.s3_origin_id}"
    

    forwarded_values {
      query_string = false

      cookies {
          forward = "none"
        }
      }

    viewer_protocol_policy = "allow-all"
    min_ttl                = 0
    default_ttl            = 3600
    max_ttl                = 86400
    }

    restrictions {
       geo_restriction {
          restriction_type = "none"
        }
    }

     viewer_certificate {
         cloudfront_default_certificate = true
        }
      } 
    
    
    
    
<img src="cdn.png">



    
We have to install the required  plugins for our terraform
using the command....

     terraform init
     
     
After Completing the code we can check or validate our code
using command....

     terrform  validate
      
      
 Now we can launch the infrastructure using the terraform 
 command.....
    
     terraform apply --auto-approve



<img src="apply.png">



Now when our infrastructure is launched we can update the code
with the cloudfront domain name.


<img src="up-cdn-link.png">


After saving these changes we can restart our httpd or apache server.
Using our instance's public IP we can see our web portal!!!!
This is what it looks like...........



<img src="output.png">













Thank You For Your Precious Time....................
