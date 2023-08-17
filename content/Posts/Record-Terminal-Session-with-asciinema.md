---
title: "Record Terminal Session with asciinema"
date: 2020-07-17
author: Yadav Lamichhane
description: "Record Terminal Session with asciinema"
tags:
- Linux
---
![Imgur](https://i.imgur.com/5PKtA80.png)

## ASCIINEMA

ASCIINEMA is a terminal session recorder that captures all the input and output that is being processed by the terminal. It is a lightweight text-based terminal recorder in which you can copy the terminal text and paste it elsewhere. asciinema lets you easily record terminal sessions and replay them in a terminal as well as in a web browser.

The installation procedure and usage are straight forward which won’t be difficult to follow along. Furthermore, you can integrate asciinema with a terminal multiplexer(tmux) to record multiple terminal sessions within single recordings.

## Installation

There are many alternatives for the installation of asciinema within your system.

### 1. Installation with pip
```
sudo pip3 install asciinema
```

### 2. Installation on Linux

Arch Linux
```
pacman -S asciinema
```
Debian/Ubuntu
```
sudo apt-get install asciinema
```
Fedora
```
sudo dnf install asciinema
```
Geneto Linux
```
emerge -av asciinema
```
openSUSE
```
zypper in asciinema
```
### 3. Installation on macOS

Homebrew
```
brew install asciinema
```
MacPorts
```
sudo port selfupdate && sudo port install asciinema
```
Nix
```
nix-env -i asciinema
```
### 4. Running on Docker Container

Pull the official image for asciinema
```
docker pull asciinema/asciinema
```
Run the container
```
docker run --rm -ti -v $HOME/.config/asciinema:/root/.config/asciinema asciinema/asciinema rec
```
Where,

t=pseudo TTY

i= Interactive session kept for the Standard input

v=Mounting the container config path to the host system

## Usage

Record the terminal session, once you are done with the recordings then hit CTRL+D or type exit to close
```
asciinema rec your_file_name
```
Play the recorded terminal session
```
asciinema play -s 2 your_file_name
```
Upload to the asciinema server

To upload to the server, you need to authenticate your account
```
asciinema auth
```
Copy the authentication token and open the link in order to add the token into your asciinema account.

Finally, upload the recorded terminal session
```
asciinema upload your_file_name
```
## asciinema with tmux (Terminal Multiplexer)

We can use asciinema with tmux for recording multiple terminal sessions. Follow the given steps to implement the asciinema with tmux.

1. Start the tmux session
```
tmux new -s omega
```
2. Create multiple windows

    For Vertical window press [ctrl + b] + %

    For horizontal window, press [ctrl+b] + "

3. Detach the current tmux session with [ctrl + b ] + d

4. Attach the tmux session along with asciinema
```
asciinema rec -c "tmux attach -t omega"
```
After the recording is completed, detach the tmux session and save the recording.

## **Preview**
[![asciicast](https://asciinema.org/a/349424.svg)](https://asciinema.org/a/349424)
