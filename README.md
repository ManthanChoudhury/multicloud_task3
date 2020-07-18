# multicloud_task3

## Create a VPC on AWS Using Terraform

![01](https://user-images.githubusercontent.com/45136716/87841044-3a8fcc80-c8c0-11ea-9995-150e0456cfd9.png)


In AWS that service which provides NAAS is known as VPC(Virtual Private Cloud).



# Task discription :-

1) Write a Infrastructure as code using terraform, which automatically create a VPC.



2) In that VPC we have to create 2 subnets:

  a) public subnet [ Accessible for Public World! ] 

  b) private subnet [ Restricted for Public World! ]



3) Create a public facing internet gateway for connect our VPC/Network to the internet world and attach this gateway to our VPC.



4) Create a routing table for Internet gateway so that instance can connect to outside world, update and associate it with public subnet.



5) Launch an ec2 instance which has Wordpress setup already having the security group allowing port 80 so that our client can connect to our wordpress site.

Also attach the key to instance for further login into it.



6) Launch an ec2 instance which has MYSQL setup already with security group allowing port 3306 in private subnet so that our wordpress vm can connect with the same.

Also attach the key with the same.

```
provider "aws" {
  region     = "ap-south-1"
  profile    = "soul"
}
```

### We have to initialize it so that it can download AWS provider plugin.

![2](https://user-images.githubusercontent.com/45136716/87840909-a7ef2d80-c8bf-11ea-9380-d4411490fd35.png)

 
### After that we have to create VPC. For this we require CIDR block to specify range of IPv4 addresses for the VPC and a name tag for unique identification.

```
resource "aws_vpc" "main" {
  cidr_block       = "192.168.0.0/16"
  instance_tenancy = "default"
  enable_dns_hostnames = "true"
  tags = {
    Name = "fvpc"
  }
}
```

### On applying this we get:

![3](https://user-images.githubusercontent.com/45136716/87840912-a9b8f100-c8bf-11ea-9688-275edd507189.jpeg)

### Creating Private Subnet:

```
resource "aws_subnet" "privateSn" {
  vpc_id     = "${aws_vpc.main.id}"
  cidr_block = "192.168.0.0/24"
  availability_zone = "ap-south-1a"

  tags = {
    Name = "fvcsubnet-1"
  }
}
```

### Creating Public Subnet:

```
resource "aws_subnet" "publicSn" {
  vpc_id     = "${aws_vpc.main.id}"
  cidr_block = "192.168.1.0/24"
  availability_zone = "ap-south-1b"
  map_public_ip_on_launch = true

  tags = {
    Name = "fvpcsubnet-2"
  }
}
```

![4](https://user-images.githubusercontent.com/45136716/87840917-aaea1e00-c8bf-11ea-96dd-9a254860cab5.jpeg)


### Create a public facing internet gateway for connect our VPC/Network to the internet world and attach this gateway to our VPC.


```
resource "aws_internet_gateway" "gw" {
  vpc_id = "${aws_vpc.main.id}"

  tags = {
    Name = "vpca"
  }
}

```

Here we have to give our VPC ID and name whatever er want to give to internet gateway.

![5](https://user-images.githubusercontent.com/45136716/87840918-ad4c7800-c8bf-11ea-9a50-08fc23c5da54.jpeg)

### Create a routing table for Internet gateway so that instance can connect to outside world, update and associate it with public subnet.

```
resource "aws_route_table" "vpcRouteTable" {
  vpc_id = "${aws_vpc.main.id}"

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = "${aws_internet_gateway.gw.id}"
  }


  tags = {
    Name = "vpcroute"
  }
}
```

### Associating Public subnet to this route table:

```
resource "aws_route_table_association" "associate" {
  subnet_id      = "${aws_subnet.publicSn.id}"
  route_table_id = "${aws_route_table.vpcRouteTable.id}"
}
```

### we have to create routing table to add internet gateway details.

Launch an ec2 instance which has WordPress setup already having the security group allowing port 80 so that our client can connect to our WordPress site.Also attach the key to instance for further login into it.

### Creating Security Group: This security group only allow ping,ssh and httpd.

```
resource "aws_security_group" "wpsg" {
  name        = "wp"
  description = "Allow TLS inbound traffic"
  vpc_id      = "${aws_vpc.main.id}"

  ingress {
    description = "ICMP"
    from_port   = 8
    to_port     = 0
    protocol    = "icmp"
    cidr_blocks = ["0.0.0.0/0"]
  }

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
    Name = "wpsg"
  }
}

```

Now we can create our instance. For creating any instance we need AMI,instance type,availability zone which we have mentioned earlier in subnet and key - here I used pre-existing key. For WordPress AMI, I used WordPress Base Version which requires subscription for that.


```
resource "aws_instance" "wpinstance" {
  ami           = "ami-7e257211"
  instance_type = "t2.micro"
  key_name      = "clusterKey"
  subnet_id =  aws_subnet.publicSn.id
  vpc_security_group_ids = [ aws_security_group.wpsg.id ]
  tags = {
    Name = "wpos"
  }
}

```

Launch an ec2 instance which has MYSQL setup already with security group allowing port 3306 in private subnet so that our WordPress instance can connect with the same.Also attach the key with the same.

### Creating MySQL Security group:

Creating MySQL Security group:

```
resource "aws_security_group" "mysqlsg" {
  name        = "basic"
  description = "Allow TLS inbound traffic"
  vpc_id      = "${aws_vpc.main.id}"

  ingress {
    description = "mysql"
    from_port   = 3306
    to_port     = 3306
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

```

### Creating Instance:

```
resource "aws_instance" "mysqlinstance" {
  ami           = "ami-08706cb5f68222d09"
  instance_type = "t2.micro"
  key_name      = "clusterKey"
  subnet_id =  aws_subnet.privateSn.id
  vpc_security_group_ids = [ aws_security_group.mysqlsg.id ]
  tags = {
    Name = "mysqlos"
  }
}

```

![6](https://user-images.githubusercontent.com/45136716/87840920-afaed200-c8bf-11ea-8e3b-7fbb12edae70.jpeg)

### As In this Instance, we don't have Public DNS and Public IP which proves that is an instance belongs to private subnet.

#### Applying complete code gives:

![7](https://user-images.githubusercontent.com/45136716/87840922-b1789580-c8bf-11ea-8c0e-04550bbb6ed0.png)

By using Public IP or Public DNS we can access our WordPress site.

![8](https://user-images.githubusercontent.com/45136716/87840924-b3425900-c8bf-11ea-9d36-b3fbd4385783.jpg)








