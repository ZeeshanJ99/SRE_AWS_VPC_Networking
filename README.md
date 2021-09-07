# AWS Virtual Private Cloud (VPC) CIDR Block for VPC
## Internet Gateway
## Subnets
### Route Tables
#### Network Access Control list (NACLs)
##### Security Groups

![image](https://user-images.githubusercontent.com/88186084/132206837-00a308d2-ee2d-4daf-ad77-8aa525a09b4a.png)

## What is a VPC?
- AWS isolated virtual network - It allows us to control virtual network environment inclusing a selcetion of your own IPs address range, we can create multiple subnets within one VPC with specific network configuration. We can use both IPv4 and IPv6 for most resources, it provides security for your services or instances

## What is a an Internet Gateway?
- An Internet gateway - can transfer communications between an enterprise network (e.g. your network) and the internet - it allows internet access into the VPC

## What is a Subnet?
- A Subnet is a segmented piece of a larger network - the goal of a subnet is to split a large network into a group of smaller , interconnected networks to help minimise traffic - or navigate traffic securely

## Route Table RT
- RT contains a set of rules, called routes, that are used to determine where the network traffic from your subnet or gateway is directed

## Network Access Control List (NACL)
-  NACLs are stateless - we have to explcitly allow inbound or outbound rules - they are an added layer of security at subnet level.

----------------------------------------------------------

## AWS Regions
A physical location around the world where we cluster our data centres. The region where the server is located that we are connecting to.

Each region provides its own facilities. Not every single region has all the services available. Need to make sure you choose the right region and Availability zone for the services that you would want to use. 

## AWS availability zone
An AZ is one or more discrete data centres with reduntant power, networking and connectivity in an AWS region. Each region has at least 2 or more availability zones. e.g. Region is Europe - availability zones would be England, Ireland, Stockholm etc.

To ensure reliability we deploy in multiple Availability Zones (multi AZ). So if theres a natural disaster in Ireland and the servers are compromised, we have a backup on London.

----------------------------------------

### Diagram of what we will create

![image](https://user-images.githubusercontent.com/88186084/132349397-60f34584-eca7-4a02-8df8-fae1be39d4d8.png)

------------------------------------------------------------------------

## Steps to create a custom VPC with subnets

### Step 1: Create a VPC with IPv CDIR block 
- `10.109.0.0/16`

-------------------------------------------------------------

### Step 2: Create internet gateway
- 2.1: Attach IG to your VPC

Add a Name tag then click Create Internet Gateway
Select action and Attach VPC. Attach the VPC you created previously.

----------------------------------------------------------

### Step 3: Create route table
- 3.1: Edit route and insert your IG in `target`

After creating the RT, select it in the list, go to the Routes tab and Edit routes

Add route with destination as `0.0.0.0/0` and target your IG (created previously)

Click the Target box and select Internet Gateway from the dropdown menu. After this, your IG should be shown.

Select your RT in the list, go to the `Subnet associations` and `Edit subnet associations`. Select your subnets and `Save associations`

----------------------------------------------------------

### Step 4: Create public subnet
- `10.109.9.0/24`
- 4.1 associate public subnet with our RT

- Create network ACL

Name --> `SRE_zeeshan_acl_public`

Select your VPC from the dropdown menu.

We will need to create a public and a private NACL for both our subnets

Select your NACL from the list, go to the `Inbound rules tab` and edit your rules. Do the same for `Outbound rules`.

-----------------------------------------------

### Step 5: Create public NACLs
- - set inbound and outbound rules for this
- star = * in all the following rules 

### Public subnet rules

#### Inbound Rules Public

Rule | Source IP | Protocol | Port      | Allow/Deny |
-----|-----------|----------|-----------|------------| 
100  | 0.0.0.0/0 | HTTP     | 80        | Allow      |
110  | My IP     | SSH      | 22        | Allow      |
120  | 0.0.0.0/0 | TCP      | 1024-65535| Allow      |
star | 0.0.0.0/0 | ALL      | ALL       | Deny       |

- Replace My IP with your IP

#### Outbound Rules Public 

Rule | Source IP        | Protocol  | Port      | Allow/Deny |
-----|------------------|-----------|-----------|------------| 
100  | 0.0.0.0/0        | HTTP      | 80        | Allow      |
110  | 10.109.10.0/24   | TCP       | 27017     | Allow      |
120  | 0.0.0.0/0        | TCP       | 1024-65535| Allow      |
star | 0.0.0.0/0        | ALL       | ALL       | Deny       |

`10.109.9.0/24` is the address of the private subnet. This allows the app to send a request for data from the db machine

-------------------------------------------------

### Private subnet rules

#### Inbound Rules Private

Rule | Source IP     | Protocol | Port      | Allow/Deny |
-----|---------------|----------|-----------|------------| 
100  | 10.109.9.0/24 | TCP      | 27017     | Allow      |
110  | My IP         | SSH      | 22        | Allow      |
120  | 0.0.0.0/0     | TCP      | 1024-65535| Allow      |
star | 0.0.0.0/0     | ALL      | ALL       | Deny       |

Again, the public subnet address is required here. This is to allow the app machine to request data from the db machine.

#### Outbound Rules Private

Rule | Source IP        | Protocol  | Port      | Allow/Deny |
-----|------------------|-----------|-----------|------------| 
100  | 0.0.0.0/0        | HTTP      | 80        | Allow      |
120  | 0.0.0.0/0        | TCP       | 1024-65535| Allow      |
star | 0.0.0.0/0        | ALL       | ALL       | Deny       |



-----------------------

### Create EC2 instances
When the network is set up, which allows communication between the subnets we can now initialise the app and db machines using the AMI. If you cant remember how to do so follow this link https://github.com/ZeeshanJ99/two_tier_architecture

Remember when launching each machine, select the new VPC and subnets that you have created

-----------------------------------------------------

### Step 6: Creating Security groups for our app and db

#### App security group rules

![App SG rules](https://user-images.githubusercontent.com/88186084/132342466-c880a61c-2d34-4131-9a48-f135199f1e9d.jpg)

Use your IP on the blacked out section with the /32 CIDR block at the end

#### DB security group rules

![DB SG rules](https://user-images.githubusercontent.com/88186084/132342858-0347f7c7-b1f1-4771-a353-df35023f82b7.jpg)

Again use your IP on the blacked out section with the /32 CIDR block at the end

----------------------------------------------------

### DB machine with private IP

In order to create a more secure db machine, when launching the db machine, DISABLE the Auto-assign Public IP option. This will prevent a public IP address being assigned to the machine.

- In order for the app machine to connect to the database, the DB_HOST environment    variable must be changed and now use the db machine's private IPv4 address.
- We can also write the variable to the /etc/environement file as well, to ensure that the variable is persistent when sshing in

Make sure that the new db IP is replaced with private IPv4 address of the db machine

[DB-IP} will be the new private IPv4 address you get

`echo "DB_HOST=[DB-IP]:27017/posts" | sudo tee -a /etc/environment`

`export DB_HOST=[DB-IP]:27017/posts`

---------------------------------------------------
Paste this in a new Readme in the same repo called S3.md
instructions to do so are in the markdown link shah rukh sent

## S3

- Ensure that your machine is connected to the internet using `ping www.bbc.com`
- `sudo apt-get update`
- `sudo apt-get upgrade`
- `sudo apt-get install python`
- `sudo apt-get install python3-pip`
- `alias python=python3`
- `python --version` should display python version as above 3
- `python3 -m pip install awscli`

### aws configure
`aws configure`
- now add your access key ID
- add Secret access key 
- region name = `eu-west-1`
- Default output format = `json`
- `aws s3 ls` - this should show a list of buckets


#### Creating a bucket
`aws s3 mb s3://srezeeshan`

#### Creating a file
`sudo nano README.md` add some text in the file and save

#### Pushing the file to the bucket
`aws s3 cp README.md s3://srezeeshan/`

