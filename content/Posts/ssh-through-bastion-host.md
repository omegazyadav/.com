---
title: "SSH into the private server through Bastion Host"
date: 2021-02-21
author: Yadav Lamichhane
description: "SSH into the private server through Bastion Host"
tags:
- Linux
---

# SSH into the private server through Bastion Host

![Connecting private server with bastion host](https://i.imgur.com/1mZuxec.png)
|:--:|
| Connecting private server with bastion host |

A bastion host is a publicly facing server that acts as an entry-point to the system which is protected from the high-end firewall or located in a private server. These servers can only be accessible from the bastion hosts so this would reduce the attack surface area from the outside world.

In this post, I will be explaining ways to ssh into the private server i.e. ec2 instance through the bastion host.

### **Agent Forwarding**

The ssh-agent is a helper program that keeps track of user's identity keys and their passphrases. The agent can then use the keys to log into other servers without having the user type in a password or passphrase again. This will temporarily store the ssh keys in an in-memory state and forwards the keys to the bastion host so that we can log into the remote server without actually need of ssh keys.

To set up the ssh-agent we need the below-mentioned procedures.

* Start the ssh-agent
```
    $ eval $(ssh-agent)
```
* Add ssh keys to the ssh-agent
```
    ssh-add /path/to/the/ssh/keys.pem
```
* Forward the ssh keys to the bastion host
```
    ssh -A user@bastion-ip
```
Here the -A flag forwards the ssh keys into the bastion host which we can verify with ssh-add -l after successful log into the bastion host.

* And finally, log into the remote host
```
    ssh user@remote-ip
```
### **Proxy Jump**

Connect to the target host by first making an ssh connection to the jump host described by destination and then establishing a TCP forwarding to the private IP of the destination server.
```
    ssh -J [user@bastion_ip] [user@Destination_IP]
```
We can also specify the server ports while connecting through the bastion host.
```
    ssh -J [user@Bastion_IP:Port] [user@Destination_IP:Port]
```
As per the documentation given in the manual pages for ssh i.e. [[man ssh]](https://man7.org/linux/man-pages/man1/ssh.1.html) we can also provide multiple bastion hosts to make ssh connections into the remote server.

    ssh -J [user1@bastion1],[user2@bastion2] [user@destination]

For one time solution, the above configuration can be fine but if in case we need to login into the remote server multiple times a day then the above method won’t be feasible. We can hard code the above procedure into the ~/.ssh/config file which eases you to log into the remote server.

    ### Bastion Host
    Host bastion-host
      HostName [Bastion IP]

    ### Remote Host
    Host remote-host
      HostName [Remote IP]
      ProxyJump bastion-hostname

Once this configuration is set into the ~/.ssh/config then you can directly ssh into the remote server.

    ssh remote-host

For ssh into the ec2 instance, we may require the ssh credentials i.e. *.pem file to log into the remote server. We can simply specify the path of the credentials in above mention config.

    ## Bastion Host
    Host bastion-host
      HostName [Bastion IP]
      IdentityFile [Path to pem file]
      User ubuntu

    ## Remote Host
    Host remote-host
      HostName [Remote IP]
      User ubuntu
      ProxyJump bastion-host

### **Proxy Command**

Similar to the Proxy Jump, proxy command ssh into the remote server by forwarding stdin and stdout through a secure connection from bastion-host.

    ## Bastion Host
    Host bastion-host
      HostName [Bastion IP]
      IdentityFile [Path to pem file]
      User ubuntu

    ## Remote Host
    Host remote-host
      HostName [Remote IP]
      User ubuntu
      ProxyCommand ssh -q -W %h:%p bastion-host

SSH is a powerful tool and consists bunch of features. Check its official site [here](https://www.ssh.com/ssh/) to find out more information on it. For configuration-related information, you can always refer to the [man page](https://man7.org/linux/man-pages/man1/ssh.1.html) which literally consists of hundreds of config and flags which can help you to meet your requirements.
