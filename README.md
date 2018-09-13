# Deploying Linux to Azure

This is the companion guide for my *Deploying Linux to Azure* session delivered as part of the [Open Source Virtual Conference](https://info.microsoft.com/CA-AzureOSS-WBNR-FY19-07JUL-24-01OSS-Conference-MASTER-VIRTUAL-EVENT-PROGRAM.html) in Canada.

## Prerequisites

* In your laptop:
  * Download and install [Visual Studio Code](https://code.visualstudio.com/download)
    * Install the [Ansible extension](https://marketplace.visualstudio.com/items?itemName=vscoss.vscode-ansible) for Visual Studio Code
    * Install the [Azure Account extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode.azure-account) for Visual Studio Code
  * Install [Azure CLI 2.0](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
  * Clone this repo, for example with `git clone https://github.com/bureado/linux-on-azure-guides`
  * Make sure you have an SSH keypair, either by looking in `~/.ssh/id_rsa.pub` (in your Mac, Linux or Cloud Shell environments) or creating a new one with `ssh-keygen`
* In Azure:
  * Have a working subscription (be able to log in, be able to create resources)
  * Click on the Cloud Shell icon and go through the [one-time setup process](https://docs.microsoft.com/en-us/azure/cloud-shell/quickstart)

## Walkthrough

Here's the walkthrough as presented in the conference, roughly separated in chapters.

1. Login to the [Azure Portal](https://portal.azure.com) and open the Azure Cloud Shell by clicking the icon next to the search bar
2. Clone this repository on Cloud Shell: `git clone https://github.com/bureado/linux-on-azure-guides`
3. Create a new resource group: `az group create -g virtual -l canadaeast` (you can pick any name for `-g`, and deploy to other locations with `-l`)
4. Search for the resource group you just created in the Azure Portal and pin it to your Dashboard by clicking on the pin icon

### Launching VMs

5. Create a new Linux VM using the Azure CLI: `az vm create -g virtual -n simple --image UbuntuLTS --no-wait` (you can pick any name for the VM for `-n`)
6. Create a new VM using the `cloud-init.yaml` file in this repo: `az vm create -g virtual -n custom --image UbuntuLTS --custom-data ./cloud-init.yaml --no-wait` (notice we changed the name from `simple` to `custom`, but you can use any name you want)
7. Create two additional VMs, in this case CentOS: `for name in db web ; do az vm create -g virtual -n $name --image CentOS --no-wait ; done`
8. Go to the resource group you pinned in step 4 (or refresh) so you can see the new resources being created
9. Open a new tab and go to the [Azure Resource Explorer](https://resources.azure.com), browse to your subscription on the left, then click on _resourceGroups_ and browse to the group you created in step 3 (in my case, called `virtual`) and then click _resources_. Browse the JSON output on the right.
10. Now, in the same tab, on the left pane go to _providers_, _Microsoft.Compute_, _virtualMachines_, and click for example on `web`. Browse the JSON output on the right. Note that many VM defaults were assumed by the Azure CLI when you created the VMs, but that you can customize those.

### Log in to the VMs

11. Go back to the Azure Portal, and list all the VMs in the resource group using Azure CLI in Cloud Shell: `az vm list -g virtual -o tsv --show-details`. Notice the output includes, for example, the public IPv4 addresses of the VMs.
12. From Cloud Shell, SSH into any of the VMs, for example the IP address for the `simple` VM: `ssh <IP>`. Notice you don't need to specify username or credentials, because those were derived from Cloud Shell.
13. Try step 12 again, but with one of the CentOS VMs (`web` or `db`): `ssh <IP>`

### Deploy from Ansible

14. Switch to Visual Studio Code, and open the vm_create_ssh.yml file. Make sure you adjust the following parameters:
  * `vars.resource_group` should point to the resource group you created in step 3
  * `vars.location` should match the location you're working on (in my case, `canadaeast`)
  * `vars.ssh_key` should be the public SSH key you want to use for authentication (for example, copy/paste the contents of `~/.ssh/id_rsa.pub`)
15. Right click on the file name on the Explorer, and select _Run Ansible Playbook in Cloud Shell_.
16. Repeat step 11 to obtain the IPv4 addresses of your VMs, and write down the IP addresses of the VMs `db` and `web` created in step 7

(The following steps can be performed either in Cloud Shell or in Visual Studio Code)

17. Edit the file `lamp/hosts` and add the IP addresses under `[webservers]` and `[dbservers]`
18. Run the Playbook in Cloud Shell: `ansible-playbook -i hosts site.yml` or in Visual Studio Code by right clicking the `site.yml` file and selecting _Run Ansible Playbook in Cloud Shell_
19. In the Azure Portal, refresh the resource group view and notice new resources being created for example, the `fc0` VM. Capture the public IP address of that VM.
20. From wherever the private key corresponding to the public key you used in step 14 is, try logging in via SSH: `ssh azureuser@<IP>`

### Deploying a Scale Set

21. In Cloud Shell, create a new Scale Set: `az vmss create -g virtual -n set --image UbuntuLTS --custom-data ./vmss-data.yaml` which will make scaling out _with custom data_ much easier. We'll come back to VMSS later.

### Deploying an Azure Database for MySQL

22. In Visual Studio Code, browse to `mysql_create.yml`. Make sure the following parameters are set up correctly:
  * `vars.resource_group` should point to the resource group you created in step 3
  * `vars.location` should match the location you're working on (in my case, `canadaeast`)
  * And, if desired, you can customize the `vars.mysqldb_name`, `vars.admin_username` and `vars.admin_password` variables.
23. Right click on `mysql_create.yml` in the Explorer and select _Run Ansible Playbook in Cloud Shell_. We'll come back to MySQL later.

### Scale out the Scale Set

24. Switch back to the Azure Portal, and search for `set`, browse to the Virtual machine scale set resource. Note the IP address. Browse to *Instances* and notice the VMs that have been created.
25. Switch back to the Azure Resource Explorer. Under the `virtual` resource group, browse to _providers_, _Microsoft.Compute_, _virtualMachineScaleSets_ and then click on `set` (you might need to refresh your tab), notice the instance count
26. Click on the _Read/Write_ button on the top right of the screen, then click on the _Edit_ button next to _GET_. The text box will become editable. Change the instance capacity from 2 to, for example, 5. Then click `PATCH`.
27. Go back to the Azure Portal and refresh the `set` blade. Notice the additional instances being created.

### Use managed MySQL

28. In Cloud Shell, run `az mysql list`, notice the managed MySQL instance that was created by Ansible.
29. In the Azure Portal, search for the MySQL instance you've created, then go to _Connection security_ and enable the option _Allow access to Azure services_, then click _Save_.
30. In Cloud Shell, run `mysql -u user@<name> --password -h <fullyQualifiedDomainName>`, using the parameters from step 28, and type in the password `Maple001$`. You can create MySQL tables, etc. Exit the MySQL CLI by typing `quit;`

### Enable external networking for the demo applications

31. Enable a frontend for the load balancing pool in the scale set: `az network lb rule create -g virtual -n myLBRuleWeb --lb-name setLB --backend-pool-name setLBBEPool --backend-port 3000 --frontend-ip-name loadBalancerFrontEnd --frontend-port 80 --protocol tcp`
32. Browse to the app in the VM scale set, by using the IP address of the scale set, obtained in step 24.
33. In step 6, you created a `custom` VM with a Web application. You can enable TCP port 80 in the firewall by searching for `customNSG`, clicking on _Inbound security rules_, clicking _Add_, and make sure 80 is in _Destination port ranges_, then click _Add_. You can then browse the web application in that VM by calling the IP directly, obtained in step 11.

### Exploring a Linux VM on Azure

Here's a list of useful commands you can use to explore a Linux VM on Azure, in this case, an Ubuntu-based VM:

    pgrep systemd
    sudo systemd-detect-virt
    lsmod | grep hv
    ps aux | grep waaagent
    hostname -f
    ip addr
    mount | grep sd
    cat /etc/fstab
    grep azure /etc/apt/sources.list

### Enabling backups for a Linux VM on Azure

34. In the Azure Portal, using the search box, search for the `custom` virtual machine, click on it, then browse to _Backup_
35. Type in your resorce group name, in my case, `canada`, and click _Enable backup_ (this uses the default backup policy)

## Follow this repo for more!

Topics in the TODO for this repo:

* Instance Metadata Service
* Accelerated Networking
* Performance tuning
* Storage architecture for Linux on Azure
* More scale-out cluster goodness
* Monitoring and security
* Custom images from on-premises, incl. with Packer

## Resources

* [What is _actually_ supported in Ansible?](https://docs.microsoft.com/en-us/azure/ansible/ansible-matrix)
* [How to do dynamic inventory in Ansible?](https://docs.ansible.com/ansible/2.6/scenario_guides/guide_azure.html#dynamic-inventory-script)
* [Automate Azure resources with Ansible](https://channel9.msdn.com/Shows/Azure-Friday/Automate-Azure-Resources-with-Ansible)
* [Information for non-endorsed distributions](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/create-upload-generic)
