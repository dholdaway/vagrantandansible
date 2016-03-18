# Web Architecture Example - Ansible

This is a simple example of a complete web architecture configuration using Ansible to configure a set of VMs either on local infrastructure using VirtualBox and Vagrant (using the included Vagrantfile), or on a cloud hosting provider (in this case, DigitalOcean).

The architecture for the example web application will be:

                        -------------------------
                       |  varnish.dev (Varnish)  |
                       |  192.168.2.2            |
                        -------------------------
                          /                   \
           ---------------------          ---------------------
          |  www1.dev (Apache)  |        |  www1.dev (Apache)  |
          |  192.168.2.3        |        |  192.168.2.4        |
           ---------------------          ---------------------
                          \                   /
                      -----------------------------
                     |  memcached.dev (Memcached)  |
                     |  192.168.2.7                |
                      -----------------------------
                          /                   \
      ----------------------------       ----------------------------
     |  db1.dev (MySQL - Master)  |     |  db2.dev (MySQL - Slave)   |
     |  192.168.2.5               |     |  192.168.2.6               |
      ----------------------------       ----------------------------

*IP addresses and hostnames in this diagram are modeled after local VirtualBox/Vagrant-based VMs.*

This architecture offers multiple levels of caching and high availability/redundancy on almost all levels, though to keep it simple, there are single points of failure. All persistent data stored in the database is stored in a slave server, and one of the slowest and most constrained parts of the stack (the web servers, in this case running a PHP application through Apache) is easy to scale horizontally, behind Varnish, which is acting as a caching (reverse proxy) layer and load balancer.

For the purpose of demonstration, Varnish's caching is completely disabled, so you can refresh and see both Apache servers (with caching enabled, Varnish would cache the first response then keep serving it without hitting the rest of the stack). You can see the caching and load balancing configuration in `playbooks/varnish/templates/default.vcl`).

## Prerequisites

Before you can run any of these playbooks, you will need to [install Ansible](http://docs.ansible.com/intro_installation.html), and run the following command to download dependencies (from within the same directory as this README file):

    $ ansible-galaxy install -r requirements.yml

If you would like to build the infrastructure locally, you will also need to install the latest versions of [VirtualBox](https://www.virtualbox.org/wiki/Downloads) and [Vagrant](https://www.vagrantup.com/downloads.html).

## Build and configure the servers (Local)

To build the VMs and configure them using Ansible, follow these steps (both from within this directory):

  1. Run `vagrant up`.
  2. Run `ansible-playbook configure.yml -i inventories/vagrant/`.

This guide assumes you already have Vagrant, VirtualBox, and Ansible installed locally.

After everything is booted and configured, visit http://varnish.dev/ (if you configured the domain in your hosts file with the line `192.168.2.2  varnish.dev`) in a browser, and refresh a few times to see that Varnish, Apache, PHP, Memcached, and MySQL are all working properly!

## Build and configure the servers (DigitalOcean)

Pre-suppositions: You have a DigitalOcean account, and you have your v1 API Client ID and API Key from the account. Additionally, you have `dopy` and Ansible installed on your workstation (install `dopy` with `sudo pip install dopy`).

To build the droplets and configure them using Ansible, follow these steps (both from within this directory):

  1. Set your DigitalOcean v1 Client ID: `export DO_CLIENT_ID=[client ID here]`
  2. Set your DigitalOcean v1 API Key: `export DO_API_KEY=[api key here]`
  3. Run `ansible-playbook provision.yml`.

After everything is booted and configured, visit the IP address of the Varnish server that was created in your DigitalOcean account in a browser, and refresh a few times to see that Varnish, Apache, PHP, Memcached, and MySQL are all working properly!

### Notes

  - Public IP addresses are used for all cross-droplet communication (e.g. PHP to MySQL/Memcached communication, MySQL master/slave replication). For better security and potentially a tiny performance improvement, you can use droplets' `private_ip_address` for cross-droplet communication.
  - Hosting active or inactive droplets on DigitalOcean will incur hosting fees (normally $0.01 USD/hour for the default 512mb droplets used in this example). While the charges will be nominal (likely less than $1 USD for many hours of testing), it's important to destroy droplets you aren't actively using!
  - You can use the included `digital_ocean.py` inventory script for dynamic inventory (`python digital_ocean.py --pretty` to test).

## Build and configure the servers (AWS)

Pre-suppositions: You have an Amazon Web Services account with a valid payment method configured, and you have your AWS Access Key and AWS Secret Key from your account. Additionally, you have `boto` and Ansible installed on your workstation (install `boto` with `sudo pip install boto`).

To build the droplets and configure them using Ansible, follow these steps (both from within this directory):

  1. Set your AWS Access Key: `export AWS_ACCESS_KEY_ID=[access key here]`
  2. Set your AWS Secret Key: `export AWS_SECRET_ACCESS_KEY=[secret key here]`
  3. Run `ansible-playbook provision.yml`.

After everything is booted and configured, visit the IP address of the Varnish server that was created in your AWS account in a browser, and refresh a few times to see that Varnish, Apache, PHP, Memcached, and MySQL are all working properly!

### Notes

  - Public IP addresses are used for all cross-instance communication (e.g. PHP to MySQL/Memcached communication, MySQL master/slave replication). For better security and potentially a tiny performance improvement, you can use instances' `private_ip` for cross-instance communication.
  - Hosting instances on AWS may incur hosting fees (unless all usage falls within AWS's first-year free tier limits). While the charges will be nominal (likely less than $1 USD for many hours of testing), it's important to destroy instances you aren't actively using!
  - You can use the included `ec2.py` inventory script for dynamic inventory (`./ec2.py --list` to test).
# Ansible Vagrant profile for a LAMP server

## Background

Vagrant and VirtualBox (or some other VM provider) can be used to quickly build or rebuild virtual servers.

This Vagrant profile installs Apache, MySQL and PHP (the 'AMP' part of 'LAMP') using the [Ansible](http://www.ansible.com/) provisioner.

## Getting Started

This README file is inside a folder that contains a `Vagrantfile` (hereafter this folder shall be called the [vagrant_root]), which tells Vagrant how to set up your virtual machine in VirtualBox.

To use the vagrant file, you will need to have done the following:

  1. Download and Install [VirtualBox](https://www.virtualbox.org/wiki/Downloads)
  2. Download and Install [Vagrant](https://www.vagrantup.com/downloads.html)
  3. Install [Ansible](http://docs.ansible.com/intro_installation.html)
  4. Open a shell prompt (Terminal app on a Mac) and cd into the folder containing the `Vagrantfile`
  5. Run the following command to install the necessary Ansible roles for this profile: `$ ansible-galaxy install -r requirements.yml`

Once all of that is done, you can simply type in `vagrant up`, and Vagrant will create a new VM, install the base box, and configure it.

Once the new VM is up and running (after `vagrant up` is complete and you're back at the command prompt), you can log into it via SSH if you'd like by typing in `vagrant ssh`. Otherwise, the next steps are below.

### Setting up your hosts file

You need to modify your host machine's hosts file (Mac/Linux: `/etc/hosts`; Windows: `%systemroot%\system32\drivers\etc\hosts`), adding the line below:

    192.168.33.33  lamp

(Where `lamp`) is the hostname you have configured in the `Vagrantfile`).

After that is configured, you could visit http://lamp/ in a browser, and you'll see the Apache 'It works!' page.
