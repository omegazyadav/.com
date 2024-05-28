---
title: "Connecting to IRC"
date: 2020-05-01
author: Yadav Lamichhane
description: "Reminder for Myself - Connecting to IRC"
layout: default
comments: true
tags:
- Linux
---
## Internet Relay Chat

## 1. What is it?

Internet Relay Chat abbreviated as IRC which is just another application layer protocol like FTP, SMTP, DHCP, etc. It is mainly used for communicating in the network in the form of text messages. Architecturally IRC is a client/server model which consists of IRC server and IRC clients. IRC server accepts and relays messages to the users who are connected via IRC clients.

There are numerous IRC clients available on the internet which we will be describing shortly. IRC consists of numerous networks and is further divided into channels. Channels are also called rooms where users of particular interest can gather and exchange messages. Each of the channels starts with a hashtag ‘#’.

## 2. Popular clients

Over the years, different IRC clients have emerged claiming to be best with performance and reliability. Some of the popular IRC clients are given below:

1. [Weechat](https://weechat.org/)

1. [Pidgin](https://pidgin.im/)

1. [Irssi](https://irssi.org/)

1. [HexChat](https://hexchat.github.io/)

## 3. Installation

For the demonstration purpose, I am using the Weechat client for connecting to the different IRC networks. Weechat is a command-line IRC client which is designed to be light and fast. It is open-source software that is released under the terms of the GNU General Public License.

Installation is straightforward since every Linux distribution has packaged it in its standard package manager. Steps to install weechat in different package managers are:-

Ubuntu

    apt-get update && apt-get install -y weechat

Fedora

    dnf update && dnf install weechat

CentOS

    yum update && yum install weechat

Arch

    pacman -Syy && pacman -S weechat

Now the weechat is installed and you can run it with the binary weechat via your standard terminal.

## 4. Connecting to the IRC Networks

### **OFTC Network**

**OFTC** refers to Open and Free Technology Community which aims to provide stable and effective collaboration services to members of the community. It was founded in 2001 by a group of Open Source and Free Software Communities providing better communication, development, and support infrastructure.

Add the **OFTC** Server to your weechat configuration
```
/server add oftc irc.oftc.net/6697 -ssl --autoconnect
```
Here 6697 is a port number for the SSL connection. You can also use other ports i.e. 6667,6668 for establishing non-SSL connection.

Connect to the OFTC network
```
/connect oftc
```
After this, you may join the different chat rooms available over the internet. For e.g. To join the Linux Kernel Newbies community, simply follow the below-given step.
```
/join #kernelnewbies
```
![Linux Kernel Newbies Channel](https://cdn-images-1.medium.com/max/3804/1*SlKtQhpgbhoROer8m4Rrhw.png)
|:--:|
| Kernel Newbies IRC Channel |

### Libera Network

Libera.Chat is a leading IRC Network, providing a community platform for free and open-source software and peer-directed projects.

Add an IRC server with /server command
```
/server add libera irc.libera.chat/6697 -ssl
```
Set the Username and Real Name
```
/set irc.server.libera.username "omegazyadav"
/set irc.server.libera.realname "Yadav Lamichhane"
```
Enable auto-connect to the server at startup
```
/set irc.server.libera.autoconnect on
```
Auto join some channels when connecting to the server
```
/set irc.server.libera.autojoin "#devops"
```
Connecting to IRC Server
```
/connect libera
```

With this command, WeeChat connects to the libera server and auto-joins the #devopschannels configured in the autojoinserver option.

![DevOps Channel in Libera Network](https://cdn-images-1.medium.com/max/3832/1*PDIj7RwUYyQzIcNgk4UU_g.png)
|:--:|
|*DevOps Channel in Libera Network*|

## 5. Connecting through the SSH Tunnel

Sometimes your Internet Service Provider blocks your connection to the IRC server because of security concerns. What I meant is IRC channel exposes your IP address to all people connected to the network and this can be a source of vulnerability for the ISP. In this case, you can only connect to the IRC via SSH tunnel.

For this setup, you need to have Virtual Private Server so that you can tunnel your connection to the VPS and bypass the it with IP address of your VPS. One of the requirements for this is your VPS server should be running SSH daemon and your local machine should have SSH client to create SSH tunnel.

### Creating ssh tunnel

```
ssh -L localhost:6667:[IRC-HOST]:6667 [USER]@[HOST] -N -v
```

For connecting the OFTC network through SSH tunnel.

```
ssh -L localhost:6667:irc.oftc.net:6667 [USER]@[HOST] -N -v
```

Now we have successfully created the SSH tunnel and ready to establish the connection. Open the weechat and connect to the respective network.

```
/server add oftc localhost
```

Localhost is now connected to the IP address of VPS where SSH tunnel is set up.

```
/connect localhost
```

Voila! Now you are connected to the IRC server and you can begin to interact with your beloved communities around the world.

Hope the instructions are pretty clear and able to follow through.

Thank you!
