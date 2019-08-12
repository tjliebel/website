# Deploying a BOSH to Google Cloud Platorm using Genesis and Terraform 

Do you have a Google Cloud Platform account? Do you wish things were more automated and just easier to manage? Want to do that using BOSH, but wish there was a simple way to deploy BOSH? Me too. So, let's do that. 

When this is all said and done, you will have a working environment in GCP with a Bastion Host to access the network, a BOSH Director to manage all the deployments, and a secure Vault instance to manage all of the credentials and certificates for the deployment. 

All you need is a fresh GCP project with a service account to perform the automation tasks for us. Create a service account with the following roles: 
* `Compute Engine` > `Computer Admin`
* `Service Accounts` > `Service Account User` 
* `Storage` > `Storage Admin`

Also, create and save the service account key as json to be used later. 

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
service_account_credentials = "path/to/your-sa-creds.json"
service_account_email       = "your-service-account@google-assigned-email.com"

ssh_creds_pub               = { bastion-user = "~/.ssh/id_rsa.pub" }
ssh_creds_priv              = { bastion-user = "~/.ssh/id_rsa" }
```

The `service_account_creds` is the `.json` downloaded above. 

With those set, get things going with the short and sweet:
```
$ make
```
Terraform will now complile all that infomation into a plan for GCP. When prompted, type `yes`. 
```
Plan: 9 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes
```
At this point, terraform will do what it needs to do and deploy the network, subnet, firewall rules, and the bastion host VM. When the dust settles, there should be a nice output looking something this: 
```
Apply complete! Resources: 9 added, 0 changed, 0 destroyed.

~~~ Terraform raw outputs ~~~

For configuring your proto-BOSH:

  Static IP:             10.0.0.3
  Range (CIDR):          10.0.0.0/24
  Gateway:               10.0.0.1
  DNS:                   169.254.169.254

  GCP Project ID:        your-gcp-project-id
  Service account Creds: path-to-your-sa-creds.json
  Network Name:          genesis-bosh-network
  Subnet Name:           genesis-bosh-subnetwork
  Availability Zone:     us-east4-a

To access the bastion host:

  ssh -i ~/id_rsa.pub bastion-user@gcp.assigned.ip.addr
```
That output provides the answers to most of questions that Genesis will ask when setting up the proto-BOSH and what you need to ssh to the bastion host. You can easily get that info again using:
```
$ make info
```

## Setting up the Bastion Host

Go ahead and ssh onto the bastion host using the `ssh` command outputted after the deploy. 
First thing, setup your git config.
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

Use the [Safe CLI](https://github.com/starkandwayne/safe) to startup a local instance of Vault.  Open up another shell session, or tmux pane on the bastion host and run:
```
$ safe local --memory  --as temp-vault
```
Now you should have an in memory Vault instance named `temp-vault` running and targeted. 

## Deploying the Proto-BOSH

First thing is to use Genesis to initialize BOSH. 
```
$ mkdir ~/ops
$ cd ~/ops
$ genesis init bosh -k bosh
```
Genesis will prompt to select a vault, there should only be the one temporary vault that we created for us to select.
```
Select Vault:

  1) temp-vault   (insecure) http://127.0.0.1:8201 (default)

Select choice > 1
```
With that, Genesis will check for the newest kit for bosh and initalize a `bosh-deployments` repository for us.
```
Verifying availability of selected vault...ok

Downloading Genesis kit bosh (latest version)...

Initialized empty Genesis repository in ./bosh-deployments
 - using vault at http://127.0.0.1:8201 (insecure)
 - using the bosh/1.4.0 kit.
```
Next is to tell Genesis what it has to work with to write an environment file for our BOSH director. 

To get started, move into the directory Genesis created and run `genesis new` with a name for the environment, ie `bosh-gcp`. This is our first BOSH, so `y` when asked if this is a proto-BOSH.
```
$ cd bosh-deployments
$ genesis new bosh-gcp
Setting up new environment bosh-gcp...

Using vault at http://127.0.0.1:8201.
Verifying availability...ok


Is this a proto-BOSH director?
[y|n] > y
```
The info needed to answer just about all of the questions below can be found from the `make` that we did on your local machine (you can do `make info` to get that information again). If you did not make any changes to the `.tf` Terraform config, your answers should be the same as mine for everything except the Project-ID and the GCP credentials. 

The first chunk of questions are regarding the networking for the BOSH director.  
```
What static IP do you want to deploy this BOSH director on?
> 10.0.0.3

What network should this BOSH director exist in (in CIDR notation)?
> 10.0.0.0/24

What default gateway (IP address) should this BOSH director use?
> 10.0.0.1
```
This is the DNS resolver that GCP uses. More info can be found in the [GCP documentation](https://cloud.google.com/compute/docs/internal-dns#access_by_internal_DNS).
```
What DNS servers should BOSH use? (leave value empty to end)
1st value > 169.254.169.254
2nd value > 
```
This is where Genesis gets into the GCP specific questions, go ahead and choose item `4` here.  
```
What IaaS will this BOSH director orchestrate?
  1) VMWare vSphere
  2) Amazon Web Services
  3) Microsoft Azure
  4) Google Cloud Platform
  5) OpenStack
  6) BOSH Warden

Select choice > 4
```
These are the two questions that are unique to you. The GCP creds are the service account credentials, provide them as the raw json contents of the `.json` key file.
```
What is your GCP project ID?
> your-gcp-project-id

