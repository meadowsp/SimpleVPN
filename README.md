# Simple VPN

## Description
### Overview
Simple VPN creates a small proof of concept VPN hosted in AWS based on WireGuard.

The architecture is defined to be as efficient (cheap :-) ) as possible. The server is running AWS Linux on an [AWS Graviton](https://en.wikipedia.org/wiki/AWS_Graviton) 64 bit ARM processor, using the smallest instance size, a [t4g.nano](https://aws.amazon.com/ec2/instance-types/t4/). This is deployed into Ohio (US-EAST-2).

The architecture defined in the templates is for one VPN server, which can support 2 clients behind a NAT'd home static IP address. When the VPN is enabled the server will have the IP address of 10.0.0.1, the first client (peer 1) will have the IP address of 10.0.0.2 and the second client will be running on 10.0.0.3.

### Prerequisites
* A domain name managed in AWS Route53. This is to define the VPN endpoint
* An AWS EC2 Key Pair. This is used to authenticate SSH access to the VPN server. Instructions on how to set one up are [here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html)
* A set of WireGuard Public/Private Keys for the server and for both clients. Instructions on how to create them are [here](https://www.wireguard.com/quickstart/#key-generation). For the initial setup, you can install wireguard on one of your clients and generate 3 key pairs there. Keep these safe, preferably in a password manager or equivalent

## Cloud Setup
### Initialisation Template
The first cloudformation template that you need to run is `vpncfninit.yaml`. This has the following parameters that you need to enter:
* `EC2IamRoleNameParameter` - The name of the IAM profile that will be used by the server EC2 instance to authorise access to the keys. Can be left as the default value.
* `SecretsNameParameter` - The name of the AWS Secrets Manager Secret that will be used to store the keys. Unless you need to change this, you can leave this as the default value. All of the keys will be stored in this secret. By definition the public keys don't need to be secret, but it makes things easier to keep them all in one place.
* `VPNPeer1PublicKey` - The public key for client/peer 1
* `VPNPeer2PublicKey` - The public key for client/peer 2
* `VPNServerPrivateKey` - The private key for the VPN server
* `VPNServerPublicKey` - The public key for the VPN server

This template will create a secrets vault in AWS Secrets Manager, store the keys in there and create the IAM instance profile which will allow the server to be able to read the keys on initialisation.

### Stack Template
The second cloudformation template to run is `vpncfnstack.yaml`. This builds out the VPN server, installs and configures WireGuard on it, sets up the DNS name and the security groups to only allow network ingress from the home network IP over the SSH and Wireguard ports. It has the following parameters that need entering. If any of the default values in the initialisation stack were changed, then the updated values will need entering here.
* `AMIParameter` - The default AMI is for AWS Linux in US-EAST-2. You will need to change this if you are building elsewhere.
* `DNSNameParameter` - The subdomain to set up for the DNS. You can leave the default of 'dns'
* `DomainParameter` - The domain that you already have set up in AWS Route 53. For example, if you own example.com and have this set up in Route 53, and you leave the default `DNSNameParameter`, then your VPN will be available on `vpn.example.com`
* `HomeIpParameter` - The static IP address of your home network
* `IAMInstanceParameter` - This relates to the EC2IamRoleNameParameter parameter in the initialisation template.
* `InstanceTypeParameter` - The EC2 instance type to use. Defaults to t4g.nano, the cheapest of the instances
* `KeyNameParameter` - The name of your AWS EC2 key pair, used to gain SSH access to the server.
* `SecretsParameter` - This relates to SecretsNameParameter in the initialisation template. 
* `VPNPortParameter` - Which port you want WireGuard to be running on, defaults to 51820

This stack can be deleted and rebuilt independently of the initialisation stack.

### Troubleshooting
If you want to see what's going on, you can SSH into the VPN server by doing the following (substituting the name of your local pem file for your key and the domain name of your VPN)
```
ssh -i mykey.pem ec2-user@vpn.example.com
```
If all is well then `sudo tail /var/log/cloud-init-output.log` should look something like
```
+ sysctl -p
net.ipv4.ip_forward = 1
+ wg-quick up wg0
[#] ip link add wg0 type wireguard
[#] wg setconf wg0 /dev/fd/63
[#] ip -4 address add 10.0.0.1/24 dev wg0
[#] ip link set mtu 8921 up dev wg0
[#] iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
Cloud-init v. 19.3-46.amzn2 finished at Fri, 03 Feb 2023 14:32:00 +0000. Datasource DataSourceEc2.  Up 88.98 seconds
```
## Client Setup
Client setup will be different based on the type of client that you're running. WireGuard installation instructions are available [here](https://www.wireguard.com/install/)

I run Ubuntu on my clients and I have a configuration file `/etc/wireguard/wg0.conf` that looks like
```
[Interface]
Address = 10.0.0.2/32
PrivateKey = CLIENT_1_PRIVATE_KEY
DNS = 9.9.9.9

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```
Replace CLIENT_1_PRIVATE_KEY with the private key created earlier for the first client (peer 1). Replace SERVER_PUBLIC_KEY with the public key of the server and update the domain value for the endpoint to be your domain.

The config is the same for client 2, but with the local IP address of 10.0.0.3 and the private key for client 2.

You can put whatever you want for the DNS resolver, but for security I like [Quad 9](https://www.quad9.net/), which is `9.9.9.9`

To start up the VPN locally, `wg-quick up wg0`, check your IP and you should be reporting as coming from somewhere in Ohio.
