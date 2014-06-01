## What is ansible-rails?

It is an [ansible](http://www.ansible.com/home) role to deploy a rails application.

### What problem does it solve and why is it useful?

I've been a rails developer for a little over a year now and every time I have to deploy a rails application I always want to take at least 15 innocent kittens and expose their guts using a poleaxe that has been dipped in acid that is strong enough to distort the space-time continuum. Let me tell you the 2 main reason why:

#### 1. Setting up infrastructure is not easy

Unless you want to pay a hefty price for heroku then you'll likely settle on using a cloud provider such as digital ocean, AWS, rackspace, linode or any other popular service. This also means it's up to you to figure out how to provision your server and manage the infrastructure.

#### 2. Capistrano is not the answer for app deployments

Let's say you pass that hurdle, now it's time to get your application on 1 or more servers.

Capistrano is a work horse but one of its main problems is it's slow. A simple rails app that didn't have to run any migrations or precompile new assets would routinely take 60 seconds on an ec2 micro instance. I timed a bunch of deploys months ago and I was averaging around 100 seconds per deploy and keep in mind this is deploying everything to 1 server. The same app can be deployed using ansible in 10 seconds and I feel like it could be trimmed down to 3 seconds with more aggressive checks to determine if certain actions need to happen.

Capistrano v2's code base is also buggy, no longer maintained and a nightmare to wade through. v3 is better but it's still slow.

Another issue I have with capistrano is that it's another tool that you need to learn and maintain. I don't want to use a tool to provision my infrastructure and then use another tool to deploy my applications. I also don't want to keep track of my infrastructure in 2 places or have to write custom tooling to glue both tools together in a way that half way works and is brittle. Luckily ansible was created to solve this exact issue.

Lastly the deploy process is overly complicated. You don't need the concept of a "current" release, a bunch of symlinks and rollbacks. Rollbacks should be handled by fixing your code or just simply rollback your code in git and deploy the last working version.

#### A lack of dependencies is king

I have seen other rails application roles but they all suffer similar problems. They make very opinionated decisions for you such as picking your database, web server, app server and so on. This isn't optimal because your rails app will likely have different needs than mine.

That is why I set this role up to do one thing and one thing only. It will take a rails app sitting somewhere, talk to a remote git repo, pull down a version of your choosing and then go through the motions of handling rails-specific things such as running database migrations and compiling assets. All of this is configurable.

There are no hard dependencies. As long as you overwrite a few default variables then you're free to use whatever you want to provision your server. I have created a few ansible roles that you can use in combination (or in total isolation if you wish) to form a rails app infrastructure with very little configuration required. We'll talk about that later tho.

## Role variables

Below is a list of default values along with a description of what they do.

```
# What is the name of your rails app? It should be lower case and a valid file name.
rails_deploy_app_name: app

# Which user account on the system will own the deployed files?
rails_deploy_user: deploy

# Where will the deployed application live on your server?
rails_deploy_path: /home/{{ rails_deploy_user }}/{{ rails_deploy_app_name }}.git

# The SSH keypair so that the server can pull in code from a remote git server.
# [more on this later in the readme]
rails_deploy_ssh_keypair_local_path: /home/yourname/dev/secrets/
rails_deploy_ssh_private_key_name: id_rsa
rails_deploy_ssh_public_key_name: id_rsa.pub

# Where is the remote repo? The default expects you to have SSH keys setup.
# The syntax for the URL is slightly different for github but you should know it.
rails_deploy_git_url: "git@bitbucket.org:yourusername/reponame.git"

# Which branch or tag should be used?
rails_deploy_git_version: master

# By default it expects you have ruby installed through rvm but this is not mandatory.
# Just replace this with the path to your ruby binary.
rails_deploy_shell_prefix: /usr/local/rvm/bin/rvm default do

# What command will start bundler? You shouldn't ever have to change this once you
# have setup the shell prefix above.
rails_deploy_bundle: "{{ rails_deploy_shell_prefix }} bundle"

# Where will the gems be installed to? This path is relative to your rails app by default
# but you can put an absolute path if you want.
rails_deploy_bundle_path: vendor/bundle

# A full list of possible environments that your app could run against. This is used by
# bundle --without and it will figure out which environments to remove for you.
rails_deploy_bundle_without:
  - development
  - staging
  - production
  - cucumber
  - test

# Should it run migrations and precompile assets?
rails_deploy_migrate: true
rails_deploy_precompile_assets: true

# This is the single app server that will run any command that issues a db related task.
# It will by default look for a group called [app] and then pick the first server in the list
# but you can change it to whatever you want.
rails_deploy_migrate_master_host: "{{ groups['app'][0] }}"

# You should set every single environment variable your app has in this list because it will be
# included in every command that requires your rails environment to be set.
rails_deploy_env:
  RAILS_ENV: production
  # FOO: bar
  # SOMETHING: else

# The amount in seconds to cache apt-update.
apt_cache_valid_time: 86400
```

## Example playbook

For the sake of this example let's assume you have a group called **app** and you have a typical `site.yml` file.

To use this role edit your `site.yml` file to look something like this:

```
---
- name: ensure app servers are configured
- hosts: app

  roles:
    # Insert other roles here to provision the server before deploying a rails app to it.
    - { role: nickjj.rails, tags: rails }
```

Let's say you want to edit a few defaults, you can do this by opening or creating `group_vars/all.yml` which is located relative to your `inventory` directory and then making it look something like this:

```
---
# Variables that could have been populated to satisfy other roles, it doesn't matter.
user_name: storm
secrets_load_path: /home/yourname/dev/secrets

# Overwrite a few rails deploy variables.
rails_deploy_app_name: testproj
rails_deploy_user: "{{ user_name }}"

rails_deploy_ssh_keypair_local_path: "{{ secrets_load_path }}"

rails_deploy_git_url: "git@bitbucket.org:yourname/testproj.git"

rails_deploy_migrate_master_host: "{{ groups['app'][0] }}"
```

#### What is the ssh keypair used for and how do you set it up?

The application servers should use ssh keys to pull code from your remote git repo. To do that successfully requires generating a public and private ssh keypair and placing it on each app server. This could have been generated automatically but if you had 10 app servers then you would end up with 10 different keypairs which means you would have to add 10 different deploy keys to github, bitbucket or whatever remote git host you're using.

To get around this you can generate a single keypair and place them in a local directory on the computer running ansible. This location should ideally be outside of version control so you don't commit sensitive information. This is a common trend I use for all secret data. Passwords, ssl certificates, ssh keys, etc..

The `secrets_load_path` is the local path where the keypair resides, by default it expects you name them `id_rsa` and `id_rsa.pub` but you can overwrite the file names.

To generate the keypair open a terminal and enter: `ssh-keygen -t rsa`.

You should use the default file names but make sure you do not save them in the default location because you could accidentally overwrite your usual keys. You should also keep the passphrase empty. Once you have the keypair then place them in the `secrets_load_path` path.

The last step you would need to do after creating your repo on github or bitbucket is to goto the admin area for this specific repo and add the public deploy key. Now each deployed app server is only capable of performing read-only actions on your repo. You do not have to worry about a potentially sinister digital ocean employee sabotaging your git repo.

#### Tracking migrations in other roles

This role will set a `rails_deploy_ran_migration` fact. You can use this to determine whether or not the database was migrated in the last deploy. For example in [nickjj.pumacorn](https://github.com/nickjj/ansible-pumacorn) I use this to decide if the server can be reloaded with no downtime or requires a full restart.

## Installation

`$ ansible-galaxy install nickjj.rails`

## Running adhoc commands

If you have migrations and/or precompile assets turned off and you want to handle these manually or perhaps you want to run arbitrary rake tasks then you can do that fairly easily thanks to ansible.

Since most commands would depend on your environment being set, this role outputs whatever you have set in `rails_deploy_env` to `/etc/default/APP_NAME` so you can `source` this file for all of your environment variables.

Here is an example of executing `rake db:version` on all **app** servers:

`ansible app -m shell -a 'executable=/bin/bash source /etc/default/testproj && cd /home/yourname/dev/testproj.git && /usr/local/rvm/bin/rvm default do bundle exec rake db:version' -u deploy -i /path/to/your/inventory/`

As you can see it's pretty long and annoying to type. I recommend creating a wrapper bash script to hide some of the complexity if you plan to run adhoc commands on a regular basis. Obviously you will need to replace certain things too if you happen to not be using rvm.

## Looking for an end to end solution?

Check out [rails-basic](https://github.com/nickjj/ansible-playbooks) to see how you can use this role along with a few other roles to create a typical rails application server stack.

## Requirements

Tested on ubuntu 12.04 LTS but it should work on other versions that are similar.

## Ansible galaxy

You can find it on the official [ansible galaxy](https://galaxy.ansible.com/list#/roles/866) if you want to rate it.

## License

MIT