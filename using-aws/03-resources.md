# Provisioning Compute Resources

Kubernetes requires a set of machines to host the master and worker nodes.
- We will setup a VPC with subnet
- A key-pair to SSH in any instance
- we will also launch few ec2 instances for contoller and worker nodes
- also we will setup a LoadBalancer which will server as a endpoint for our controllers to communicate
- also we will setup some security groups for nodes and loadbalancer

### 1. VPC using CloudShell

We will setup a dedicated a VPC network, and use to setup to host the Kubernetes Cluster

we are using netmask of 0.0.255.255 so that we can associate ip ranges from 10.240.0.1 to 10.240.255.255

Create a `kubenetes-hard-way-VPC` custom VPC network:

    aws ec2 create-vpc --cidr-block 10.240.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=kubernetes-hard-way-VPC}]'

- Note `"VpcId": "vpc-0254c5ffb651b3e93"`

### 2. Subnet 

We will be using a CIDR range of 10.240.0.0/24 for our subnet inside the VPC network:

    aws ec2 create-subnet --vpc-id vpc-0254c5ffb651b3e93 --cidr-block 10.240.0.0/24 --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=kubernetes-hard-way-subnet}]'

- Note  `"SubnetId": "subnet-05e2515493647a469"`
#### A Public Subnet

    aws ec2 create-subnet --vpc-id vpc-0254c5ffb651b3e93 --cidr-block 10.240.254.0/24 --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=kubernetes-hard-way-public-subnet}]'
---
    "SubnetId": "subnet-0b08f9e8a68543cbc"


### 3. Key-Pair

    aws ec2 create-key-pair --key-name kubernetes-key | jq -r '.KeyMaterial' | sed 's/\\n/\n/g' > pvt-key.pem

    chmod 400 pvt-key.pem

This CMD will create a key-pair  and store on the CloudShell instance, we can copy it to our local machine also, using CloudShell console tools

### 4. Creating security Group for our instances inside the `kubernetes-hard-way-subnet`

    aws ec2 create-security-group --group-name kube-instance-SG --description "SG for instances inside kubernetes-hard-way-subnet" --vpc-id vpc-0254c5ffb651b3e93

- Note `"GroupId": "sg-0674fc466cb11999e"`

#### Adding security-group rules for SSH from external network

    aws ec2 authorize-security-group-ingress --group-id sg-0674fc466cb11999e --protocol tcp --port 22 --cidr 0.0.0.0/0

- Note `"SecurityGroupRuleId": "sgr-0e75cbe017eb7a950"`

#### Adding security-group rules for all traffic from same security-group

    aws ec2 authorize-security-group-ingress --group-id sg-0674fc466cb11999e --protocol -1 --port -1 --source-group sg-0674fc466cb11999e

- Note `"SecurityGroupRuleId": "sgr-0cf85adb482eb5d87"`

### 5. Launch 3 kube-controller instances

    for i in 0 1 2; do
        aws ec2 run-instances \
            --image-id ami-0a0409af1cb831414 \
            --count 1 \
            --instance-type t2.micro \
            --key-name kubernetes-key \
            --security-group-ids sg-0674fc466cb11999e \
            --subnet-id subnet-05e2515493647a469 \
            --private-ip-address 10.240.0.1${i} \
            --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=controller-${i}}]"
    done

- Note 
    1. "Value": "controller-0", "InstanceId": "i-05fe289c667f655aa", "PrivateIpAddress": "10.240.0.10"
    2. "Value": "controller-1", "InstanceId": "i-02a2d32243767e68f", "PrivateIpAddress": "10.240.0.11"
    3. "Value": "controller-2", "InstanceId": "i-0a950b5574707dc82", "PrivateIpAddress": "10.240.0.12"

### 6. Launch 3 worker nodes instances

    for i in 0 1 2; do
    aws ec2 run-instances \
        --image-id ami-0a0409af1cb831414 \
        --count 1 \
        --instance-type t2.micro \
        --key-name kubernetes-key \
        --security-group-ids sg-0674fc466cb11999e \
        --subnet-id subnet-05e2515493647a469 \
        --private-ip-address 10.240.0.2${i} \
        --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=worker-${i}}]"
    done

- Note:
    1. "Value": "worker-0", "InstanceId": "i-087fb1abb45618034", "PrivateIpAddress": "10.240.0.20"
    2. "Value": "worker-1", "InstanceId": "i-098b6c7f9464ebccc", "PrivateIpAddress": "10.240.0.21"
    3. "Value": "worker-2", "InstanceId": "i-0c82c61e4898467f6", "PrivateIpAddress": "10.240.0.22"

