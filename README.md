# Ansible with virtualbox example
This repo is an example of how to set up, configure, and run ansible to control hosts with virtualbox installed on them.  For the most part the yamls are relatively simple, the configuration/networking was the most difficult part of this endeavor because I was using windows, the deployment should be a lot easier on a linux host.  Of note, ansible does not have a module for virtualbox, so I utilized the shell modules availible for the os I was using to execute what I needed.  

## Setup for Windows
As mentioned above, this is the most involved part of this process.
Essentially, I wanted to make a network similar to what is pictured below to emulate managing a remote host with virtualbox installed.
![alt text](https://github.com/ngdoc/ansible_with_virtualbox/blob/main/net_Diagram.png?raw=true)

First I installed virtualbox on windows using the gui from https://www.virtualbox.org/wiki/Downloads. Virtualbox comes with its own set of command line instructions called VBoxManage, but this does not automatically get added to my path so I added it to the path by identifying the folder that contained vboxmanage.exe (C:\Program Files\Oracle\VirtualBox\VBoxManage.exe) and adding it to the path environment variable by going to "edit environment variables" -> environment variables -> clicking path in the user environment variables -> pasting the path to the directory with vboxmanage.exe -> hit okay -> repeating for system variables -> press ok -> press ok

Then I installed windows native ssh server.  Ssh is needed to allow ansible to connect and manage remote machines. ansible with windows also supports winrm, but i've found it to be buggy so I used ssh. Ssh can be installed by going to settings -> system -> search optional features -> view features -> search openssh server -> next -> install.  Once installed, the service needs to be turned on by going to services in the windows search bar -> find and click openssh server -> click start

Following that I needed to install windows subsystem for linux (wsl) because ansible does not support running from windows.  I did this by opening an elevated command prompt and running:
 ``` wsl --install ```
to start wsl simply run 
``` wsl ```
and it will give you a command line into a segmented linux environment.  

Now I needed to change firewall settings to allow wsl to connect to the windows machine, I did this by following these steps to disable the firewall blocking wsl traffic:
https://superuser.com/questions/1714002/wsl2-connect-to-host-without-disabling-the-windows-firewall
If using an actual remote machine, it would be better to add a firewall rule allowing ssh traffic for better security by going to windows defender firewall -> advanced settings -> inbound rules -> new rule -> port -> next -> leave TCP checked, add 22 to the "specific local port" field-> hit next-> leave "allow the connection" as checked -> hit next -> leave all options check -> hit next ->  name the rule anything -> hit finish.
Note: when openssh is installed it puts a rule on the firewall to allow ssh traffic, but i've run into issues with it so I add a more permissive one as shown above.

Now its time to install an ssh client in wsl to allow ansible to ssh to remote machines.  This can be done by running: 
``` apt install sshpass ```

Next we need to install python in the wsl environment because it is a requirement for ansible.  This can be done by running 
``` apt install python ``` 
in the wsl environment.  Note that if the command fails, its most likely versioning, look at the output of the command and it should give availible versions, select the newest.

Sometimes pip will not install automatically so it can be neccessary to run:
``` apt install pip ```

Finally its time to install ansible with:
``` pip install ansible ```



## Setup for Linux
I did my setup on Windows using wsl, so most of the commands should remain the same but there may be some differences in networking or commands depending on the linux distribution.
note: the package manager may differ, I'll be using apt.  However, using a different package manager will work as well, just make sure to use pip for the install of ansible.

Assuming virtualbox is already installed:

Virtualbox comes with its own set of command line instructions called VBoxManage, but this does not automatically get added to path so add it to the path by identifying the folder that contained vboxmanage.exe and running:
```
export PATH=$PATH:<path to vboxmanage>
```

Next you will want to install an ssh client by running:
``` apt install sshpass ```

Next we need to install python because it is a requirement for ansible.  This can be done by running 
``` apt install python ``` 
Note that if the command fails, its most likely versioning, look at the output of the command and it should give availible versions, select the newest.

Sometimes pip will not install automatically so it can be neccessary to run:
``` apt install pip ```

Finally its time to install ansible with:
``` pip install ansible ```

## inventory file
inventory.ini is the file ansible uses to track and manage clients along with how to interact with them.
My inventory.ini file is composed of:
```
[win]
<ip of the computer you want to execute on>

[win:vars]
ansible_user=<username>
ansible_password=<password>
ansible_connection=ssh
remote_tmp = <a directory that ssh user has access too>
become_method = runas
ansible_shell_type = cmd
shell_type = cmd 
```
[win] denotes the name of a group, this section is used to list the ips and hostnames of clients.
[win:vars] are environment variables used when the win group is being targeted.  I hard coded my username and password, but its best to use -u and --ask-pass options in the command calling ansible (ill talk about that later). connection type is set to ssh, but can be changed if using winrm.  remote_tmp is the directory ansible will use for temporary storage on the remote host.  become is used if the command being used on the remote machine requires elevated permissions, it shouldnt be important in this use case. The shell types are important, ansible runs startup scripts during initial connection and if the shell type is wrong, the startup scripts could run in the incorrect language.  To know which shell type to use, ssh to the remote machine and see what the shell defaults to.  This option supports cmd, powershell, and bash, the default is bash.

## ansible.cfg
ansible.cfg is used for configuration of the ansible instance. I didn't need it for this use case, but it can be used for more functionality as an instance expands its reach.

## playbooks
Playbooks are yaml files used by ansible to interact with the remote host machines.  They are comprised of:
Name: name of the playbook
hosts: the group name of the hosts being targeted for the playbook 
tasks: the tasks being executed on the target
I also set the the gather_facts flag to false to try to un-complicate this instance's playbook runs.

The tasks listed in the playbook are very wide ranging, but generally are composed of a name and a module, in my case the module is ansible.builtin.win_shell to denote that I am interacting with a shell on a windows machine.  Normally, applications will have a specific ansible module to make interaction with them easy, but virtualbox does not which is why I chose to use the shell. 

## commands to run the playbooks
To run the playbooks, I used the format below:
```
ansible-playbook <playbook name> -i < inventory file name > -vvv
```

So if I wanted to run the delete playbook I would run:
```
ansible-playbook delete_vbox.yaml -i inventory.ini -vvv
```
-i is the inventory option
-vvv means that output should be verbost

additionally, ansible can support other options including:
-u username for ssh connection
--ask-pass prompt for the password for the ssh connection

## commands used by playbooks to interact with virtualbox
The commands used by the playbooks to interact with virtualbox come from virtualbox's native cli vboxmanage.  To use it, just ensure it is added to the Path variable. Here is the documentation for the command: https://docs.oracle.com/en/virtualization/virtualbox/6.0/user/vboxmanage-cmd-overview.html

### create_vbox.yaml
Create and register a vm in virtualbox
```
vboxmanage createvm --name <name of machine> --basefolder "C:\Users\ngdoc\VirtualBox VMs" --ostype "Oracle Linux (64-bit)" --register
```
Attach storage to a vm to prep for attaching the ise
```
VBoxManage storagectl "<name of machine>" --name IDE --add ide 
```
Add memory to the machine to make it faster
```
VBoxManage modifyvm "<name of machine>" --memory 2048
```
attached the iso to the machine
```
VBoxManage storageattach "<name of machine>" --storagectl IDE --port 0 --device 0 --type dvddrive --medium 
"C:\Users\ngdoc\Downloads\NoblePup32-24.04-240406.iso"
```
start the vm 
```
vboxmanage startvm <name of machine> --type gui
```

### delete_vbox.yaml
delete the vm
```
VBoxManage unregistervm <name of machine> --delete
```
### list_vbox.yaml
list all the vms in the virtual box instance
```
vboxmanage list vms
```
### reboot_vbox.yaml
reboot a vm in virtual box
```
VBoxManage controlvm <name of machine> reset
```
### start_vbox.yaml
start a vm in virtualbox
```
vboxmanage startvm <name of machine> --type gui
```
### stop_vbox.yaml
stop a virtual machine
```
VBoxManage controlvm "<name of machine>" poweroff
```
## how to extend the functionality of this example
I built this repo as a quick example of how to use ansible to interact with virtualbox, so there definitely a couple things that could be added to make this work in an actual full lab or production environment. Below are some options that I see that could extend the value of this example.

### substitue in variables for hard coded values
I hardcoded machine names and other information into the playbooks, if this is removed and replaced with a variable these playbooks could could interact with many machines at once and use different isos, OS's, memory size, etc.
### auto add vm names to a list for use
Keeping a list of vm names would be exceptionally helpful with interacting with multiple machines at once, this could be done by executing commands on localhost rather than the remote machine to create a file or variable thats a list of vms.
### auto build the inventory with vms in virtual box
Currently, these playbooks only interact with the administrative tasks associated with creating, rebooting, deleting, etc vms.  This could be extended by adding another group to the inventory file and appending vm ip addresses or hostname which would allow administration/configuration within the vms themselves.


