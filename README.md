# rak3s - 5 less than rak8s

(pronounced rackees - /ˈrækiːs/)

Stand up a Raspberry Pi based k3s Kubernetes cluster with Ansible.

Inspired by the work done on [rak8s](https://github.com/rak8s/rak8s) and [k3s-ansible](https://github.com/itwars/k3s-ansible).

The latest versions of this has moved all the heavy lifting to [k3sup](https://github.com/alexellis/k3sup). Install this on your local machine first.

- [rak3s - 5 less than rak8s](#rak3s---5-less-than-rak8s)
  - [Why](#why)
  - [FYI](#fyi)
  - [Preflight](#preflight)
    - [Install k3sup](#install-k3sup)
    - [Modify for your environment](#modify-for-your-environment)
    - [DNS](#dns)
    - [Copy ssh private key](#copy-ssh-private-key)
    - [Change default pass](#change-default-pass)
    - [Apt update](#apt-update)
  - [Install](#install)
    - [Get kube config](#get-kube-config)
  - [Add more nodes](#add-more-nodes)
    - [Add worker nodes](#add-worker-nodes)
    - [Add master nodes](#add-master-nodes)
  - [Uninstall](#uninstall)
    - [Complete Uninstall](#complete-uninstall)
    - [Remove a single node](#remove-a-single-node)

## Why

- k3s-ansible targets multiple platforms, this is only for Raspberry Pis
- rak8s used Kubernetes, this uses k3s
- Neither support high availability deployments, this does

## FYI

My setup is currently all pi 4 with 4 gb ram

As of version v1.17.2+k3s1 you may see an error while installing an HA setup, you can ignore it. The playbook will pause for one minute then check that the service is running ok. If you are able to see all the nodes after the playbook finishes then all is well.

## Preflight

Each of these should be run once for initial setup. Can be run as needed later.

### Install k3sup

Install k3sup using brew or [these instructions](https://github.com/alexellis/k3sup#download-k3sup-tldr)

### Modify for your environment

Modify the hosts.ini file to suit your environment. Change the names to your liking and the IPs to the addresses of your Raspberry Pis.

If your SSH user on the Raspberry Pis is not the Raspbian default (pi) modify remote_user in the ansible.cfg.

In group_vars/all.yaml you can specify [k3sup extra args](https://github.com/alexellis/k3sup#usage-) and [k3s extra args](https://rancher.com/docs/k3s/latest/en/installation/install-options/#registration-options-for-the-k3s-server) for the server install script. Check out the commented out examples of the extra args I use.

### DNS

Set up dns names and static ips in your dns/dhcp server. They should match the hosts.ini, so if you use something other than the default make sure to update hosts.ini with the correct values.

### Copy ssh private key

runs ssh-keyscan and ssh-copy-id for each ip in hosts.ini that is not commented out. You will have to type in default password "raspberry" (unless you changed it) for each.

    touch ~/.ssh/known_hosts
    grep -v "(;|#)" hosts.ini | grep -E -o "([0-9]{1,3}[\.]){3}[0-9]{1,3}" | xargs -I{} ssh-keyscan -H {} >> ~/.ssh/known_hosts
    grep -v "(;|#)" hosts.ini | grep -E -o "([0-9]{1,3}[\.]){3}[0-9]{1,3}" | xargs -I{} ssh-copy-id pi@{}

### Change default pass

Will prompt for new password for pi user. If left blank a random password will be generated.

Password will be printed in final debug message (make sure no h@ck3rs are looking over your shoulder). It is not saved anywhere so if you allowed a random password to be generated make sure to save in a password manager!

    ansible-playbook auth.yaml

### Apt update

Runs apt update and upgrade on all machines. Can take a while.

    ansible-playbook apt-update.yaml

## Install

    ansible-playbook --ask-become-pass install.yaml

This will ask for the sudo password (required by k3sup) for your local machine.

If you want an HA setup with multiple masters you must go into hosts.ini and uncomment the extra master nodes.

Make sure you have an odd number of masters!

You may want to put a load balancer like HA proxy infront of your api servers on the master nodes. If so you should edit the k3s_server_extra_args in group_vars/all.yaml and add the --tls-san arg with the ip/hostname of the haproxy, then update your kubeconfig to use this ip/hostname.

### Get kube config

The kubeconfig file is saved automatically on install.
The default path is ~/.kube/config. If this file exists the rak3s config will be merged in, you must change the context manually.

If you need to redownload the kubeconfig you can run the following at any time.

    ansible-playbook kubeconfig.yaml

## Add more nodes

### Add worker nodes

You can add more worker nodes at any time. Just add the correct values in hosts.ini and rerun the install script. Existing masters and workers will not be altered.

### Add master nodes

If you started with a single master you can NOT add new ones. You must uninstall and reinstall with multiple masters from the begining.

If you started with multiple masters and you want to add more you can add them to the hosts.ini and rerun the install script. Existing masters and workers will not be altered. The only gotcha is you can't place a new master as the first item in the hosts.ini masters section. You must have a working good master as the first item because this is where ansible will look for the token to add new nodes to the cluster.

Make sure you have an odd number of masters!

## Uninstall

### Complete Uninstall

Completely destroys the cluster!

    ansible-playbook uninstall.yaml

### Remove a single node

    kubectl delete node rak3s-worker-X
    ansible-playbook -l rak3s-worker-X  uninstall.yaml