What are your GCP credentials (generally supplied as a JSON block)? (Enter <CTRL-D> to end)
-------------------------------------------------------------------------------------------
{
  ~~~ Service Account creds as raw JSON ~~~
}
```
Genesis now wants the network, subnetwork, and availability zone for the director to operate on. 
```
What is your GCP network name?
> genesis-bosh-network

What is your GCP subnetwork name?
> genesis-bosh-subnetwork

What availability zone do you want the BOSH VM to reside in?
> us-east4-a
```
We need to give the evironment VM at least one tag. ie `bosh`.
```
What tags would you like to be set on the BOSH VM? (leave value empty to end)
1st value > bosh
2nd value >
```
If you have a desire to look at the environment file at this point, go ahead. Otherwise, the environment file is ready to go without any edits. We can let Genesis take the info we provided and provision the environment.
```
Would you like to edit the environment file?
[y|n] > n
 - auto-generating credentials (in secret/bosh/gcp/bosh)...
 - auto-generating certificates (in secret/bosh/gcp/bosh)...
New environment bosh-gcp provisioned!

To deploy, run this:

  genesis deploy 'bosh-gcp'
```
With that, Genesis has generated passwords and X.509 certs that the BOSH director needs, added them to the temporary vault and we are ready to deploy our BOSH director. 
```
$ genesis deploy bosh-gcp
```
This will likely take some time as it needs to compile quite a few packages. 

## Preparing the BOSH Director for use
The BOSH director is now deployed and running, but it needs a couple more things before we can use it to deploy anything else. Those being an Ubuntu Xenial stemcell and a cloud config. Luckily, we have tools in the BOSH Genesis kit and Terraform makefile to make these things pretty easy for us.

## Uploading Stemcells
Let's start with logging into our new BOSH director. When the BOSH director finished deploying, Genesis outputed a few different command options that are now available to us. Including what we need to login. 
```
$ genesis do bosh-gcp -- login

Using vault at http://127.0.0.1:8201.
Verifying availability...ok

Running login addon for bosh-gcp
Logging you in as user 'admin'...
Using environment 'https://10.0.0.3:25555'

Email (): admin
Password ():

Successfully authenticated with UAA

Succeeded
```
Now we can use Genesis' handy `upload-stemcells` option which is a convenient way to upload stemcells. I chose the latest stemcell that was available at the time. 
```
$ genesis do bosh-gcp -- upload-stemcells

Using vault at http://127.0.0.1:8201.
Verifying availability...ok

Running upload-stemcells addon for bosh-gcp

Select the release family for the google-kvm ubuntu-xenial stemcell you wish to upload:
  1) 456.x (2 minor versions available)
  2) 315.x (14 minor versions available)
  3) 250.x (19 minor versions available)
  4) 170.x (5 minor versions available)

Select choice > 1

Select one of the available 456.x versions:
  1) 456.3
  2) 456.1

Select choice > 1

~~~ Upload Details ~~~

Succeeded

Upload another?
[y|N] > n
```

## Updating Cloud Config

BOSH Cloud Configs can be a bit hand-wavey and hard to follow. So, to streamline the process a bit, we have a function in our Genesis Terraform repo that will output a simple cloud config to get us going. 

From your local machine, in the terraform directory:
```
$ make cc
```
Paste that YAML into a file on the bastion host vm, ie `cloud-config.yml`. Upload that to the BOSH Director.
```
$ bosh -e bosh-gcp update-cloud-config cloud-config.yml
```
This cloud config is pretty basic. It defines defaults for the network, vm types, disk, and availability zone. We also defined another network for the Vault that we will be deploying shortly. 

## Deploying a Real Vault

We deployed a local vault in memory, which works great and does everyhing we need it to do... _Except_ persist outside of the memory of the bastion host, which is a pretty important feature for this entire deployment's credentials. Luckily, we now have a BOSH director that can deploy a Vault for us.

Additionally, there is a Genesis kit for Vault that will do most of the heavy lifting for us. From the bastion host, move outside the `bosh-deployments` directory and initialize the new Vault.
```
$ cd ~/ops
$ genesis init -k vault
```
Select the temporary vault for the BOSH director to use to get this new deployment running.
```
Select Vault:

  1) temp-vault    (insecure) http://127.0.0.1:8201 (default)

Select choice > 1
```
This Genesis kit is quite a bit more simple than the BOSH kit. The only question it will ask at this time is if this is our Genesis Vault, which it is, so `y` when prompted.
```
$ cd vault-deployments
$ genesis new bosh-gcp
```
You can try to deploy right now, but Genesis will complain about the AZs. Vault by default looks to be in 3 availabilty zones. In our case, one AZ is fine for this lab environment. It is worth noting that in prod Vault should be distributed across availabilty zones to prevent losing all the creds to your entire deployment from a single point of failure. 

Open up the `bosh-gcp.yml` and add these params:
```
params:
  availability_zones: [z1]
  stemcell_os: ubuntu-xenial
```
We also add the `stemcell_os` param because the Vault kit might be expecting different stemcells, but the stemcells we gave to BOSH are Xenial. 

Now we are ready to deploy. This will also take some time, but slightly less than the Proto-BOSH. 
```
$ genesis deploy bosh-gcp
```

## Transferring Secrets to the new Vault

We now have a full fledged Vault instance deployed and ready to go.  Initialize the new Vault.
```
$ genesis do bosh-gcp -- init
```
Export all the creds from the local `temp-vault` to this new Vault.
```
$ safe -T temp-vault export | safe -T bosh-gcp import
```
With those transfered, it is safe to kill the local temporary vault process. 

That is a BOSH director primed and ready to deploy whatever you need in GCP! Check out other what other Genesis kits are available at [Genesis Community](https://github.com/genesis-community).  