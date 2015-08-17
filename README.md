# Connecting AWS VPC to Azure VNet via Resource Manager

## Why do this?

Often times companies with sizeable deployments opt to run their infrastructure on multiple platforms or environments. This can be their own on-premesis datacenter, or multiple cloud platforms. Azure Virtual Gateways allow for any service deployment on-premesis as well deployments running on other clouds such as AWS to interact, providing redundancy and failover necessary for very high reliability.

## About This Template

This template will create all the Azure infrastructure necessary to connect to an existing [AWS VPC](https://aws.amazon.com/vpc/), without the need of powershell. This guide uses [StrongSwan](https://www.strongswan.org/) for the VPN IPSec software and is largley based a few articles:

- [Connecting to a Windows Azure Vitual Network via a Linux Based Software VPN Device](http://azure.microsoft.com/blog/2014/05/22/connecting-to-a-windows-azure-virtual-network-via-a-linux-based-software-vpn-device/)
- [Connecting Clouds – Creating a site-to-site Network with Amazon Web Services and Windows Azure](http://michaelwasham.com/2013/09/03/connecting-clouds-site-to-site-aws-azure/)
- [Create a virtual network with a site-to-site VPN connection using Azure Resource Manager and PowerShell](https://azure.microsoft.com/en-us/documentation/articles/vpn-gateway-create-site-to-site-rm-powershell/)

Although unlike the guides above, this guide specifically details connecting to an Amazon AWS VPC using Azure Resource Manager.

## Getting Started

### Setting up your VPC Part I

If you already have a VPC setup, go ahead an skip this section.

Using the AWS console, create a new VPC, you can give a big address space such as `10.0.0.0/16`: 

![](http://az731655.vo.msecnd.net/content/azure-vnet-aws-vpc/ss1.png)

Create a new subnet and associate it with the given vpc. Any subnet size will do as long as its within a subset of the ip address space defined in the previous step. I'm going with `10.0.0.0/24`.

![](http://az731655.vo.msecnd.net/content/azure-vnet-aws-vpc/ss2.png)

Then launch an EC2 ubuntu instance with connected to the VPC and subnet created in the step above:

![](http://az731655.vo.msecnd.net/content/azure-vnet-aws-vpc/ss3.png)

Disable source/destination checking on the instance:

![](http://az731655.vo.msecnd.net/content/azure-vnet-aws-vpc/ss4.png)

Now, in your VPC, allocate a new Elastic IP (public ip) address and associate it with the instance:


![](http://az731655.vo.msecnd.net/content/azure-vnet-aws-vpc/ss5.png)
![](http://az731655.vo.msecnd.net/content/azure-vnet-aws-vpc/ss6.png)


If you haven't already, its a good idea to change or at least add the hostname of your EC2 instance to it's `/etc/hosts` file (pointing to `127.0.0.1`) as StrongSwan's ipsec command seems to look for your current instance's hostname on the network:

```
vim /etc/hostname
vim /etc/hosts
sudo service hostname restart
```

### Deploying the Azure VNet & Gateway

Deploying the Azure VNet and connection is made easy with the included resource manager template. This template includes a VirutalNetworkGateway, a LocalNetworkGateway and a VirtualNetworkGatewayConnection, as well as an instance to connect to an existing AWS VPC.

First, fill in the required parameters defined by [azuredeploy-parameters.json](./azuredeploy-parameters.json). This includes `vpcGatewayIpAddress`, `vpcAddressPrefix`, `sharedKey` and `adminPassword`. There's also lots of other parameters you can change which you can find within the [template file](./azuredeploy.json). The most important parmaters are `vpcGatewayIpAddress` which is the public IP of the EC2 isntance created above and `vpcAddressPrefix` which is the CIDR block of the VPC subet.

To deploy the parameters using the [azure cli](https://npmjs.com/azure-cli) create a new resource group then deploy the template to it.

```
azure group create aws2azure westus
azure group deployment create aws2azure --template-file azuredeploy.json --parameters-file azuredeploy-parameters.json
```

As of writing this a resource manger deployment with a virtual network gateway allocation may take up to 30 minutes to provision. Hopefully that gets fixed soon!

Afterwards you can verify within the Resource Explorer blade on portal.azure.com that the resource group you deployed to contains a VirtualNetworkGatewayConnection to the AWS VPC.


### Setting up your VPC Part II

We needed some details from the Azure deployment in order to complete configuring the VPC on the AWS side. First we need to add a route the VPC routing table and associate it with our subnet. Add a route to the route table associated with the VPC, with the Azure VNet CIDR as the destination, and the EC2 instance as the target:

![](http://az731655.vo.msecnd.net/content/azure-vnet-aws-vpc/ss7.png)

Next, go to your security group associated with the ec2 instance and add 2 inbound custom UDP rules for ports 500 and 4500. The source for both should be a fixed single address CIDR block pointing your Azure VirtualNetworkGateway public IP formatted as: `[Azure VNet Public IP]/32`.

![](http://az731655.vo.msecnd.net/content/azure-vnet-aws-vpc/ss8.png)

### Setting up Strongswan

Finally, we need to setup the EC2 instance as an StrongSwan VPN server for the VPC. We can do this by `ssh`'ing into the instance and setting up strongswan:

```
sudo apt-get update
sugo apt-get install strongswan
```

Now, modify the strongswan configuration with a new connection called `azure`:

```
conn azure
  authby=secret
  type=tunnel
  leftsendcert=never
  left=10.0.0.28
  leftsubnet=10.0.0.0/24
  #leftnexthop=%defaultroute
  right=104.40.19.206
  rightsubnet=10.3.0.0/24
  keyexchange=ikev2
  ikelifetime=10800s
  keylife=57m
  keyingtries=1
  rekeymargin=3m
  #pfs=no
  compress=no
  auto=start
```

Here are the keys you should change for your specific deployment:

- `left` - The local ip address of the strongswan server
- `leftsubnet` - The local subnet of the VPC
- `right` - The public IP address of the Azure VNet Gateway
- `rightsubnet` - The local subnet of the Azure VNet (not to be confused with the gateway subnet)

Now we need to provide StrongSwan with the shared secret. To do this modify the file `/etc/ipsec.secrets` and add the line, replacing with your own values (do not include the `[]` brackets):

`[STRONGSWAN LOCAL IP] [AZURE VNET GATEWAY PUBLIC IP] : PSK "[YOUR SHARED KEY]"`

This should match the shared key used in the azure template parameters from the previous section.

For the strong swan instance to forward traffic between Azure VNet and AWS VPC, we'll have enable forwarding. On the EC2 instance uncomment, or add the following line to the file, `/etc/sysctl.conf `:

```
net.ipv4.ip_forward=1
```

Now, just restart the ipsec service and you should see a connection:

```
ipsec restart
ipsec status
Security Associations (1 up, 0 connecting):
       azure[1]: ESTABLISHED 3 seconds ago, 10.0.0.28[10.0.0.28]...AZURE PUBLIC IP[AZURE PUBLIC IP]
       azure{1}:  INSTALLED, TUNNEL, ESP in UDP SPIs: c3e607f3_i 280cade0_o
       azure{1}:   10.0.0.0/24 === 10.3.0.0/24 
```

### Testing the Connection

We've already created an Ubuntu instance in Azure by deploying the template. Create another ubuntu instance on AWS, associated with the same VPC, subnet and security group.

#### Enable Ping

In order to configure ubuntu to respond to ping requests, run this command on both the AWS and Azure ubuntu instances:

```
iptables -A INPUT -p icmp -j ACCEPT
```

From the AWS instance, attempt to ping the Azure instance:

```
ping [Azure Instance Local IP]
64 bytes from 10.0.0.232: icmp_seq=892 ttl=62 time=26.1 ms
64 bytes from 10.0.0.232: icmp_seq=893 ttl=62 time=26.2 ms
64 bytes from 10.0.0.232: icmp_seq=894 ttl=62 time=25.8 ms
...
64 bytes from 10.0.0.232: icmp_seq=929 ttl=62 time=25.4 ms
```

You should see the same thing as well when you ping the the Azure instance:

```
ping [AWS Instance Local IP]
64 bytes from 10.0.0.232: icmp_seq=892 ttl=62 time=26.1 ms
64 bytes from 10.0.0.232: icmp_seq=893 ttl=62 time=26.2 ms
64 bytes from 10.0.0.232: icmp_seq=894 ttl=62 time=25.8 ms
...
64 bytes from 10.0.0.232: icmp_seq=896 ttl=62 time=26.1 ms
```

You now have two subnets that are connected, one on AWS the other on Azure. Enjoy.