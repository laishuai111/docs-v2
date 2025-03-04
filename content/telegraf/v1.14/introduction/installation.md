---
title: Installing Telegraf

menu:
  telegraf_1_14:
    name: Installing
    weight: 20
    parent: Introduction
---

This page provides directions for installing, starting, and configuring Telegraf.

## Requirements

Installation of the Telegraf package may require `root` or administrator privileges in order to complete successfully.

### Networking

Telegraf offers multiple service [input plugins](/telegraf/v1.14/plugins/inputs/) that may
require custom ports.
All port mappings can be modified through the configuration file,
which is located at `/etc/telegraf/telegraf.conf` for default installations.

### NTP

Telegraf uses a host's local time in UTC to assign timestamps to data.
Use the Network Time Protocol (NTP) to synchronize time between hosts; if hosts' clocks
aren't synchronized with NTP, the timestamps on the data can be inaccurate.

## Installation

{{< tabs-wrapper >}}
{{% tabs %}}
[Ubuntu & Debian](#)
[RedHat & CentOS](#)
[SLES & openSUSE](#)
[FreeBSD/PC-BSD](#)
[macOS](#)
[Windows](#)
{{% /tabs %}}
{{% tab-content %}}
For instructions on how to install the Debian package from a file, please see the [downloads page](https://influxdata.com/downloads/).

Debian and Ubuntu users can install the latest stable version of Telegraf using the `apt-get` package manager.

**Ubuntu:** Add the InfluxData repository with the following commands:

{{< code-tabs-wrapper >}}
{{% code-tabs %}}
[wget](#)
[curl](#)
{{% /code-tabs %}}

{{% code-tab-content %}}
```bash
wget -qO- https://repos.influxdata.com/influxdb.key | sudo apt-key add -
source /etc/lsb-release
echo "deb https://repos.influxdata.com/${DISTRIB_ID,,} ${DISTRIB_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
```
{{% /code-tab-content %}}

{{% code-tab-content %}}
```bash
curl -s https://repos.influxdata.com/influxdb.key | sudo apt-key add -
source /etc/lsb-release
echo "deb https://repos.influxdata.com/${DISTRIB_ID,,} ${DISTRIB_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
```
{{% /code-tab-content %}}
{{< /code-tabs-wrapper >}}  

**Debian:** Add the InfluxData repository with the following commands:

{{< code-tabs-wrapper >}}
{{% code-tabs %}}
[wget](#)
[curl](#)
{{% /code-tabs %}}

{{% code-tab-content %}}
```bash
# Before adding Influx repository, run this so that apt will be able to read the repository.

sudo apt-get update && sudo apt-get install apt-transport-https

# Add the InfluxData key

wget -qO- https://repos.influxdata.com/influxdb.key | sudo apt-key add -
source /etc/os-release
test $VERSION_ID = "7" && echo "deb https://repos.influxdata.com/debian wheezy stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
test $VERSION_ID = "8" && echo "deb https://repos.influxdata.com/debian jessie stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
test $VERSION_ID = "9" && echo "deb https://repos.influxdata.com/debian stretch stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
test $VERSION_ID = "10" && echo "deb https://repos.influxdata.com/debian buster stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
```
{{% /code-tab-content %}}

{{% code-tab-content %}}
```bash
# Before adding Influx repository, run this so that apt will be able to read the repository.

sudo apt-get update && sudo apt-get install apt-transport-https

# Add the InfluxData key

curl -s https://repos.influxdata.com/influxdb.key | sudo apt-key add -
source /etc/os-release
test $VERSION_ID = "7" && echo "deb https://repos.influxdata.com/debian wheezy stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
test $VERSION_ID = "8" && echo "deb https://repos.influxdata.com/debian jessie stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
test $VERSION_ID = "9" && echo "deb https://repos.influxdata.com/debian stretch stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
```
{{% /code-tab-content %}}
{{< /code-tabs-wrapper >}}

Then, install and start the Telegraf service:

```bash
sudo apt-get update && sudo apt-get install telegraf
sudo service telegraf start
```

Or if your operating system is using systemd (Ubuntu 15.04+, Debian 8+):
```
sudo apt-get update && sudo apt-get install telegraf
sudo systemctl start telegraf
```

{{% /tab-content %}}
{{% tab-content %}}
For instructions on how to install the RPM package from a file, please see the [downloads page](https://influxdata.com/downloads/).

**RedHat and CentOS:** Install the latest stable version of Telegraf using the `yum` package manager:

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/influxdb.repo
[influxdb]
name = InfluxDB Repository - RHEL \$releasever
baseurl = https://repos.influxdata.com/rhel/\$releasever/\$basearch/stable
enabled = 1
gpgcheck = 1
gpgkey = https://repos.influxdata.com/influxdb.key
EOF
```

Once repository is added to the `yum` configuration,
install and start the Telegraf service by running:

```bash
sudo yum install telegraf
sudo service telegraf start
```

Or if your operating system is using systemd (CentOS 7+, RHEL 7+):

```sh
sudo yum install telegraf
sudo systemctl start telegraf
```
{{% /tab-content %}}
{{% tab-content %}}
There are RPM packages provided by openSUSE Build Service for SUSE Linux users:

```bash
# add go repository
zypper ar -f obs://devel:languages:go/ go
# install latest telegraf
zypper in telegraf
```
{{% /tab-content %}}

{{% tab-content %}}
Telegraf is part of the FreeBSD package system.
It can be installed by running:

```bash
sudo pkg install telegraf
```

The configuration file is located at `/usr/local/etc/telegraf.conf` with examples in `/usr/local/etc/telegraf.conf.sample`.
{{% /tab-content %}}

{{% tab-content %}}
Users of macOS 10.8 and higher can install Telegraf using the [Homebrew](http://brew.sh/) package manager.
Once `brew` is installed, you can install Telegraf by running:

```bash
brew update
brew install telegraf
```

To have launchd start telegraf at next login:

```sh
ln -sfv /usr/local/opt/telegraf/*.plist ~/Library/LaunchAgents
```
To load telegraf now:

```sh
launchctl load ~/Library/LaunchAgents/homebrew.mxcl.telegraf.plist
```

Or, if you don't want/need launchctl, you can just run:

```sh
telegraf -config /usr/local/etc/telegraf.conf
```
{{% /tab-content %}}
{{% tab-content %}}
Install Telegraf as a [Windows service](https://github.com/influxdata/telegraf/blob/master/docs/WINDOWS_SERVICE.md) (Windows support is experimental):

```sh
telegraf.exe -service install -config <path_to_config>
```
{{% /tab-content %}}

{{< /tabs-wrapper >}}

### Verify the authenticity of downloaded binary (optional)

InfluxData cryptographically signs each Telegraf binary release.
For added security, follow these steps to verify the signature of your download with `gpg`.

(Most operating systems include the `gpg` command by default.
If `gpg` is not available, see the [GnuPG homepage](https://gnupg.org/download/) for installation instructions.)

1. Download and import InfluxData's public key:

    ```
    curl -s https://repos.influxdata.com/influxdb.key | gpg --import
    ```

2. Download the signature file for the release by adding `.asc` to the download URL.
   For example:

    ```
    wget https://dl.influxdata.com/telegraf/releases/telegraf-{{< latest-patch >}}_linux_amd64.tar.gz.asc
    ```

3. Verify the signature with `gpg --verify`:

    ```
    gpg --verify telegraf-{{< latest-patch >}}_linux_amd64.tar.gz.asc telegraf-{{< latest-patch >}}_linux_amd64.tar.gz
    ```

    The output from this command should include the following:

    ```
    gpg: Good signature from "InfluxDB Packaging Service <support@influxdb.com>" [unknown]
    ```

## Configuration

### Create a configuration file with default input and output plugins.

Every plugin will be in the file, but most will be commented.

```
telegraf config > telegraf.conf
```

### Create a configuration file with specific inputs and outputs
```
telegraf --input-filter <pluginname>[:<pluginname>] --output-filter <outputname>[:<outputname>] config > telegraf.conf
```

For more advanced configuration details, see the
[configuration documentation](/telegraf/v1.14/administration/configuration/).
