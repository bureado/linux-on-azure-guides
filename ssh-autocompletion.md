# SSH autocompletion for Azure VMs

By default, a Linux VM you create with the minimal `az vm create` options (name, resource group and image) will provision a public IPv4 address, but not a FQDN.

When you have really large, production-scale Linux deployments, you'll probably enable those FQDNs in the default, Azure-provided domains or use your own domains.

For smaller scale and POC deployments, sometimes it's annoying to keep track of the VM names you're creating and query the public IPv4 addresses of those VMs. Yes, you can do so with `az vm list --show-details -o tsv`, for example, but you still have to copy/paste IPs around, etc.

Very simplistically, if all you want is to be able to type `ssh` and then use tab autocompletion to get the VM names and then connect to the right IPv4 address, there are a number of approaches you could take, from the very simple, like adding the hostnames to `/etc/hosts` to the more complicated (writing an NSS plugin for Azure VMs) and many options in between.

Here, I document the `ssh_config` route. In the global SSH client configuration file for OpenSSH (`/etc/ssh/ssh_config`) or the per-user one (`~/.ssh/config`) you can define `Host` stanzas that can provide specific SSH client options for each host (that override the defaults for all hosts), and that includes, for example, specifying the hostname/IP address or the username.

A script that integrates `az vm list` (and its JSON output) with `ssh_config` could look like:

  #!/bin/sh

  if [ -f ˜/.ssh/config ]
  then # Backup config, if it exists
    cp ~/.ssh/config ~/.ssh/config.orig
  fi

  # Reset the VM IP list
  echo > ~/.ssh/azure_vms

  # Go through the `az vm list` output
  for vm in `az vm list --show-details | jq -r '.[]|[.name,.resourceGroup,.publicIps,.osProfile.adminUsername]|@csv'`
  do
    name=`echo $vm | cut -f1 -d, | sed 's/\"//g'`
    group=`echo $vm | cut -f2 -d, | sed 's/\"//g'`
    ip=`echo $vm | cut -f3 -d, | sed 's/\"//g'`
    user=`echo $vm | cut -f4 -d, | sed 's/\"//g'`
    if [ -z "$ip" ] # No PIP assigned
    then
      continue
    else # TODO: repeat hostname handling
      echo "Host $name" >> ~/.ssh/azure_vms
      echo "      User $user" >> ~/.ssh/azure_vms
      echo "      HostName $ip" >> ~/.ssh/azure_vms
      echo >> ~/.ssh/azure_vms
    fi
  done

  if grep "#AZURE_VMS" ~/.ssh/config > /dev/null
  then # TODO: there's previous VMs to update
    echo
  else
    echo "#AZURE_VMS" >> ~/.ssh/config
    cat ~/.ssh/azure_vms >> ~/.ssh/config
  fi
  
In Cloud Shell, you can store this script in your home directory, and make it executable with `chmod 755 ssh-host-builder`. You can call this script manually, or when you login (adding to your `.bashrc`). You can also call this script whenever you run an `az vm` command (for example, `create`, `start` or `stop` operations that can mutate the public IP address status of your VM) by adding the following to `.bashrc`:

  PROMPT_COMMAND=__prompt_command
  
  __prompt_command() {
  local EXIT="$?"
  if fc -ln -1 | grep "az vm" > /dev/null 2>&1
  then
     if [ "$EXIT" == "0" ]
     then
       echo "Scheduling a background refresh of your Azure VM IP addresses..."
       ./ssh-host-builder.sh &
     fi
  fi
  }

If you add a `sleep` stanza to the main shell script, you could wait a minute or two before actually running the refresh. In the future, you could use Azure-generated events (like "new IP address allocated") to refresh `.ssh/config`, but it's likely there would still be a polling situation, particularly if you use Cloud Shell.
