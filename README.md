# Deploying WordPress on Elastic Beanstalk
These instructions were tested with WordPress 4.6.1

## Install the EB CLI

The EB CLI integrates with Git and simplifies the process of creating environments, deploying code changes, and connecting to the instances in your environment with SSH. You will perform all of these activites when installing and configuring WordPress.

If you have pip, use it to install the EB CLI.

```Shell
$ pip install --user --upgrade awsebcli
```

Add the local install location to your OS's path variable.

###### Linux
```Shell
$ export PATH=~/.local/bin:$PATH
```
###### OS-X
```Shell
$ export PATH=~/Library/Python/3.4/bin:$PATH
```
###### Windows
Add `%USERPROFILE%\AppData\Roaming\Python\Scripts` to your PATH variable. Search for **Edit environment variables for your account.** in the Start menu.

If you don't have pip, follow the instructions [here](http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/eb-cli3-install.html).

## Download and extract WordPress and the configuration files

```
~$ curl https://wordpress.org/wordpress-4.6.1.tar.gz -o wordpress.tar.gz
~$ curl https://github.com/awslabs/eb-php-wordpress/releases/download/v1.0/eb-php-wordpress-v1.zip -o eb-php-wordpress.zip
~$tar -xvf wordpress.tar.gz && mv wordpress wordpress-beanstalk && cd wordpress-beanstalk
~/wordpress-beanstalk$ eb init --platform php7.0 --region us-west-2
~/wordpress-beanstalk$ eb ssh --setup
~/wordpress-beanstalk$ eb create wordpress-beanstalk --sample --database
(choose database username/password, CTRL+C to exit once creation is in-progress)
~/wordpress-beanstalk$ unzip ~/eb-php-wordpress-v1.0.zip
```

## Networking configuration
Modify the configuration files in the .ebextensions folder with the IDs of your [default VPC and subnets](https://console.aws.amazon.com/vpc/home#subnets:filter=default), and [your public IP address](https://www.google.com/search?q=what+is+my+ip). 

 - `.ebextensions/efs-create.config` creates an EFS file system and mount points in each Availability Zone / subnet in your VPC.
 - `.ebextensions/ssh.config` restricts access to your environment to your IP address to protect it during the WordPress installation process.

## Deploy WordPress to your environment
Deploy the project code to your Elastic Beanstalk environment. 

First, confirm that your environment is `Ready` with `eb status`. Environment creation takes about 15 minutes due to the RDS DB instance provisioning time.

```
~/wordpress-beanstalk$ eb status
~/wordpress-beanstalk$ eb deploy
```

### NOTE: security configuration

This project includes a configuration file (`loadbalancer-sg.config`) that creates a security group and assigns it to the environment's load balancer, using the IP address that you configured in `ssh.config` to restrict HTTP access on port 80 to connections from your network. Otherwise, an outside party could potentially connect to your site before you have installed WordPress and configured your admin account.

You can [view the related SGs in the EC2 console](https://console.aws.amazon.com/ec2/v2/home#SecurityGroups:search=wordpress-beanstalk)

## Install WordPress

Open your site in a browser.

```
~/wordpress-beanstalk$ eb open
```

You are redirected to the WordPress installation wizard because the site has not been configured yet.

Perform a standard installation. The `wp-config.php` file is already present in the source code and configured to read database connection information from the environment, so you shouldn't be prompted to configure the connection.

Installation takes about a minute to complete.

# Updating keys and salts

The WordPress configuration file `wp-config.php` also reads values for keys and salts from environment properties. Currently, these properties are all set to `test` by the `wordpress.config` configuration file in the `.ebextensions` folder

The hash salt can be any value but shouldn't be stored in source control. Use `eb setenv` to set these properties directly on the environment.

    AUTH_KEY, SECURE_AUTH_KEY, LOGGED_IN_KEY, NONCE_KEY, AUTH_SALT, SECURE_AUTH_SALT, NONCE_SALT

```
~/wordpress-beanstalk$ eb setenv AUTH_KEY=randomnumbersandletters89237492374
~/wordpress-beanstalk$ eb setenv SECURE_AUTH_KEY=ah24h3drfh97623ljkhsdf293t5fghks
...
```

Setting the properties on the environment directly by using the EB CLI or console overrides the values in `wordpress.config`.

Remove the custom load balancer configuration to open the site to the Internet.
```
~/wordpress-beanstalk$ rm .ebextensions/loadbalancer-sg.config
~/wordpress-beanstalk$ eb deploy
```

When the deployment completes, open the site.
```
~/wordpress-beanstalk$ eb open
```

Finally, scale up to run the site on multiple instances for high availability.
```
~/wordpress-beanstalk$ eb scale 3
```

# Backup

Now that you've gone through all the trouble of installing your site, you will want to back up the data in RDS and EFS that your site depends on. See the following topics for instructions.

 - [DB Instance Backups](http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.BackingUpAndRestoringAmazonRDSInstances.html)
 - [Back Up an EFS File System](http://docs.aws.amazon.com/efs/latest/ug/efs-backup.html)