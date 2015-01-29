---
title: Rerouting Outbound Email to Prevent Spamming During Debugging and Testing 
description: Prevent your Drupal site's development and testing environments from sending emails to your real users, with the reroute_email module and Pantheon environment variables.
category:
  - developing
  - Drupal
---
Drupal sites that send email using the php sendmail() function or with modules like [SMTP](https://www.drupal.org/project/smtp) have been known to spam real users with emails from local, development, or staging environments. While there are [other ways to manage email handling for development or testing](https://www.drupal.org/node/201981), the Drupal [Reroute Email](https://www.drupal.org/project/reroute_email) module is a great solution to this problem. This guide will walk you through installation, configuration in [settings.php](https://www.getpantheon.com/docs/articles/drupal/configuring-settings-php/) with [environmental variables](https://www.getpantheon.com/docs/articles/sites/code/reading-pantheon-environment-configuration/), and testing.

## Installation

Download and install as usual. You can use [SFTP](https://www.getpantheon.com/docs/articles/sites/code/developing-directly-with-sftp-mode/) on Pantheon or the [Drupal UI](https://www.getpantheon.com/docs/articles/sites/code/more-ways-of-managing-code-in-sftp-mode/#installing-modules-and-themes-with-drupal%27s-update-manager) to install a module, but my preference is to stay in Git mode, keep those automated backups running on Dev, and stay on the command line whenever possible.

I chose [Drupal 7 as a start state](https://www.getpantheon.com/docs/articles/users/choosing-start-state/#importing-existing-sites) and performed a [git clone](https://www.getpantheon.com/docs/articles/local/starting-with-git/) of my Pantheon site.
```
$ cd sites
$ git clone [pantheon git clone ssh connection string]
$ cd reroute-email
$ mkdir sites/all/modules/contrib
```
I added a /sites/all/modules/contrib directory, so that Drush will download contrib modules to it. This is a common way to keep modules organized. Next, I'll install the module I need.  
```
$ drush dl reroute_email
```
The following line isn’t necessary, but it’s a good idea to use git status to understand the state of your local Git repository, especially if you’re new to Git. If you’re just starting with Git, I encourage you to do a git status between each of the steps. Verify that the new, untracked files are in /sites/all/modules/contrib/reroute_email.
```
$ git status
```
Add the module to git.
```
$ git add sites/all/modules/contrib/reroute_email
```
Commit and push the module to Pantheon.
```
$ git commit -m "Add reroute_email module"
$ git push origin master
```
Now check your Site Dashboard and you’ll see that the module’s code has been deployed to your Dev environment.

![The dashboard's showing the code was deployed to the Dev environment](/source/docs/assets/images/verify-reroute-email-dashboard-commits1.png)

Visit the development environment and enable the module.

##Configuration

If you don’t have a settings.php file, copy the default.settings.php file.  You can copy the file however you like, but my preference is from the command line:
```
$ cp sites/default/default.settings.php sites/default/settings.php
```
Using your favorite editor or IDE (lately I use [vim](http://www.vim.org) or [atom.io](https://atom.io)), open settings.php, and add the following:
```
if (defined('PANTHEON_ENVIRONMENT')) {
  if ($_SERVER['PANTHEON_ENVIRONMENT'] == 'live') {
	// Do not reroute email on Live
	$conf['reroute_email_enable'] = 0;
  }
  else {
       // Reroute email on all Pantheon environments but Live
	$conf['reroute_email_enable'] = 1;
	$conf['reroute_email_address'] = "youremail+qa-" . PANTHEON_ENVIRONMENT . "@example.com";
	$conf['reroute_email_enable_message'] = 1;
  }
}

if (!defined('PANTHEON_ENVIRONMENT')) {
  // Reroute email when site is not on Pantheon (local install)
  $conf['reroute_email_enable'] = 1;
  $conf['reroute_email_address'] = "youremail+qa-local@example.com";
  $conf['reroute_email_enable_message'] = 1;
}
```
A few notes:
- In order for the snippet to work as intended, the module must be enabled in all environments.
- The PANTHEON_ENVIRONMENT variable changes the reroute_email settings based on environment. This will override any settings in the Drupal Admin UI.
- For the email address, I chose to not create several new email addresses, although you can definitely do that.
- I used my existing email address, taking advantage of the plus sign so I can have “extra” email addresses that are all delivered to my existing email address. It’s not a new trick, but it’s a handy feature [baked into Gmail](http://gmailblog.blogspot.com/2008/03/2-hidden-ways-to-get-more-from-your.html) and some other mail services. If you’re taking this route, you’ll also want to set up [email filters](https://support.google.com/mail/answer/6579?hl=en) to skip the inbox and label it appropriately based on the To: header.

For more about Reroute Email’s settings, see the README.txt that ships with the module.


##Stage and Commit Settings.php
```
$ git add sites/default/settings.php
$ git commit
```
I’ve chosen not to use the -m flag with the commit so I can use my text editor to write a longer, more informative commit message that communicates exactly what my intent is:
```
Configure reroute_email via settings.php

Intercept all outgoing emails for all environments but Live and reroute to QA email addresses so I never spam customers during testing again!

* Do not reroute email on Live
* Reroute email on all other Pantheon environments
* Reroute email on non-Pantheon environments (local)

**Note:** In order for the settings.php config to work correctly, the reroute_email module must be enabled in all environments.

Project page: https://www.drupal.org/project/reroute_email
```

Finally, push the code to Pantheon and enable the module in all environments.
```
$ git push origin master
$ drush pauth  # See [Terminus - the Pantheon Command-Line Interface](https://www.getpantheon.com/docs/articles/local/terminus-the-pantheon-command-line-interface/)
```
I am only enabling it in Dev because that’s the only environment that currently exists on this site. When I deploy and create other environments, the database will be cloned and the module will be enabled in those environments as well.
```
$ drush @pantheon.mysite.dev reroute_email en -y
```
Now the Dev environment’s settings page for reroute_email (/admin/config/development/reroute_email) should look something like this:

![The Reroute Email Configuration menu shows the email settings](/source/docs/assets/images/reroute-email-config-settings.png)

If you don’t see what you’re expecting, review your settings.php and ensure the commit is showing on your Dashboard:

![The dashboard showing the code was deployed to the Dev environment](/source/docs/assets/images/verify-reroute-email-dashboard-commits2.png)

##Go Forth and Test

That’s it! Now when Drupal sends out an email from any environment (except Live), it will get rerouted to the email address specified in settings.php. Our settings.php will make sure email is not rerouted on Live, so it’s business as usual. Make sure you’re using a [SMTP gateway](https://www.getpantheon.com/docs/articles/sites/code/email/#outgoing-email) on Live to ensure email deliverability.

##See Reroute Email In Action
To see exactly what we did, I forked a new [MultiDev](https://www.getpantheon.com/docs/articles/sites/multidev/) CDE called ```demo``` and requested a new account:

![The Reroute Email module account creation page](/source/docs/assets/images/reroute-email-account-requested.png)

Requesting a new account fires off two emails: one to the requestor and another to the site owner; both are successfully rerouted to the email address defined in settings.php:

![Email showing the reroute was successful](/source/docs/assets/images/reroute-email-confirmation.png)