### 7. A jump-server to communicate with the other instances inside the subnet

    aws ec2 run-instances \
        --image-id ami-0a0409af1cb831414 \
        --count 1 \
        --instance-type t2.micro \
        --key-name kubernetes-key \
        --security-group-ids sg-0674fc466cb11999e \
        --subnet-id subnet-0b08f9e8a68543cbc \
        --private-ip-address 10.240.254.254 \
        --associate-public-ip-address \
        --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=kubernetes-jump-server}]"

- `"InstanceId": "i-0b53fe47e878446d6", "PublicIpAddress": "54.67.70.16"`


### 8. Create a Internet Gateway

    aws ec2 create-internet-gateway --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=kubernetes-IGW}]"

- `"InternetGatewayId": "igw-0449355ae87d4f124"`

### Attach Internet Gateway to VPC

    aws ec2 attach-internet-gateway --internet-gateway-id igw-0449355ae87d4f124 --vpc-id vpc-0254c5ffb651b3e93

### 9. Create a Public Route Table

    aws ec2 create-route-table --vpc-id vpc-0254c5ffb651b3e93  --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=public-route-table}]"

- `"RouteTableId": "rtb-0192bc8ea4a4e9588"`

### 10. Associate the Internet Gateway with the Route Table

    aws ec2 associate-route-table --route-table-id rtb-0192bc8ea4a4e9588 --subnet-id subnet-05e2515493647a469 --gateway-id igw-0449355ae87d4f124

- `"AssociationId": "rtbassoc-07322428ebe70ff1f"`

### 11. Create a route in route-table to route the traffic from subnet to internet using IGW

    aws ec2 create-route --route-table-id rtb-0192bc8ea4a4e9588 --destination-cidr-block 0.0.0.0/0 --gateway-id igw-0449355ae87d4f124

- `  "Return": true`


### 12. Allocate a Elastic IP for Network Load Balancer

    aws ec2 allocate-address
---

OUTPUT

    {
        "PublicIp": "54.177.96.206",
        "AllocationId": "eipalloc-0b525b9474f63b6c7",
        "PublicIpv4Pool": "amazon",
        "NetworkBorderGroup": "us-west-1",
        "Domain": "vpc"
    }

### 13. Internet Facing Network Load Balancer

    aws elbv2 create-load-balancer \
    --name network-load-balancer-kubernetes \
    --type network \
    --subnet-mappings SubnetId=subnet-05e2515493647a469,AllocationId=eipalloc-0b525b9474f63b6c7

- `"DNSName": "network-load-balancer-kubernetes-13cdd4a28d9787a0.elb.us-west-1.amazonaws.com"`
- `"LoadBalancerAddresses": [
                        {
                            "IpAddress": "54.177.96.206",
                            "AllocationId": "eipalloc-0b525b9474f63b6c7"
                        }
                    ]`



### 14. Creating NAT-Gateway and attaching it with VPC and subnet

#### Allocate a ElasticIPAddress

    aws ec2 allocate-address
---
    {
    "PublicIp": "54.193.170.209",
    "AllocationId": "eipalloc-05e5b8dfce8a628cd",
    "PublicIpv4Pool": "amazon",
    "NetworkBorderGroup": "us-west-1",
    "Domain": "vpc"
    }

#### Create a NAT Gateway in pubic-subnet

    aws ec2 create-nat-gateway \
    --allocation-id eipalloc-05e5b8dfce8a628cd \
    --subnet-id subnet-0b08f9e8a68543cbc \
    --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=kubernetes-NATGateway}]'
---
    "NatGatewayId": "nat-0b978517ffab1f701"

#### Create a Route Table for Private Subnet

    aws ec2 create-route-table \
    --vpc-id vpc-0254c5ffb651b3e93 \
    --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=-kubernetes-PrivateRouteTable}]'
---
    "RouteTableId": "rtb-0764901928e3a0d6b"

#### Associate the Route Table with the Subnet:

    aws ec2 associate-route-table \
    --subnet-id subnet-05e2515493647a469 \
    --route-table-id rtb-0764901928e3a0d6b
---
    "AssociationId": "rtbassoc-024ea166849eb24f2"

#### Create a Route in the Private Route Table to the NAT Gateway

    aws ec2 create-route \
    --route-table-id rtb-0764901928e3a0d6b \
    --destination-cidr-block 0.0.0.0/0 \
    --nat-gateway-id nat-0b978517ffab1f701





