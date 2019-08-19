# Managing Secrets in Genesis: A Guide to Vault

Vault is an open-source secrets management software. We will talk a bit about the problem that secrets present, vault as a solution to that problem, and using genesis and safe to implement vault in a deployment. I will also assume that you used Genesis to deploy your primary vault as done in any of the Deploying Bosh using Genesis posts. 

## The Problem with Secrets

Even a simple software deployment is riddled with secrets. SSH, API, and RSA keys, certificates, usernames, passwords - the list goes on. So that begs the question, where do we store all of that? 

We could hard code them. Make the repo private, only those in your org can see it. Great, no problem. Until someone accidentally makes the wrong repo public, and your whole deployment is exposed. 

Then how about using environment variables? Keep them all local, awesome. Well, they are in your environment and you need to share them with the team. Oh, we'll just put them in a private repo just for the env vars! Wait, but we already ruled that out. 

Then lets use a database. That'll do it. As long as you trust your database administrator, the database host against online attacks, and that no one can get to the physical disks for an offline attack... Okay, lets encrypt the data base, but then, what do we do with the encryption key? We could hard code it! ...Or make it an environment variable! ...Or Store it in the database next to what it is encrypting? 

Vault can help with the problem of storing the secrets for the deployment. It isn't perfect, there is still the case of the unseal keys, but there are solutions in the Vault Genesis kit to help with that as well. 

## Basics of Sealing and Unsealing

Vault encrypts all of its contents and the key is only ever held in memory. This means that the vault will automatically "reseal" itself on reboot. What if you want to manually seal the vault in a "break glass in case of emergency" type of situation. The vault we deployed is by default in [High Availability Mode (HA)](https://www.vaultproject.io/docs/concepts/ha.html). In our case, there are three instances of the same vault each on their own VM. If one is a primary node and if that goes down, there are two redundant nodes ready to take over. That means that each vault node needs to be sealed starting with the active node, then recursively finding the new active node one by one until all are sealed Not too bad in our deployment of 3 nodes. I have seen as many as 5 or 7 redundant nodes. Safe makes sealing the entire vault easy. 

    $ safe seal
    sealed https://10.0.0.5:443...
    sealed https://10.0.0.4:443...
    sealed https://10.0.0.6:443...

So we can seal a vault manually with safe and we know that the vault will be sealed on reboot. What about unsealing?

When the vault is initialized with the Vault Genesis kit, you (hopefully) noticed Genesis write this to the terminal:

    Unseal Key #1: ~~~ Key 1 ~~~
    Unseal Key #2: ~~~ Key 2 ~~~
    Unseal Key #3: ~~~ Key 3 ~~~
    Unseal Key #4: ~~~ Key 4 ~~~
    Unseal Key #5: ~~~ Key 5 ~~~
    Initial Root Token: ~~~ Token ~~~

    Vault initialized with 5 keys and a key threshold of 3. Please
    securely distribute the above keys. When the Vault is re-sealed,
    restarted, or stopped, you must provide at least 3 of these keys
    to unseal it again.

    Vault does not store the master key. Without at least 3 keys,
    your Vault will remain permanently sealed.

These keys and token are important. The token is used with `safe auth` to authenticate with the vault. This is saved in your `.saferc`, but would be needed if that file is lost or you were allowing another machine access to the vault. 

As the output above states, three keys must be provided to build the master key. This means you can store all those keys yourself, or divide them up - give one to yourself, John, Jane, Mary, and Mark. 

As long as you have the keys, unsealing with safe is straight forward. 

    $ safe status
    https://10.0.0.5:443 is sealed
    https://10.0.0.6:443 is sealed
    https://10.0.0.4:443 is sealed

    $ safe unseal
    You need 3 key(s) to unseal the vaults.

    Key #1:
    Key #2:
    Key #3:
    unsealing https://10.0.0.5:443...
    unsealing https://10.0.0.6:443...
    unsealing https://10.0.0.4:443...
    
    $ safe status
    https://10.0.0.4:443 is unsealed
    https://10.0.0.5:443 is unsealed
    https://10.0.0.6:443 is unsealed

## Primary vs Secondary Vault

When you deploy a vault using the Vault Genesis kit, the first thing it asks is:

    Is this your Genesis Vault (for storing deployment credentials)?
    [y|n] > 

When you select `y` here, Genesis makes that deployment the primary vault for the entire deployment. This will store the creds required to deploy any apps - things like creds to access vms to spin up applications, not creds for individual applications or users. For example, say the dev team wants to store creds for their app in a vault. Naturally, you don't want to give every engineer on the dev team access to your primary vault. You would command `genesis new vault-dev`, select `n` when prompted, deploy with `genesis deploy vault-dev` then that team performs `genesis do vault-dev -- init`. It is then up to them to actually manage the vault and its contents. But you would still manage the deployment. 

## Vault in a Pipeline

Using Vault in a pipeline can be tough. For example, new stemcells are released, the pipeline goes into action and starts with primary vault. The vault deployment updates just fine. The pipeline moves to the next deployment, asks vault for the creds - vault was just sealed at reboot...

Upon initializing the primary vault with the vault Genesis kit, the unseal keys are stored inside the newly initialized vault. That may seem like a strange place to store the keys, like locking your keys inside your car. This allows Genesis to grab the unseal keys prior to rebooting the vault deployment. Genesis then unseals the vault at reboot and the pipeline can continue running. 

## Working with Multiple Vaults

If you have multiple deployments, you may be dealing with multiple vaults. Safe has a convenient setup for that as well. 

    $ safe targets
    Known Vault targets - current target indicated with a (*):
    (*) buffalo-lab	 (noverify) https://10.128.4.4
        genesis-ci 	 (noverify) https://10.128.8.12
        jumpbox    	 (insecure) http://10.128.80.192:8200

To change targets:

    $ safe target genesis-ci
    Now targeting genesis-ci at https://10.128.8.12 (skipping TLS certificate verification)

    $ safe targets
    Known Vault targets - current target indicated with a (*):
        buffalo-lab	 (noverify) https://10.128.4.4
    (*) genesis-ci 	 (noverify) https://10.128.8.12
        jumpbox    	 (insecure) http://10.128.80.192:8200

To add a new target:

    $ safe target my-vault https://vault.ip
    Now targeting my-vault at https://vault.ip

    $ safe targets
    Known Vault targets - current target indicated with a (*):
        buffalo-lab	 (noverify) https://10.128.4.4
        genesis-ci 	 (noverify) https://10.128.8.12
        jumpbox    	 (insecure) http://10.128.80.192:8200
    (*) my-vault   	            https://vault.ip

Use `-k` to skip TLS:

    $ safe -k target  my-vault-k https://vault.ip
    Now targeting my-vault-k at https://vault.ip

    $ safe targets
    Known Vault targets - current target indicated with a (*):
        buffalo-lab	 (noverify) https://10.128.4.4
        genesis-ci 	 (noverify) https://10.128.8.12
        jumpbox    	 (insecure) http://10.128.80.192:8200
        my-vault   	            https://vault.ip
    (*) my-vault-k 	 (noverify) https://vault.ip