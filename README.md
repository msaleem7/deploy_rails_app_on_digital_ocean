# Ruby on Rails Deployment on DigitalOcean

## Prequisites

  - HTTP server `nginx`
  - Rails web server `passenger`
  - Database `PosgreSQL`
  - Deployment utility `mina`

## Step 1 - Create your droplet
  - Visit [DigitalOcean droplet creation url](https://cloud.digitalocean.com/droplets/new)
  - Set a name for your droplet
  - Select your droplet region
  - Select your droplet image (OS), `Ubuntu` is prefered.
  - After creating your droplet, you are going to receive an email from DigitalOcean which contains your `root password` and `instance's IP`

  	> Your new Droplet is all set to go! You can access it using the following credentials

  	> Droplet Name: eastagile-droplet

	> IP Address: 104.236.34.222

	> Username: root

	> Password: dbvpnxgasrse


## Step 2 - Add a sudo user for deployment

  - SSH to your `instance's IP` with your `root password`

 	>  ssh root@INSTANCE_IP

  - As of May 7, Ubuntu 14.04 x64, you may be asked to change root password
  	> Changing password for root.

  	> (current) UNIX password:

  	> Enter new UNIX password:

  	> Retype new UNIX password:

  - Create a new user used for deployment

  	> adduser deploy

  - You will be asked for your `deploy` user password. Please keep it safe.
  - You will be asked a few questions, all of them are optional so that you can hit `ENTER` to skip those fields.
  - Add your newly created user to `sudo` group

  	> gpasswd -a deploy sudo

  - Add a public key authentication
    - Generate a key pair (from your local machine)
    	> ssh-keygen
   	- Assumming your local username is `eastagile`, you will see output similar to this
    	> Generating public/private rsa key pair.

    	> Enter file in which to save the key (/Users/eastagile/.ssh/id_rsa):
    - Copy the public key

    	> cat ~/.ssh/id_rsa.pub

    - It should print your public SSH key, which looks similar to this

    	> ssh-rsa AAAAB3NzaC1yc2EAAAA...

    - Add public key to the new `deploy` user
      + On the server, as the root user, switch to `deploy` user

      	> su - deploy

      + You are now in the `deploy` user home folder, now create a `.ssh` folder

      	> mkdir ~/.ssh

      	> chmod 700 ~/.ssh

      + In `.ssh` folder, create a file called `authorized_keys` and add your public key to this file

      	> nano .ssh/authorized_keys

      + Restrict the permissions of this file

      	> chmod 600 ~/.ssh/authorized_keys

      + Return to root user

      	> exit

    - Configuring SSH
      + Opening configuration file

      	> nano /etc/ssh/sshd_config

      + Disable root login through SSH by modifying this line in the configuration file

      	> PermitRootLogin no

      + Save the file and reload SSH service

      	> service ssh restart

## Step 3 - Installing ruby

  - SSH to your server via `deploy` user

    > ssh deploy@INSTANCE_IP
  - Installing rvm
  	> curl -sSL https://rvm.io/mpapis.asc | gpg --import -

  	> curl -sSL https://get.rvm.io | bash -s stable
  - Exit your server and ssh to it again to re-sourcing the `.bashrc` file so that RVM is available
  - Update all packages

  	> sudo apt-get update
  - Install ruby

  	> rvm install 2.2.3

  - Use current ruby as default system ruby

  	> rvm use 2.2.3 --default

  - Double check if ruby is installed properly

  	> ruby -v

  	> ruby 2.2.3

## Step 4 - Installing nginx and passenger

  - Install `passenger` gem

  	> gem install passenger
  	> gem install bundler

  - Run the phusion passenger installer and follow the onscreen instruction

   	> rvmsudo passenger-install-nginx-module
  - In my case, I have to run the following commands so that the installation process can continue
  	> sudo apt-get install libcurl4-openssl-dev

  	> sudo dd if=/dev/zero of=/swap bs=1M count=1024

  	> sudo mkswap /swap

  	> sudo swapon /swap
  - I chosed option 1 for `nginx` installation (`download, compile and install Nginx for me`)

## Step 5 - Setup PosgreSQL

  - Installing PosgreSQL
  	> sudo apt-get install postgresql postgresql-contrib

  - Login as `posgres` user

  	> sudo -i -u postgres

  - Run the PosgreSQL console

  	> psql

  - Create a new user and password for your upcoming Rails database (your database username will be `deploy` and your database password will be `deploypassword`)

  	> create role deploy with createdb login password 'deploypassword';

  - Also create a `deploy` database so that you can run `psql` from `deploy` user (type `\q` to quit the `psql` console)

  	> create database deploy;

  - Exit the `posgres` user

  	> exit

## Step 5 - Setup nginx
  - Install nginx as a service
   	> wget -O init-deb.sh https://gist.githubusercontent.com/chautoni/1e1dea0ca1a43d245393/raw/c8825bf2e9c9243201e4e0e974626501592ce81e/init-deb.sh

   	> sudo mv init-deb.sh /etc/init.d/nginx

   	> sudo chmod +x /etc/init.d/nginx

   	> sudo /usr/sbin/update-rc.d -f nginx defaults

   	> sudo service nginx start
  - Edit the nginx configuration
  	> sudo nano /opt/nginx/conf/nginx.conf

  - Please make sure that your `nginx.conf` looks similar to this
  ```ruby
  	server {
		listen 80;
		server_name example.com;
		passenger_enabled on;
		root /home/deploy/eastagile_digital_ocean_deployment/current/public;
	}
  ```
  - Reload the nginx server
  	> sudo service nginx restart

## Step 6 - Setup mina for deployment automation

  - Generate mina sample `deploy.rb`
  	> mina init

  - This is a sample `config/deploy.rb` after configration
  ```ruby
    require 'mina/bundler'
    require 'mina/rails'
    require 'mina/git'
    require 'mina/rvm'    # for rvm support. (http://rvm.io)

    set :domain, '104.236.34.222'
    set :deploy_to, '/home/deploy/eastagile_digital_ocean_deployment'
    set :repository, 'git@github.com:chautoni/eastagile_digital_ocean_deployment.git'
    set :branch, 'master'

    set :shared_paths, ['config/database.yml', 'config/secrets.yml', 'log']
    set :user, 'deploy'    # Username in the server to SSH to.
    set :forward_agent, true     # SSH forward_agent.

    task :environment do
      invoke :'rvm:use[ruby-2.1.2@digital_ocean_deployment]'
    end

    task :setup => :environment do
      queue! %[mkdir -p "#{deploy_to}/#{shared_path}/log"]
      queue! %[chmod g+rx,u+rwx "#{deploy_to}/#{shared_path}/log"]

      queue! %[mkdir -p "#{deploy_to}/#{shared_path}/config"]
      queue! %[chmod g+rx,u+rwx "#{deploy_to}/#{shared_path}/config"]

      queue! %[touch "#{deploy_to}/#{shared_path}/config/database.yml"]
      queue  %[echo "-----> Be sure to edit '#{deploy_to}/#{shared_path}/config/database.yml'."]

      queue! %[touch "#{deploy_to}/#{shared_path}/config/secrets.yml"]
      queue  %[echo "-----> Be sure to edit '#{deploy_to}/#{shared_path}/config/secrets.yml'."]
    end

    desc "Deploys the current version to the server."
    task :deploy => :environment do
      to :before_hook do
        # Put things to run locally before ssh
      end
      deploy do
        invoke :'git:clone'
        invoke :'deploy:link_shared_paths'
        invoke :'bundle:install'
        invoke :'rails:db_migrate'
        invoke :'rails:assets_precompile'
        invoke :'deploy:cleanup'

        to :launch do
          queue "mkdir -p #{deploy_to}/#{current_path}/tmp/"
          queue "touch #{deploy_to}/#{current_path}/tmp/restart.txt"
        end
      end
    end
  ```
  - ssh to your server and create the designated folder for deployment

  	> mkdir /home/eastagile_digital_ocean_deployment

  - Then run mina setup command in your local machine
  	> mina setup

  	> -----> Done.

  	> -----> Be sure to edit '/home/deploy/eastagile_digital_ocean_deployment/shared/config/database.yml'.

  	> -----> Be sure to edit '/home/deploy/eastagile_digital_ocean_deployment/shared/config/secrets.yml'.
  - ssh to your server and edit your `database.yml` and `secrets.yml` file with appropriate credentials.

## Step 7 - Deploy

  - Installing git onto your server

  	> sudo apt-get install git

  - Add `github.com` cert to your server `known_hosts`

  	> ssh-keyscan -H github.com >> ~/.ssh/known_hosts

  - Run `psql` and create `eastagile_digital_ocean_deployment` database
  	> create database digital_ocean_production;

  - Install additional PosgreSQL and JS runtime packages
  	> sudo apt-get install nodejs libpq-dev
  
  - Install gem bundle to install gem when deploy on specific gemset
        > rvm use ruby-2.1.2@digital_ocean_deployment
        > gem install bundle

  - Deploy to server from your local machine
  	> mina deploy

## There is no step 8. Enjoy your deployment.

## Revision
  - Version 1.0 - Initial guide ([ThachChau](http://github.com/chautoni))
  - Version 1.1 - Update guide to latest ruby version 2.2.3
  - Version 1.2 - Update missing command: install bundle before run `mina deploy`

## References
  - [How To Deploy a Rails App with Passenger and Nginx on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-rails-app-with-passenger-and-nginx-on-ubuntu-14-04)
  - [Initial Server Setup with Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-14-04)
  - [How To Install and Use PostgreSQL on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-14-04)
