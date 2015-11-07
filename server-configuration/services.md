#Virtualbox Basic Services for an Ubuntu Local Development Machine on OS X

This article will walk you through the installation and configuration of the basic services needed for a LAMP-based development server.

Before installing apt packages, first run an update:
`sudo apt-get update`

###OpenSSH and cURL
`sudo apt-get install -y openssh-server curl`

###Ruby and RVM

```bash

`gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3`
`\curl -sSL https://get.rvm.io | bash -s stable`
`sudo apt-get install -y ruby`
`source /home/[USER_NAME]]/.rvm/scripts/rvm`
```

###Linuxbrew
You heard that right. Linuxbrew!!!

```bash
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/linuxbrew/go/install)"
```

Add Linuxbrew to your path in .bashrc or .zshrc

```bash
export PATH="/home/[USER_NAME]/.linuxbrew/bin:$PATH" # Linuxbrew
```

###Git, Apache, PHP, MySQL
brew install curl git php_54 mysql

Linuxbrew alternative
If you do not want to use Linuxbrew, you can use apt, the standard package manager for Ubuntu

`sudo apt-get install -y curl git lamp-server`
