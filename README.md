# Deploy modern infrastructure to SL using Cuttle

We will use cuttle to deploy two distinct environments into softlayer.  

The first will be the bastion environment consisting of a bastion and a console log server,
the second will be the rest of the infrastructure that will be accessed strictly via the bastion.


## Prepare local environment

Set up local environment by cloning down both the environment and cuttle git repos:

```bash
mkdir demo
$ git clone ... environment
$ git clone cuttle cuttle
$ mkvirtualenv cuttle-demo
$ pip install -r environment/requirements.txt
```

Set up a softlayer config file:

```bash
$ slcli config setup
Username: XXXXXX
API Key or Password: XXXXXXXXXXXXXXXXXXXXX
Endpoint (public|private|custom) [public]:
Timeout [0]:
:..............:..................................................................:
:         name : value                                                            :
:..............:..................................................................:
:     Username : XXXXXX                                                           :
:      API Key : XXXXXXXXXXXXXXXXXXXXX                                            :
: Endpoint URL : https://api.softlayer.com/xmlrpc/v3.1/                           :
:      Timeout : not set                                                          :
:..............:..................................................................:
Are you sure you want to write settings to "/home/XXXX/.softlayer"? [Y/n]: Y
```

Create a SSH key to use:

```bash
$ ssh-keygen -f ~/.ssh/cuttle-demo -P ""
Generating public/private rsa key pair.
Overwrite (y/n)? y
Your identification has been saved in /home/XXXX/.ssh/cuttle-demo.
Your public key has been saved in /home/XXXX/.ssh/cuttle-demo.pub.
slcli sshkey add -f ~/.ssh/cuttle-demo.pub cuttle-demo
slcli sshkey list                                     
:........:....................:.................................................:.......:
:   id   :       label        :                   fingerprint                   : notes :
:........:....................:.................................................:.......:
: 926125 :    cuttle-demo     : 2c:ba:f4:8d:0c:4f:35:3d:fb:98:0b:30:b3:2e:a4:09 :   -   :
:........:....................:.................................................:.......:
```

> Set the `sl_ssh_key` variable in `environment/playbooks/create_vms.yml` with the id
  from the output of the `slcli sshkey list` command.

Pick a datacenter and a VLAN to deploy to:

```
slcli vlan list
:.........:........:......:..........:............:..........:.................:............:
:    id   : number : name : firewall : datacenter : hardware : virtual_servers : public_ips :
:.........:........:......:..........:............:..........:.................:............:
: 2073387 :  866   :  -   :    No    :   dal09    :    0     :        1        :     61     :
: 2073385 :  812   :  -   :    No    :   dal09    :    0     :        1        :     13     :
```

> Set the `sl_datacenter` and `sl_vlan` variables in `environment/playbooks/create_vms.yml`
  using the datacenter name (ex dal09) and vlan id (ex 2073387). Unless you're very familiar
  with your softlayer account you may need to use the softlayer
  web portal to pick a VLAN to use.  If you skip this step SL will pick a random vlan and you
  may not be able to communicate on the backend network.

## Deploy Bastion VMs

Run the playbook to create the bastion and ttyspy servers:


```bash
$ ansible-playbook playbooks/create_bastion_vms.yml
PLAY [Build Bastions] ******************************************************************************************************************************************************************************************

TASK [Build Bastion server] ************************************************************************************************************************************************************************************
changed: [localhost] => (item=bastion01)
...
```

This step will take a while.  You can chill for a bit and run the following command
to check if its completed.

> When `action` is `-` for all hosts you should be able to access them via ssh.

```
$ slcli vs list --tag cuttle-demo
:..........:...........:...............:..............:............:..............:
:    id    :  hostname :   primary_ip  :  backend_ip  : datacenter :    action    :
:..........:...........:...............:..............:............:..............:
: 36592565 : bastion01 : xxxx          : XXXXXXXX     :   dal09    : -            :
:..........:...........:...............:..............:............:..............:
$ ssh -i ~/.ssh/cuttle-demo root@xxxx
Last login: Sat Jul 22 21:46:46 2017 from 10.0.80.186
root@bastion01:~# ps aux | grep php
root      1539  0.0  0.0  14992   928 pts/0    S+   21:50   0:00 grep --color=auto php

```

Run the following command to ensure that the machines have python installed
and create a ssh_config file that you can use later:

```bash
$ ansible-playbook -i bastion/hosts playbooks/bastion-ssh_config.yml \
    -u root -e "etc_hosts=~"
```

> Note: If SSH gives you any errors here its possibly because you have conflicting host keys.
  Try to SSH to the offending IP and fix any conflicts until you can succesfully SSH into
  each host. the `-e` is overriding the use of a custom filter that isn't available on
  a standard ansible run.


## Configure Bastion VMs

Change into the `cuttle` directory we cloned earlier and run the `ursula`
command to deploy our `bastion` and `ttyspy` servers.

```bash
$ ursula ../cuttle-infra-on-sl/cuttle-env-bastion site.yml \
    --limit cuttle-env-bastion --ursula-user root
```

## Deploy Cuttle Example Environment

Run the playbook to create the infrastructure servers:


```bash
$ ansible-playbook playbooks/create_infra_vms.yml
PLAY [Build ...] ******************************************************************************************************************************************************************************************

TASK [Build ... server] ************************************************************************************************************************************************************************************
changed: [localhost] => (item=...)
...
```

Run the base bootstrap playbook:

```bash
$ ansible-playbook -u root -i cuttle-env-infra/hosts playbooks/bootstrap-infra.yml -e "bastion=xxxxx"
PLAY [Build ...] ******************************************************************************************************************************************************************************************

TASK [Build ... server] ************************************************************************************************************************************************************************************
changed: [localhost] => (item=...)
...
```

Run just the common role to kill off our remote root:

```
ursula --ursula-user=root ../cuttle-infra-on-sl/cuttle-env-infra site.yml --tags=common --limit cuttle-env-infra
```
