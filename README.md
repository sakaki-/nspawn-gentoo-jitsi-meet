# nspawn-gentoo-jitsi-meet

Bootable systemd-nspawn Gentoo rootfs (OpenRC and systemd) containing a prebuilt Jitsi Meet server set.

## Introduction

[Jitsi](https://jitsi.org) is a set of open-source components that allow you to easily build and deploy secure 
videoconferencing solutions. 

<img src="https://jitsi.org/wp-content/uploads/2018/08/brady-bunch-stand-up-1024x632.jpg" alt="[Jitsi Meet Screenshot (from jitsi.org)]" title="[Image credit: jitsi.org]" width="400px" align="right"/>

This project provides two **pre-built root filesystems** ("rootfs"), each containing **the baseline Jitsi Meet server set (v4548) on Gentoo**, under **OpenRC** and **systemd** init respectively. The rootfs have been created so as to be bootable as containers under `systemd-nspawn` (in which they have their [own process namespace](https://blog.selectel.com/systemd-containers-introduction-systemd-nspawn/) and pid 1 init).

The purpose of *this*, in turn, is to allow a Gentoo Jitsi Meet server instance to easily be deployed, without compilation, on e.g., a [VPS](https://en.wikipedia.org/wiki/Virtual_private_server) running a non-Gentoo base OS, such as e.g. Debian or Ubuntu, while retaining the flexibility to customize elements of the install as desired later (up to and including a full `emerge -e --with-bdeps=y @world`).

> The ebuilds on which these rootfs are based may be found in my [`sakaki-tools` overlay](https://github.com/sakaki-/sakaki-tools), (mostly) [here](https://github.com/sakaki-/sakaki-tools/tree/master/net-im).

Both instances (which are identical in function) contain an installed but not-yet-configured Jitsi Meet server set. The configuration process (described [below](#config)) uses a straightforward prompt-driven system, and follows the logic of the upstream `debconf` /  `postinst` flow.

By default, automatic activation (and upkeep) of a (free) [Let's Encrypt](https://letsencrypt.org/) TLS certificate for your server is supported, as is the mandatory use of a password to validate new videoconference sessions.

The container rootfs may be downloaded from the appropriate link below (or via `wget`, per the instructions [that follow](#downloadboot)).

<a id="downloadlinks"></a>Variant | Version | Image | Digital Signature
:--- | ---: | ---: | ---:
Gentoo Jitsi Meet server rootfs (OpenRC) | v1.0.0 | [gentoo-jitsi-openrc.tar.xz](https://github.com/sakaki-/nspawn-gentoo-jitsi-meet/releases/download/v1.0.0/gentoo-jitsi-openrc.tar.xz) | [gentoo-jitsi-openrc.tar.xz.asc](https://github.com/sakaki-/nspawn-gentoo-jitsi-meet/releases/download/v1.0.0/gentoo-jitsi-openrc.tar.xz)
Gentoo Jitsi Meet server rootfs (systemd) | v1.0.0 | [gentoo-jitsi-systemd.tar.xz](https://github.com/sakaki-/nspawn-gentoo-jitsi-meet/releases/download/v1.0.0/gentoo-jitsi-systemd.tar.xz) | [gentoo-jitsi-systemd.tar.xz.asc](https://github.com/sakaki-/nspawn-gentoo-jitsi-meet/releases/download/v1.0.0/gentoo-jitsi-systemd.tar.xz.asc)

The underlying Jitsi Meet version is [v4548](https://github.com/jitsi/jitsi-meet/releases/tag/stable%2Fjitsi-meet_4548) (stable, 1 May 2020).

## Table of Contents

- [Prerequisites](#prereq)
- [Downloading and Booting the Container rootfs](#downloadboot)
  * [OpenRC Container (rootfs)](#openrc)
  * [systemd Container (rootfs)](#systemd)
- [Jitsi Configuration and Bring-Up](#config)
- [Starting Jitsi Automatically](#startonboot)
- [Feedback and Bugs](#feedback)

## <a id="prereq"></a>Prerequisites

It is assumed that you have:

* An x86_64 target host (whether a VPS, dedicated server or otherwise), with:
  * a mainstream, systemd-based Linux OS (Debian, Ubuntu etc.)
  * \>=2GiB of RAM;
  * \>=1 dedicated IP address;
  * no current web, prosody, turn, videobridge or jicofo server running; and
  * root access.
* A registered, fully qualified domain name (FQDN) with a DNS 'A' record pointing to the above host's IP address
  * You should be able to externally `ping` the FQDN from an arbitrary machine.

Also, before commencing, ensure the following inbound IP ports are open on your target host:

* 80/tcp (for Let's Encrypt challenges; other traffic will be auto-redirected to https)
* 443/tcp (to serve the Jitsi Meet web app to clients; BOSH; also muxed TURN/TLS on nginx)
* 4445/tcp (TURN/TLS)
* 4446/udp (STUN)
* 10000/udp (videobridge SFU)

> Consult your VPS help for how to do this. New VPS instances often have *no* ports blocked by default - in which case you will of course meet the criteria above, but will probably want to tighten things down prior to production release (please make sure, when editing firewall settings, not to lock yourself out from `ssh` access; this is most commonly on port 22/tcp, but the setup on your VPS may differ).

## <a id="downloadboot"></a>Downloading and Booting the Container rootfs

Begin by logging into your target system as root. Then, ensure that `systemd-nspawn`, `bsdtar` and `wget` are installed. The following is appropriate for (modern) Ubuntu and Debian, adapt as necessar for Fedora etc:

```console
vps ~ # apt-get update && apt-get install -y systemd-container bsdtar wget
```

Next, choose the init system you prefer, and then continue with either the [OpenRC](#openrc) or [systemd](#systemd) sections below.

> Both deployments have identical functionality, so choose the init system you are more comfortable working with.

### <a id="openrc"></a>OpenRC Container (rootfs)

Download the OpenRC rootfs tarball (you can also verify the signature if you like, out of scope for this short tutorial), and unpack into `/var/lib/machines/`, a special marshalling location used by `systemd-nspawn`. To do so, issue:

```console
vps ~ # wget -c https://github.com/sakaki-/nspawn-gentoo-jitsi-meet/releases/download/v1.0.0/gentoo-jitsi-openrc.tar.xz
vps ~ # bsdtar --numeric-owner xf gentoo-jitsi-openrc.tar.xz -C /var/lib/machines/
vps ~ # rm gentoo-jitsi-openrc.tar.xz
```

Create the `systemd-nspawn` boot settings file for this container; issue:

```console
vps ~ # cat >/etc/systemd/nspawn/gentoo-jitsi-openrc.nspawn <<EOD
[Exec]
PrivateUsers=no
Capability=CAP_NET_ADMIN

[Files]

[Network]
Private=no
VirtualEthernet=no
EOD
```

Now you are ready to boot the container. Issue:

```console
vps ~ # systemctl start systemd-nspawn@gentoo-jitsi-openrc
```

Your Gentoo rootfs should now be started in its own process namespace! Unfortunately, despite the recent transition to elogind, you cannot use `machinectl login` with OpenRC targets. However, the image *does* have root login enabled on localhost port 2222, via `ssh`, so issue:

```console
vps ~ # ssh -p 2222 root@localhost
<enter password when prompted>
...
gentoo-jitsi-openrc ~ #
```

> If you have problems with the above `ssh` invocation, try using `ssh -p 2222 -o PubkeyAuthentication=no root@localhost` instead.

The initial container root password is **gentoo64** (of course, you should change this prior to use in production!).

Next, continue reading at "Jitsi Configuration and Bring-Up", [below](#config).

### <a id="systemd"></a>systemd Container (rootfs)

Download the systemd rootfs tarball (you can also verify the signature if you like, out of scope for this short tutorial), and unpack into `/var/lib/machines/`, a special marshalling location used by `systemd-nspawn`. Issue:

```console
vps ~ # wget -c https://github.com/sakaki-/nspawn-gentoo-jitsi-meet/releases/download/v1.0.0/gentoo-jitsi-systemd.tar.xz
vps ~ # bsdtar --numeric-owner xf gentoo-jitsi-systemd.tar.xz -C /var/lib/machines/
vps ~ # rm gentoo-jitsi-systemd.tar.xz
```

Create the `systemd-nspawn` boot settings file for this container; issue:

```console
vps ~ # cat >/etc/systemd/nspawn/gentoo-jitsi-systemd.nspawn <<EOD
[Exec]
PrivateUsers=no
Capability=CAP_NET_ADMIN

[Files]

[Network]
Private=no
VirtualEthernet=no
EOD
```

Now you are ready to boot the container. Issue:

```console
vps ~ # systemctl start systemd-nspawn@gentoo-jitsi-systemd
```

And your Gentoo rootfs should now be booted in its own process namespace! As you have `systemd` running in both the host and guest, you can log in to your Gentoo system simply as follows:


```console
vps ~ # machinectl login gentoo-jitsi-systemd
<enter password when prompted>
...
gentoo-jitsi-systemd ~ #
```

The initial container root password is **gentoo64** (of course, you should change this prior to use in production!).

Next, continue reading at "Jitsi Configuration and Bring-Up", [immediately below](#config).

## <a id="config"></a>Jitsi Configuration and Bring-Up

Configuration of the Jitsi server set is done by following a prompt-driven process to fill out a 'master' configuration file (`/etc/jitsi/jitsi-meet-master-config`; this is a little like the ['env' file](https://github.com/jitsi/docker-jitsi-meet/blob/master/env.example) in the Docker variant of Jitsi Meet), and is common to both OpenRC and systemd containers. A set of callbacks, one per component, are then automatically invoked to build the underlying configurations (and associated setup steps) for jitsi-videobridge, jicofo, the webserver etc. This ensures that passwords, UUIDs etc. are consistent across all components.

> Of course, the underlying blocks may also be configured directly by those who are knowledgeable about Jitsi's 'wiring', and indeed non-single-box installs will require you to do so. But the point of the approach taken here is to help you get a simple instance up and running with the minimum of hassle.

In what follows, I'm going to assume you have a DNS 'A' record pointing to your VPS, IP address 82.221.139.201, with fully-qualified domain name (FQDN) of foo.barfoo.org &mdash; obviously, adapt this for your actual system.

OK, so now (at the container shell you opened during the last step) issue:

```console
gentoo-jitsi-<initsys> ~ # emerge --config jitsi-meet-master-config
```

and you should be prompted to enter the answers to various questions. In what follows, you can simply press <kbd>Enter</kbd> to accept the offered default (shown in square brackets) or, type a different answer and then press <kbd>Enter</kbd>. The configuration process provides quite a lot of explanatory text for each question but, in the interests of brevity, in the below only the questions (and responses) will be shown.

> Remember, this for the (notional) domain foo.barfoo.org, at IP address 82.221.139.201, administrative email your@email.address; obviously, adapt for your own system!

Here's a full flow (minus commentary); parts you may or may not see (depending on whether or not it is the first time you are running the config) are shown in parentheses:

```console
gentoo-jitsi-<initsys> ~ # emerge --config jitsi-meet-master-config
(* Re-use existing configuration for ...? (y/n) [n]: <press Enter>)
* Hostname [...]: <type foo.barfoo.org and press Enter>
* External IP address [...]: <type 82.221.139.201 and press Enter>
* Internal IP address [...]: <type 82.221.139.201 and press Enter>
* Create a localhost alias for foo.barfoo.org? (recommended) (y/n) [y]: <press Enter>
* Copy foo.barfoo.org into "/etc/hostname"? (recommended) (y/n) [y]: <press Enter>
* Max videobridge daemon RAM [1024m]: <press Enter>
* Username [admin]: <press Enter>
* Password [horse-battery-staple]: <press Enter - your generated password will differ>
* Supply pre-existing key/crt pair for foo.barfoo.org? (y/n) [n]: <press Enter>
* Activate Let's Encrypt for foo.barfoo.org? (y/n) [y]: <press Enter>
* Email address [...]: <type your@email.address and press Enter>
* Configuration complete: write it? (y/n) [y]: <press Enter>
* Build component configurations from this? (y/n) [y]: <press Enter>
```

> It is possible to reconfigure the server any number of times.


Once the underlying component configuration completes, you can try starting up your new Jitsi Meet server instance!

To do so, if working in the OpenRC container, issue:

```console
gentoo-jitsi-openrc ~ # rc-service jitsi-meet-server restart
```

Or, in the systemd container issue:

```console
gentoo-jitsi-systemd ~ # systemctl restart jitsi-meet-server

```

This will start the various component servers (nginx, jitsi-videobridge, jicofo, turnserver/coturn and prosody).

Startup will take about five seconds. Once done, you should be able to browse to https://foo.barfoo.org (remember, this is just a *fictional* example, use your own FQDN!) from any remote machine and start a new meeting! If all is well, you should find that the browser reports a valid TLS certificate ('padlock' symbol) issued by Let's Encrypt.

> An automatic renewal job is also scheduled for you, which will keep this certificate up-to-date, restarting the webserver and turnserver as required.

In the browser, allow use of your camera / microphone when prompted, and then click on 'I am the host' in the pop-up dialog which appears, and use the credentials above (in this example "admin" / "horse-battery-staple"; obviously, adapt as appropriate). The new meeting will start, and you will see yourself on screen. You should then be able to send the meeting URL (click the circular (i) icon at the bottom of the window to see it, or look in your browser address bar) to others to enable them to join - they will not require these credentials (however, you *can* set a 'room' password before inviting any guests, if you wish, again via the circular (i) icon). Note that as you have an externally valid TLS certificate, users can *also* join via the official Android or iOS Jitsi apps.

> To allow Android or iOS app users to *initiate* new meetings, you'll need to have them add your host address (in this example, `https://foo.barfoo.org/`) in the Server URL field, under the settings tab of the app (you will also need to let them know the convener credentials &mdash; here, "admin" / "horse-battery-staple").

Should you wish to bring the server complex down at any point, in the OpenRC container, issue:

```console
gentoo-jitsi-openrc ~ # rc-service jitsi-meet-server stop
```

or, in the systemd container issue:

```console
gentoo-jitsi-systemd ~ # systemctl stop jitsi-meet-server

```

> Logs may be viewed at `/var/log/jitsi/...` on both containers, and in `/var/log/messages` on the OpenRC container / the local journal on the systemd container.


## <a id="startonboot"></a>Starting Jitsi Automatically

Once you have your Jitsi server instance working well, you may decide you want to start it up automatically whenever your VPS reboots.

To do so, if working with the OpenRC container, issue:

```console
gentoo-jitsi-openrc ~ # rc-service add jitsi-meet-server default
```

Or, if using the systemd variant container, instead issue:

```console
gentoo-jitsi-systemd ~ # systemdctl enable jitsi-meet-server
```

The above ensures that Jitsi will start whenever the container does, but of course, you *also* need to ensure the container *itself* is booted at (host) system startup. To do so, exit back out of your container shell using <kbd>Ctrl</kbd><kbd>d</kbd>, and then, in the host shell, issue either:

```console
vps ~ # systemctl enable systemd-nspawn@gentoo-jitsi-openrc
```

if you are using the OpenRC container, or

```console
vps ~ # systemctl enable systemd-nspawn@gentoo-jitsi-systemd
```

if you are using the systemd container.

That's it! Now when the host VPS boots, your Gentoo container will automatically start, and then *its* init system will boot Jitsi.

## <a id="feedback"></a>Feedback and Bugs

These rootfs are made available in the hope that they will be useful but without any warranty. Use at your own risk!

Please report any issues (or feedback, which is welcomed) to <sakaki@deciban.com>.

Have fun ^-^
