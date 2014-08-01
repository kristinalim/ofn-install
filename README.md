Provisioning OFN with Ansible
===========================================

These are [Ansible] playbooks (scripts)  for managing an Open Food Network app server using nginx, unicorn and rails.

These are the main Ansible playbooks:

* **Install** the underlying software and systems needed by OFN onto a server, then use the **deploy** playbook to install and deploy OFN on the server.
* **Deploy** OFN to a server by copying OFN from a git repo to the server and using ruby/rake/rails tasks to configure OFN so it will run.
* **Backup** database files on the server and also to a local machine


You first must set up a server with the **install** playbook.  This will set up the software and systems that OFN needs and will also call the **deploy** playbook to put OFN onto your server and get it up and running.

You can use the **deploy** playbook repeatedly to deploy specific (e.g. newer/updated) versions of OFN onto your server.

You can use the **backup** playbook to make backups of the OFN database.


The playbooks take information from the vars.yml file. You *must* create your own copy of the file and edit it to put in information that is specific to your OFN installation. Read more in the Requirements section below.

You may want to use the [anisble option "checkrun"](http://docs.ansible.com/playbooks_checkmode.html) to do a dry-run of the playbooks.  (With this option, Ansible will run the playbooks, but not actually make changes on the server.) 



#Requirements

##1. apt-compatible OS: Debian or Ubuntu
These Ansible provisioning scripts will install the infrastructure needed and OFN onto a server running an apt-compatible OS (Debian or Ubuntu).  It's currently been tested on **Ubuntu 12.04 (64 bit)** on both AWS and Digital Ocean cloud servers.
 
##2. Ansible
Install Ansible by following the documentation on [the official Ansible site.]  If you are not already familiar with Ansible, check out the documentation on the site and keep it handy.

[the official Ansible site.]: http://docs.ansible.com/intro_installation.html

##3. Additional Ansible modules
You will need to install the following additional Ansible modules before running the OFN provisioning playbooks: `zzet.rbenv` and `mortik.nginx-rails`.  Here are the Ansible commands to use to install them:

* `ansible-galaxy install zzet.rbenv`
* `ansible-galaxy install mortik.nginx-rails`

##4. A vars.yml File that has Values Set for Your Specific OFN Installation
The playbooks use values set in the `vars.yml` file to accomplish their tasks. Information includes such things as the specific domain name for your OFN system, the password to the database used by OFN, file names, paths, etc.  This repository does *not* have a `vars.yml` file by default:  You must copy the `example-vars.yml` file to create your own `vars.yml` file.  (Ex: `cp example-vars.yml vars.yml`)  Then you must edit the `vars.yml` file and put in the values that are appropriate for your system and set-up.  Note that you may need to edit your `vars.yml` file before running a playbook; you may want to change the `rails_env` from `testing` to `staging` for example.  In other words, you don't just create and edit the `vars.yml` file once and forget about it.
   


#Install


The install playbook installs basic packages (software) and configures them as needed for OFN, then calls the **deploy** playbook to install and deploy OFN onto the server.  
 
Specifically, it installs packages such as curl and git, then installs:

* [rbenv]
* Ruby
* Postgresql
* [Nginx]
* [Unicorn]

  [Ansible]: http://ansible.cc
  [Vagrant]: http://www.vagrantup.com
  [rbenv]: https://github.com/sstephenson/rbenv
  [Nginx]: http://nginx.org/h
  [Unicorn]: http://unicorn.bogomips.org/



##Provide the Information Needed for Install


###1. Set Values for Your Specific Install in vars.yml

If you haven't already created a `vars.yml` file, copy the provided `example-vars.yml` to `vars.yml` and fill in your site-specific variables:

* `cp example-vars.yml vars.yml`

###2. Add Seed Data Specific to your OFN Installation

If you don't want to use seed (initial) data specific to Austalia, provide your own country, state and postcode (a.k.a. suburb or zipcode) files for your location. This is a work-around until a cleaner config process is available.

These are the files that are used for seed data:

* db/default/spree/states.yml
* db/default/spree/countries.yml
* db/seeds.rb
* db/suburb_seeds.rb - This surburbs data is not currently used but may be again in future. But don't spend too much time on it!
* config/initializers/spree.rb

Look at the existing files in the openfoodnetwork repo for examples of how the files are used.

Put these seed files into the 'files' folder.  (Do not put them into any subdirectories. The playbooks will simply look for the files named `states.yml, countries.yml, seeds.rb, suburb_seeds.rb, and spree.rb` in the `files` directory; the playbooks will not look for the files in any subdirectories.)

###3.  Add SSL Certificate Files

For production and staging environments, you will need SSL certificates for the OFN domain name.  (The domain name is specified in the `vars.yml` file.)  Specifically, you will need to provide these two files with these exact file names:

* server.crt 
* server.key 

Put these in the `files` folder.


###4.  Specify the Server IP in your Local Ansible Hosts file

Add the IP or URL for your server(s) to your local ansible hosts file.  (On a unix system, this file is probably at `/etc/ansible/hosts`)



##Set Up the Server and a User:

###1. Set Up an Ubuntu Precise 64 Box

You will need to set up an instance of [Ubuntu "Precise Pangolin" x64 server](https://wiki.ubuntu.com/PrecisePangolin/ReleaseNotes/UbuntuServer) (a.k.a. *an Ubuntu Precise 64 box*), though this may work on other Debian based systems too.

For vagrant, if you don't allready have an Ubuntu Precise 64 box set up, run:

`vagrant box add precise64 https://cloud-images.ubuntu.com/vagrant/precise/current/precise-server-cloudimg-amd64-vagrant-disk1.box`

The DigitalOcean Ubuntu precise 64 image is solidly tested, and Amazon Ubuntu precise 64 images have also worked.


###2. Set Up a Default User

Ansible needs at least one user created on the system so that it can run and install software as that user. You specify the user name and password user in your `vars.yml` file.  (On Ubuntu systems, it is standard practice to create a user named "ubuntu" for this purpose.)  
 
On DigitalOcean servers and any system where there is no default user set up, you will need to run the 'user.yml'playbook.

`ansible-playbook user.yml -f 10 -vvvv`

On vagrant this is done automatically. 

On standard Amazon (AWS) Ubuntu images that block root login, the `user.yml` playbook won't run, and doesn't need to.


##Run the Install Playbook:

Run:

`ansible-playbook install.yml -f 10` (with `-vvvv` for full debug output) 

##Build and Install Notes

For production and staging servers you will need to enter the site url and smtp details through the admin mail interface before the site is fully functional.

*@TODO What does this mean?  What "admin mail interface" is this?  OFN?  ubuntu?*


#Deploy
This gets a copy of OFN from a git repo that your specify (in vars.yml), put it onto the server, and the do ruby/rake/rails tasks and configuration needed so that it can run.  The main tasks accomplished are:

* copy OFN from a git repo to the server
* create the db if needed
* run db migrations
* precompile assets used by OFN
* creates seed data
* creates an initial admin user
This list isn't comprehensive; those are the highlights.

##Provide the Information Needed for Deploy:

###1. Set Values for Your Specific Deployment in vars.yml

If you haven't already created a `vars.yml` file, copy the provided `example-vars.yml` to `vars.yml` and fill in your site-specific variables:

* `cp example-vars.yml vars.yml`

Be sure that you have specified the git repo that you want and the exact version information (tag, commit reference, etc.) that you want from that git repo.


###2. Add Seed Data Specific to your OFN Installation (optional)

If you don't want to use seed (initial) data specific to Austalia *and you haven't already provided it,* or if you want to update it, provide your own country, state and postcode (a.k.a. suburb or zipcode) files for your location. This is a work-around until a cleaner config process is available.

These are the files that are used for seed data:

* db/default/spree/states.yml
* db/default/spree/countries.yml
* db/seeds.rb
* db/suburb_seeds.rb - This surburbs data is not currently used but may be again in future. But don't spend too much time on it!
* config/initializers/spree.rb

Look at the existing files in the openfoodnetwork repo for examples of how the files are used.

Put these seed files into the 'files' folder.  (Do not put them into any subdirectories. The playbooks will simply look for the files named `states.yml, countries.yml, seeds.rb, suburb_seeds.rb, and spree.rb` in the `files` directory; the playbooks will not look for the files in any subdirectories.)


##Run the Deploy Playbook:

Make sure that the git repo has the OFN code (e.g. by pushing code to your repo or branch, or cloning from a different repo), then run: 

`ansible-playbook deploy.yml -f 10` (with `-vvvv` for full debug output) 

to deploy it onto the server(s). 


##Deploy Notes

There is a timestamped backup process included in deployment and a rollback version saved each time.
To roll back after a deployment error, currently you need to manually run:

`ansible-playbook rollback.yml -f 10`

making sure the deployment actually got to the point of making a rollback...


#Backup

##Provide the Information Needed for Backup:

###1. Set Values for Your Specific Systems in vars.yml

If you haven't already created a `vars.yml` file, copy the provided `example-vars.yml` to `vars.yml` and fill in your site-specific variables:

* `cp example-vars.yml vars.yml`

Be sure that you have specified paths for where the backups should be on your server and where they should be put on your local machine.


##Run the Backup Playbook:

To backup on the server and to the local machine run:

`ansible-playbook backup.yml -f 10`

##Backup Notes

This doesn't clean up old backups automatically yet.


#Credits

Written by Rafael Schouten, after inital work from Paul Mackey.

