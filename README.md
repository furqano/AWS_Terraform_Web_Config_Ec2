# aws_terraform_web


##### It is a simple Terraform Infrastructure as code (IaC) to launch and setup webserver in AWS EC2 service ..


````
provider "aws" {
  region = "ap-south-1"
  profile = "fate"
}

// Creating Security Groups

resource "aws_security_group" "allow_tls" {
  name        = "allow_tls"
  ingress {
    description = "Security group for ssh"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    description = "Security group for http"
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
    Name = "allow_tls"
  }
}

// Creating AWS Instance

resource "aws_instance" "web" {
  ami           = "ami-0447a12f28fddb066"
  instance_type = "t2.micro"
  key_name = "fate_key"
  security_groups = [ "allow_tls" ]

  connection {
    type     = "ssh"
    user     = "ec2-user"
    private_key = file("E:/fate_key.pem")
    host     = aws_instance.web.public_ip
  }
  provisioner "remote-exec" {
    inline = [
      "sudo yum install httpd  php git -y",
      "sudo systemctl restart httpd",
      "sudo systemctl enable httpd",
    ]
  }

  tags = {
    Name = "lwos1"
  }

}






// Creating EBS Volume

resource "aws_ebs_volume" "ebs" {
  availability_zone = aws_instance.web.availability_zone
  size              = 1
  tags = {
    Name = "lwebs"
  }
}

// Attaching EBS Volume

resource "aws_volume_attachment" "ebs_att" {
  device_name = "/dev/sdh"
  volume_id   = "${aws_ebs_volume.ebs.id}"
  instance_id = "${aws_instance.web.id}"
  force_detach = true
}


// Creating s3 Bucket

resource "aws_s3_bucket" "buck" {
  bucket = "fateultimate1"
  acl    = "public-read"

  tags = {
    Name        = "Mybucket"
  }
}

// uploading file in s3

resource "aws_s3_bucket_object" "object" {
  depends_on = [
       aws_s3_bucket.buck ,
      ]

  bucket = "${aws_s3_bucket.buck.id}"
  key    = "fate.jpg"
  source = "E:/fate.jpg"
  etag = "${filemd5("E:/fate.jpg")}"
  acl = "public-read"
}

// Print IP

output "myos_ip" {
  value = aws_instance.web.public_ip
}


// Copy Format Attach

resource "null_resource" "null_att"  {

depends_on = [
    aws_volume_attachment.ebs_att,
  ]


  connection {
    type     = "ssh"
    user     = "ec2-user"
    private_key = file("E:/fate_key.pem")
    host     = aws_instance.web.public_ip
  }



provisioner "remote-exec" {
    inline = [
      "sudo mkfs.ext4  /dev/xvdh",
      "sudo mount  /dev/xvdh  /var/www/html",
      "sudo rm -rf /var/www/html/*",
      "sudo git clone https://github.com/FateDaeth/aws_terraform_web.git /var/www/html/"
    ]
  }
}




// Launching Browser 

resource "null_resource" "null_chrome"  {


depends_on = [
    aws_cloudfront_distribution.s3_distribution,
  ]

	provisioner "local-exec" {
	    command = "microsoftedge  ${aws_instance.web.public_ip}"
  	}
}

//Cloud Front

locals {
  s3_origin_id = "myS3Origin"
}

resource "aws_cloudfront_distribution" "s3_distribution" {
depends_on = [
    aws_s3_bucket.buck,
  ]
  origin {
    domain_name = "${aws_s3_bucket.buck.bucket_regional_domain_name}"
    origin_id   = "${local.s3_origin_id}"

  }

  enabled             = true
  is_ipv6_enabled     = true
  comment             = "Some comment"
  default_root_object = "index.php"


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

  # Cache behavior with precedence 0
  ordered_cache_behavior {
    path_pattern     = "/content/immutable/*"
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD", "OPTIONS"]
    target_origin_id = "${local.s3_origin_id}"

    forwarded_values {
      query_string = false
      headers      = ["Origin"]

      cookies {
        forward = "none"
      }
    }

    min_ttl                = 0
    default_ttl            = 86400
    max_ttl                = 31536000
    compress               = true
    viewer_protocol_policy = "redirect-to-https"
  }

  # Cache behavior with precedence 1
  ordered_cache_behavior {
    path_pattern     = "/content/*"
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "${local.s3_origin_id}"

    forwarded_values {
      query_string = false

      cookies {
        forward = "none"
      }
    }

    min_ttl                = 0
    default_ttl            = 3600
    max_ttl                = 86400
    compress               = true
    viewer_protocol_policy = "redirect-to-https"
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

}



````

