# Deploying a BOSH to Google Cloud Platorm using Genesis and Terraform 

Identify the need the reader has, scope of the blog ("when you are done you will have X"), first things/expectations

##  Creating a Service Account

A service account is an account that can perform automation tasks on GCP. 

Navigate to `IAM & Admin > Service accounts`, or search for `Service accounts` and select the `IAM & admin` option.
Create a new service account. Give it a name and description, ie. `bosh-terraform`. Give it the role `Service Accounts` > `Service Account User`. Give users admin/user privledges as you like for this account (at least yourself). And finally create and save a key as json to be used later. 

## Getting Started with Terraform

Terraform is a simple way to manage our infrastructure on GCP. Terraform documentation and binaries can be found at [https://terraform.io](https://terraform.io). Clone [this repo](https://github.com/genesis-community/terraforms) to get started
```
$ git clone https://github.com/genesis-community/terraforms
```
We will be working in the gcp lab area, here you will find a sample terraform config file that is ready to go, only needing a few inputs from you. So make a `gcp.tfvars` file in the gcp lab area.
```
$ cd terraforms/lab/gcp
$ vim gcp.tfvars
```
Here are the variables that you need to define and what the file should look like:
```
project_id                  = "your-gcp-project-id"
service_account_credentials = "path-to-your-sa-creds.json"
service_account_email       = "your-service-account@google-assigned-email.com"

ssh_creds_pub               = { bastion-user = "~/id_rsa.pub" }
ssh_creds_priv              = { bastion-user = "~/id_rsa" }
```

Note that the `service_account_creds` is the `.json` downloaded above. 

With those set, get things going with the short and sweet:
```
$ make
```
Terraform will now complile all that infomation into a plan for GCP. When prompted, type `yes`. 
```
Plan: 7 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes
```
At this point, terraform will do what it needs to do and deploy the network, subnet, firewall rules, and the bastion host VM. When the dust settles, there should be a nice output looking something this: 
```
Apply complete! Resources: 7 added, 0 changed, 0 destroyed.

~~~ Terraform raw outputs ~~~

For configuring your proto-BOSH:

  Static IP:             10.0.0.3
  Range (CIDR):          10.0.0.0/24
  Gateway:               10.0.0.1
  DNS:                   8.8.8.8, 8.8.4.4

  GCP Project ID:        your-gcp-project-id
  Service account Creds: path-to-your-sa-creds.json
  Network Name:          genesis-bosh-network
  Subnet Name:           genesis-bosh-subnetwork
  Availability Zone:     us-east4-a

To access the bastion host:

  ssh -i ~/id_rsa.pub bastion-user@gcp.assigned.ip.addr
```
That output provides the answers to most of questions that Genesis will ask when setting up the proto-BOSH and what you need to ssh to the bastion host. You can easily get that info again using 
```
$ make info
```

## Setting up the Bastion Host

Go ahead and ssh onto the bastion host using the `ssh` command outputted after the deploy. 
First thing, setup your git config:
```
$ git config --global user.name "Your Name"
$ git config --global user.email "Your Email"
```
This bastion host is currently a fresh ubuntu VM. As part of the Terraform config, we pushed over a script that will get us what  we need to stand up our Proto-BOSH using Genesis.
```
$ sudo jumpbox system
```
The things being installed by this script include `bosh`,`cf`, and `genesis` as well as some other utilities such as [safe](https://github.com/starkandwayne/safe), [spruce](https://github.com/geofffranks/spruce), and [jq](https://github.com/stedolan/jq). 

To See what is installed, you can run `jumpbox` with no arguements.
```
$ jumpbox
~~~ jumpbox art ~~~
>> Checking jumpbox installation
   jumpbox installed - jumpbox v55
   ruby installed - ruby 2.3.1p112 (2016-04-26) [x86_64-linux-gnu]
   bosh installed - version 3.0.1-712bfd7-2018-03-13T23:26:43Z
   cf installed - cf version 6.46.0+29d6257f1.2019-07-09
   jq installed - jq-1.5
   spruce installed - spruce - Version 1.22.0
   safe installed - safe v1.3.0
   vault installed - Vault v0.9.6 ('7e1fbde40afee241f81ef08700e7987d86fc7242')
   genesis installed - Genesis v2.6.17 (6dfca15edc) build 20190719.205359
   sipcalc installed - sipcalc 1.1.6

   git user.name  is 'Troy Liebel'
   git user.email is 'tliebel@starkandwayne.com'

   To bootstrap this installation,  try `jumpbox system`
   To set up your personal environment: `jumpbox user`
   To update this copy of jumpbox, use: `jumpbox sync`
   To create a new local user account:  `jumpbox useradd`
```

## Creating a Temporary Vault

[Vault](https://www.vaultproject.io/) is useful tool that Genesis taps into to generate and manage SSH keys, certs, and more. Once we get BOSH up and running, we will deploy an instance of Vault to manage those things, but until then we need something to get us started. We can do that by starting a local vault instance. 

On the now setup bastion host, use the [Safe CLI](https://github.com/starkandwayne/safe) to startup a local instance of Vault.  Open up another shell session, or tmux pane on the bastion host and run:
```
$ safe local --memory
```
Now you should have an in memory Vault instance running and targeted. 

## Deploying the Proto-BOSH

First thing is to use Genesis to initialize BOSH. 
```
$ genesis init bosh -k bosh
```
Genesis will prompt to select a vault, there should only be the one temporary vault that we created for us to select.
```
Select Vault:

  1) unbreakable-donjon   (insecure) http://127.0.0.1:8201 (default)

Select choice > 1
```
With that, Genesis will check for the newest kit for bosh, and initalize a `bosh-deployments` repository for us.
```
Verifying availability of selected vault...ok

Downloading Genesis kit bosh (latest version)...

Initialized empty Genesis repository in ./bosh-deployments
 - using vault at http://127.0.0.1:8201 (insecure)
 - using the bosh/1.4.0 kit.
```
Next is to tell Genesis what it has to work with to write an environment file for our BOSH director. 
```
$ cd bosh-deployments
```

```
$ genesis new bosh-gcp
Setting up new environment bosh-gcp...

Using vault at http://127.0.0.1:8201.
Verifying availability...ok


Is this a proto-BOSH director?
[y|n] > y
```

from `make info`

```
What static IP do you want to deploy this BOSH director on?
> 10.0.0.3

What network should this BOSH director exist in (in CIDR notation)?
> 10.0.0.0/24

What default gateway (IP address) should this BOSH director use?
> 10.0.0.1

What DNS servers should BOSH use? (leave value empty to end)
1st value > 8.8.8.8
2nd value > 8.8.4.4
3rd value >
```

```
What IaaS will this BOSH director orchestrate?
  1) VMWare vSphere
  2) Amazon Web Services
  3) Microsoft Azure
  4) Google Cloud Platform
  5) OpenStack
  6) BOSH Warden

Select choice > Google Cloud Platform

What is your GCP project ID?
> gcp-genesis-docs


What are your GCP credentials (generally supplied as a JSON block)? (Enter <CTRL-D> to end)
-------------------------------------------------------------------------------------------
{
  ~~~ Service Account creds as raw JSON ~~~
}

What is your GCP network name?
> genesis-bosh-network

What is your GCP subnetwork name?
> genesis-bosh-subnetwork

What availability zone do you want the BOSH VM to reside in?
> us-east4-a

What tags would you like to be set on the BOSH VM? (leave value empty to end)
1st value > bosh
2nd value >
```

```
Would you like to edit the environment file?
[y|n] > n
 - auto-generating credentials (in secret/bosh/gcp/bosh)...
 - auto-generating certificates (in secret/bosh/gcp/bosh)...
New environment bosh-gcp provisioned!

To deploy, run this:

  genesis deploy 'bosh-gcp'
```

When asked, select the temporary vault that was just created. 