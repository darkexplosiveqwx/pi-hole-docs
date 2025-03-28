### Why use DNS-Over-HTTPS? [^guide]

DNS-Over-HTTPS is a protocol for performing DNS lookups via the same protocol you use to browse the web securely: **HTTPS**.

With standard DNS, requests are sent in plain-text, with no method to detect tampering or misbehavior. This means that not only can a malicious actor look at all the DNS requests you are making (and therefore what websites you are visiting), they can also tamper with the response and redirect your device to resources in their control (such as a fake login page for internet banking).

DNS-Over-HTTPS prevents this by using standard HTTPS requests to retrieve DNS information. This means that the connection from the device to the DNS server is secure and can not easily be snooped, monitored, tampered with or blocked.
It is worth noting, however, that the upstream DNS-Over-HTTPS provider will still have this ability.

## Configuring DNS-Over-HTTPS

Along with releasing their DNS service [1.1.1.1](https://blog.cloudflare.com/announcing-1111/) (and later [1.1.1.1 for Families](https://blog.cloudflare.com/introducing-1-1-1-1-for-families)) Cloudflare implemented DNS-Over-HTTPS proxy functionality into one of their tools: [`cloudflared`](https://github.com/cloudflare/cloudflared).

In the following sections, we will be covering how to install and configure this tool on `Pi-hole`.

!!! info
    The `cloudflared` binary will also work with other DoH providers (for example, [Google's DoH service](https://developers.google.com/speed/public-dns/docs/doh) or [Quad9's DoH service](https://quad9.net/service/service-addresses-and-features)).

### Installing `cloudflared`

The installation is fairly straightforward, however, be aware of what architecture you are installing on (`amd64` or `arm`).

#### AMD64 architecture (most devices)

Download the installer package, then use `apt-get` to install the package along with any dependencies. Proceed to run the binary with the `-v` flag to check it is all working:

```bash
# For Debian/Ubuntu
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo apt-get install ./cloudflared-linux-amd64.deb
cloudflared -v

# For CentOS/RHEL/Fedora
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-x86_64.rpm
sudo yum install ./cloudflared-linux-x86_64.rpm
cloudflared -v
```

#### armhf architecture (32-bit Raspberry Pi)

Here we are downloading the precompiled binary and copying it to the `/usr/local/bin/` directory to allow execution by the cloudflared user. Proceed to run the binary with the `-v` flag to check it is all working:

```bash
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm
sudo mv -f ./cloudflared-linux-arm /usr/local/bin/cloudflared
sudo chmod +x /usr/local/bin/cloudflared
cloudflared -v
```

!!! info
    Users [have reported](https://github.com/cloudflare/cloudflared/issues/38) that the current version of cloudflared produces a segmentation fault error on Raspberry Pi Zero W, Model 1B and 2B. Currently, there is [no known workaround](https://github.com/pi-hole/docs/issues/710).

#### arm64 architecture (64-bit Raspberry Pi)

```bash
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64
sudo mv -f ./cloudflared-linux-arm64 /usr/local/bin/cloudflared
sudo chmod +x /usr/local/bin/cloudflared
cloudflared -v
```

#### `cloudflared` archive page

You can find all `cloudflared` binary releases on <https://github.com/cloudflare/cloudflared/releases>.

### Configuring `cloudflared` to run on startup

Create a `cloudflared` user to run the daemon:

```bash
sudo useradd -s /usr/sbin/nologin -r -M cloudflared
```

Proceed to create a configuration file for `cloudflared`:

```bash
sudo nano /etc/default/cloudflared
```

Edit configuration file by copying the following in to `/etc/default/cloudflared`. This file contains the command-line options that get passed to cloudflared on startup:

```bash
# Commandline args for cloudflared, using Cloudflare DNS
CLOUDFLARED_OPTS=--port 5053 --upstream https://cloudflare-dns.com/dns-query
```

!!! info
    See the other available [Cloudflare endpoints](https://developers.cloudflare.com/1.1.1.1/infrastructure/network-operators/#available-endpoints).

If you're running cloudflared on different host than pi-hole, you can add listening address to all IPs (for security, change 0.0.0.0 to your machine's IP, e.g. 192.168.1.1):

```bash
# Commandline args for cloudflared, using Cloudflare DNS
CLOUDFLARED_OPTS=--port 5053 --upstream https://1.1.1.1/dns-query --upstream https://1.0.0.1/dns-query --address 0.0.0.0
```

Update the permissions for the configuration file and `cloudflared` binary to allow access for the cloudflared user:

```bash
sudo chown cloudflared:cloudflared /etc/default/cloudflared
sudo chown cloudflared:cloudflared /usr/local/bin/cloudflared
```

Then create the `systemd` script by copying the following into `/etc/systemd/system/cloudflared.service`. This will control the running of the service and allow it to run on startup:

```bash
sudo nano /etc/systemd/system/cloudflared.service
```

```ini
[Unit]
Description=cloudflared DNS over HTTPS proxy
After=syslog.target network-online.target

[Service]
Type=simple
User=cloudflared
EnvironmentFile=/etc/default/cloudflared
ExecStart=/usr/local/bin/cloudflared proxy-dns $CLOUDFLARED_OPTS
Restart=on-failure
RestartSec=10
KillMode=process

[Install]
WantedBy=multi-user.target
```

Enable the `systemd` service to run on startup, then start the service and check its status:

```bash
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
sudo systemctl status cloudflared
```

Now test that it is working! Run the following `dig` command, a response should be returned similar to the one below:

```text
pi@raspberrypi:~ $ dig @127.0.0.1 -p 5053 google.com

; <<>> DiG 9.11.5-P4-5.1-Raspbian <<>> @127.0.0.1 -p 5053 google.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12157
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: 22179adb227cd67b (echoed)
;; QUESTION SECTION:
;google.com.                    IN      A

;; ANSWER SECTION:
google.com.             191     IN      A       172.217.22.14

;; Query time: 0 msec
;; SERVER: 127.0.0.1#5053(127.0.0.1)
;; WHEN: Wed Dec 04 09:29:50 EET 2019
;; MSG SIZE  rcvd: 77
```

### Configuring Pi-hole

Finally, configure Pi-hole to use the local `cloudflared` service as the upstream DNS server by specifying `127.0.0.1#5053` as the Custom DNS (IPv4):

![Screenshot of Pi-hole configuration](../../images/DoHConfig.png)

(don't forget to hit Return or click on `Save`)

### Updating `cloudflared`

The `cloudflared` tool will not receive updates through the package manager. However, you should keep the program update to date. You can either do this manually (e.g. by watching their [repo](https://github.com/cloudflare/cloudflared) for new releases), or via a cron script.

The procedure for updating depends on how you configured the `cloudflared` binary.

#### If you configured cloudflared with your own service files

If you configured `cloudflared` manually (by writing a systemd unit yourself), to update the binary you'll simply redownload the binary from the same link, and repeat the install procedure.

```bash
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm
sudo systemctl stop cloudflared
sudo mv -f ./cloudflared-linux-arm /usr/local/bin/cloudflared
sudo chmod +x /usr/local/bin/cloudflared
sudo systemctl start cloudflared
cloudflared -v
sudo systemctl status cloudflared
```

#### If you configured cloudflared via `service install`

If you configured `cloudflared` using their `service install` command, then you can use the built in update command.

```bash
sudo cloudflared update
sudo systemctl restart cloudflared
```

#### Automating Cloudflared Updates

If you want to have the system update `cloudflared` automatically, simply place the update commands for your configuration method in the
file `/etc/cron.weekly/cloudflared-updater`, and adjust permissions:

```bash
sudo chmod +x /etc/cron.weekly/cloudflared-updater
sudo chown root:root /etc/cron.weekly/cloudflared-updater
```

<!-- markdownlint-disable code-block-style -->
!!! warning
    Make sure to add shebang `#!/bin/bash` in the beginning of  `/etc/cron.weekly/cloudflared-updater`.
    Otherwise, the command will not be executed.
<!-- markdownlint-enable code-block-style -->

The system will now attempt to update the cloudflared binary automatically, once per week.

### Uninstalling `cloudflared`

#### If installed the manual way

*Courtesy of <https://discourse.pi-hole.net/t/uninstall-cloudflare/21459/3>*

```bash
sudo systemctl stop cloudflared
sudo systemctl disable cloudflared
sudo systemctl daemon-reload
sudo deluser cloudflared
sudo rm /etc/default/cloudflared
sudo rm /etc/systemd/system/cloudflared.service
sudo rm /usr/local/bin/cloudflared
```

#### If installed with `cloudflare service install`

```bash
sudo cloudflared service uninstall
sudo systemctl daemon-reload
```

After the above, don't forget to change the DNS back to something else in Pi-hole's DNS settings!

[^guide]: Guide based on [this guide by Ben Dews | bendews.com](https://bendews.com/posts/implement-dns-over-https/)
