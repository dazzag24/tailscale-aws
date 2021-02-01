## Overview

![license](https://img.shields.io/badge/license-MIT-green)

This repo contains a CloudFormation template that automates the deployment process described in the [Setting up Tailscale on AWS EC2](https://tailscale.com/kb/1021/install-aws) project knowledgebase article.  See also this article on [subnet routes](https://tailscale.com/kb/1019/subnets).

_With the default configuration this solution will deploy resources that cost about $0.01/hr_

Resource List:

* A VPC with one public and one private subnet
* A NAT/Bastion EC2 instance provisioned in the public subnet
* A second EC2 instance provisioned in the private subnet

Both of the EC2 instances have the Tailscale software installed and configured but need to be initialized using commands run on the console of each node and then an additional configuration step in the [Tailscale management console](https://login2.tailscale.io/admin).

Use [t3a.nano](https://aws.amazon.com/ec2/instance-types/t3/) EC2 instances which are AMD based.

I also tried to use a [t4g.nano](https://aws.amazon.com/ec2/instance-types/t4/) which are ARM (Graviton) based [Amazon Linux 2](https://aws.amazon.com/amazon-linux-2/) instances but these fail when deploying the NAT server with an unhelpfully generic error:

`Status Reason: The requested configuration is currently not supported. Please check the documentation for supported configurations. Launching EC2 instance failed.`

See this for more details: https://stackoverflow.com/questions/45691826/aws-launch-configuration-error-the-requested-configuration-is-currently-not-sup

Furthermore the installation of tailscale currently fails on Amazon Linux 2, so I have changed the template to use Ubuntu 20.04.

If the deployment does not work then look at the cloudformation deloyment logs to help find the cause:

`aws cloudformation describe-stack-events --stack-name tailscale-vpn`

If these are OK, then perhaps some aspect of the cloud-init script failed.  Log into the instances and use the contents of /var/log/cloud-init-output.log on the deployed EC2 instance to see what happened.


## Step 1: Deploy the CloudFormation Template

If you're using Mac or Linux _and_ have your AWS CLI configured for your account you can deploy using the project Makefile.  

* First

* Copy the sample .env file (`.env-example`) to become your local `.env` file (DO NOT CHECK THIS INTO GITHUB).  Most likely you will need to edit the SSH key name to match yours, but you WILL HAVE TO configure it with your own Tailscale pre-authentication key.  Go to https://login.tailscale.com/admin/authkeys and create a reusable key You can optionally update the SSH_ALLOWED_IPS variable.
  * `SSH_KEY='<YOUR_SSH_KEY_HERE>'` <~~ Your key name here
  * `TAILSCALE_PREAUTH_KEY='<PLACE_YOUR_TAILSCALE_PREAUTH_KEY_HERE>'`


* Deploy or delete using make
  * `make deploy`
  * `make delete`

## Step 2: Configuration

### Step 2.1 - SSH Config File

Read the SSH Config File Suggestion section below for a suggestion on how to configure your SSH using a config file. You can accomplish the ssh steps in a number of different ways but the remainder of this section assumes you've set up your config file as suggested. If you can't connect try running ssh with the `-v` flag to identify the connection problem.

Using config file example: `ssh -F ssh.config ts-nat-public`

### Step 2.2 - Configure the NAT 

If the cloudformation deployment is successfuly you will see the EC2 nodes appear and you should also see 2 new nodes appear in your Tailscale admin page https://login.tailscale.com/admin/machines.

* From the Tailscale web console click the Enable Subnet Routes option for the NAT server

* The routes to both the public and private subnets have been configured as described in the [documentation](https://tailscale.com/kb/1019/install-subnets) but an additional step of Enabling Subnet Routes from the admin console must be completed before connectivity to nodes in the private subnet can be established.

## Step 3: Tests

* To test an individual node you should be able to curl the IRC web client from the console
  * `curl http://100.101.102.103`
  * If this command doesn't succeed and return a redirect then the node has not been configured correctly
* To test connectivity between nodes you should be able to ping the 100.x.x.x address between any two nodes
* If you've installed the client on your Mac or Windows workstation you should be able to ping or SSH into either of these EC2 instances using their 100.x.x.x IPs despite the fact that one of them is in a private subnet and doesn't even have a public IP
* In your ssh.config file update the HostName values for the `ts-nat` and `ts-node` blocks with the 100.x.x.x IPs found in the admin console and attempt to ssh into the servers again using their Tailscale IPs

## Step 4: Delete the CloudFormation Stack

To tear down the stack and release all the provisioned resources [use the Delete button](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-delete-stack.html).

## Misc Notes

### SSH Config File Suggestion

Once you have successfully tested you may want to merge the content of the ssh.config permanently witj your `~/.ssh/config` file to map the various name and ip combinations. An example below showing the two nodes below described in Step 2 of the setup instructions.

Update the x.x.x.x placeholders with the IPs of your servers and change the IdentityFile name if you didn't save your key pair as ts.pem

```
# The public IP of the NAT server
Host ts-nat-public
	HostName x.x.x.x

# The Tailscale IP of the NAT server once configured
Host ts-nat
	HostName 100.x.x.x

# The private IP of the node in the private subnet
Host ts-node-private
    HostName x.x.x.x
    ProxyJump ts-nat-public

# The Tailscale IP of the node in the private subnet
Host ts-node
    HostName 100.x.x.x

# Settings that apply to all hosts. Update IdentityFile if needed.
Host *
	User ec2-user
    IdentityFile ~/.ssh/ts.pem
```

### Contributing

Run the lint utilities to ensure the CloudFormation/YAML stays tidy

```
# Once per project
python3 -m venv ./venv
pip install -r requirements.txt

# Once per session
. ./venv/bin/activate

# When you want to check
make lint
```
