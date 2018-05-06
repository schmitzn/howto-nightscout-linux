# How to install Nightscout on Linux
## Table of contents
* [Why yet another tutorial for installing Nightscout?](#introduction)
* [Before we get started](#before-we-get-started)
* [Installation and configuration](#installation-and-configuration)
  * [Install required packages](#install-required-packages)
  * [Configure MongoDB](#configure-mongodb)
  * [Download and install Nightscout](#install-nightscout)
  * [Run Nightscout](#run-nightscout)
  * [Enable SSL support](#enable-ssl)
  * [Create a service](#create-service)

<a name="introduction"></a>
## Why yet another tutorial for installing Nightscout?

When I found out about Nightscout I felt like drowning by the huge amount of info pages out there on the Internet. I read a lot about the existing solutions with [Azure](http://www.azure.com/), [Hiroku](https://www.heroku.com/) or [Vagrant](https://www.vagrantup.com/). The reason I don't use these is that I have been running my own private (web) server for a long time and therefore I wanted to install it on my existing server. Thus I became interested in how to setup Nightscout on Linux. Unfortunately I did not find an article for the setup, so I decided to adapt [this](https://github.com/jaylagorio/Nightscout-on-Windows-Server) installation guide and rewrite it for Linux.

If you are heading for a quick development setup, you might prefer to use a [docker image](https://github.com/nightscout/nightscout-docker).

<a name="before-we-get-started"></a>
## Before we get started

As I mentioned above, this documentation is based on the docs for [Nightscout on Windows Server](https://github.com/jaylagorio/Nightscout-on-Windows-Server). I am going to show you how to install Nightscout on [Arch Linux](https://www.archlinux.org/). I assume that you have installed Linux on your system already.

**For those who are using a different Linux distribution:**

The instructions in the examples may differ from other Linux distributions like Debian/Fedora/OpenSuse. Basically you have to do the same steps but probably with different commands or system tools. I am using the [pacman package manager](https://wiki.archlinux.org/index.php/pacman). In your distribution it may be `apt-get` or `yum`. For service managing I use [systemd](https://wiki.archlinux.org/index.php/Systemd). Your distro might use `init.d`. A detailed comparison of Arch to other Linux distributions can be found [here](https://wiki.archlinux.org/index.php/Arch_compared_to_other_distributions#General).


<a name="installation-and-configuration"></a>
## Installation and configuration

<a name="install-required-packages"></a>
### Install required packages
There are a couple of packages that need to be installed:
- [MonboDB](https://wiki.archlinux.org/index.php/MongoDB) and tools
- [git](https://wiki.archlinux.org/index.php/git)
- [Python](https://wiki.archlinux.org/index.php/python)
- [Node.js](https://wiki.archlinux.org/index.php/Node.js_) and npm
- [GNU C++ Compiler](https://www.archlinux.org/packages/core/i686/gcc/)

```
# pacman -S mongodb mongodb-tools git python nodejs npm gcc
```

<a name="configure-mongodb"></a>
### Configure MongoDB
After installation, start/enable the `mongodb.service` daemon:
```
# systemctl enable mongodb.service
# systemctl start mongodb.service
```
Then create a new database and a new user:
```
$ mongo
> use Nightscout
> db.createUser({user: "username", pwd: "password", roles:["readWrite"]})
> quit()
```

<a name="install-nightscout"></a>
### Download and install Nightscout

Create a folder for your nightscout installation. Then clone the Nightscout repository. If you have your own fork, use the URL to your git repository instead.

```
$ cd your/favorite/path/for/nightscout
$ git clone https://github.com/nightscout/cgm-remote-monitor.git
$ cd cgm-remote-monitor
$ npm install
```

Finally set up the environment variables. There are [several ways](https://wiki.archlinux.org/index.php/environment_variables#Per_user) to do so. However the easiest way is just to create a startup script for your Nightscout server. Create a new file `start.sh` and open it with an editor of your choice:

```
$ nano start.sh
```
Paste those lines (a detailed description of the variables is [here](https://github.com/jaylagorio/Nightscout-on-Windows-Server#installation-and-configuration)):
```
#!/usr/bin/bash

# environment variables
export DISPLAY_UNITS="mg/dl"
export MONGO_CONNECTION="mongodb://username:password@localhost:27017/Nightscout"
export BASE_URL="https://YOUR.WEBPAGE.URL"
export PORT=80
export API_SECRET="YOUR_API_SECRET_HERE"

export PUMP_FIELDS="reservoir battery status"
export DEVICESTATUS_ADVANCED=true
export ENABLE="careportal iob cob openaps pump bwg rawbg basal"

export TIME_FORMAT=24

# start server
node server.js
```

<a name="run-nightscout"></a>
### Run Nightscout

You can now run Nightscout by entering `./start.sh`. If you have used a different way to configure the system variables, just type `node server.js` to start the server.

<a name="enable-ssl"></a>
### Enable SSL support

You can obtain your own trusted SSL certificate at [LetsEncrypt.org](https://letsencrypt.org/).

First of all, you need to install the `https` package for NodeJS:
```
$ npm install https
```

Enable Nightscout's SSL support by just adding the following environment variables to your start script:
```
export SSL_KEY=privkey.pem
export SSL_CERT=fullchain.pem
export SSL_CA=fullchain.pem
```

Please recognize that I've used the fullchain for both `SSL_CERT` and `SSL_CA`. Before that, I tried it with `SSL_CERT=cert.pem` which later led me to an error in xDrip+:

```
Unable to do REST API Download javax.net.ssl.SSLHandshakeException:
java.security.cert.CertPathValidatorException: Trust anchor for certification path not found.
```

Setting the `SSL_CERT` to `fullchain.pem` worked for me.

<a name="create-service"></a>
### Create a service

I recommend using a systemd service which automatically starts Nightscout on system startup. To do so, create ``/etc/systemd/system/nightscout.service`` and paste the following configuration:
```
[Unit]
Description=Nightscout Service      
After=network.target

[Service]
Type=simple
WorkingDirectory=/your/nightscout/path/cgm-remote-monitor
ExecStart=/your/nightscout/path/cgm-remote-monitor/start.sh

[Install]
WantedBy=multi-user.target
```

Reload systemd:
```
# systemctl daemon-reload
```

Start and enable
```
# systemctl enable nightscout.service
# systemctl start nightscout.service
```

Finally check if the service is running:
```
# systenctl status nightscout.service
```
