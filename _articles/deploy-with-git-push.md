---
title:  "Deploy with git push"
description: How to automatically deploy (ie publish) to a remote server with a simple git push.
date: 2016-06-15T14.36.55+02:00
---

* TOC
{:toc}

## Introduction

Wouldn't it be nice to publish a website with a simple git push?  
Here I'll explain you how I usually do this.

I assume that the environment of the staging/production servers are the following:

- Operating system: Ubuntu 14.04 or Ubuntu 16.04

If your servers are not the above, the approach described in this document should work with minor changes given that they are *nix.


## Configuration

Let's assume that:

- the remote server is available at the address <input type="text" id="dwgp-serveraddress" value="www.example.com" />
- you want to create a new repository named <input type="text" id="dwgp-reponame" value="MYSITE" />
- the web server impersonates the user <input type="text" id="dwgp-webuser" value="www-data" /> in the group <input type="text" id="dwgp-webgroup" value="www-data" />
- you want to execute these commands after the push:
  - <label><input type="checkbox" id="dwgp-x-composer-nocache" />`Composer` (with cache disabled)</label>


## One-time server setup

### Install required packages

First of all, you need to install git on the server.  
You can do this by running the following command:

```bash
sudo apt-get update
sudo apt-get install -y git
```

If the above command fails with an error like `Unable to locate package git`, you can try this:

```bash
sudo apt-get install -y git-core
```

<div class="dwgp-x-composer-nocache-on" markdown="1" style="display: none">
### Install and configure Composer

In order to have Composer, you need PHP and some PHP extension.
On Ubuntu you can install all of them with:

```bash
sudo apt-get update && sudo apt-get install -y php-cli php-json php-mbstring
```

Then you can download Composer with this command:

```bash
# Go to your home directory
cd
# Download and install Composer
curl --silent --show-error https://getcomposer.org/installer | php
# Move Composer to the default binary folder
sudo mv composer.phar /usr/local/bin/composer
# Check that Composer works
composer --version
```

In order to save some space, let's add a script that execute Composer with cache disabled.

```bash
sudo nano /usr/local/bin/composer-without-cache
```

And type this content:

```bash
#!/bin/bash
export COMPOSER_CACHE_DIR=/dev/null
composer "$@"
```
</div>

Finally, make it executable:

```bash
sudo chmod a+x /usr/local/bin/composer-without-cache
```

### Create the `git` user

We need to create a user account on the server. This account will be the one used by the publish process.

With the following command we create that account:

<div class="language-bash highlighter-rouge">
    <pre class="highlight"><code>sudo adduser --gecos Git --disabled-login --disabled-password --shell /usr/bin/git-shell --home /home/git --ingroup <span class="dwgp-webgroup"></span> git</code></pre>
</div>

Here's the explanation of the above options:

- `--gecos Git`: set the full name of the account to `Git` (this essentially in order to avoid asking useless data like the account room number and work/home phone)
- `--disabled-login`: the user won't be able to use the account until the password is set.
- `--disabled-password`: disable the login using passwords (we'll access the system with SSH RSA keys)
- `--shell /usr/bin/git-shell`: when the user access the system, he will use the fake shell provided by git
- `--home /home/git`: set the home directory for the user to `/home/git`
- <code class="highlighter-rouge">--ingroup <span class="dwgp-webgroup"></span></code>:  add the new user to the <code class="highlighter-rouge"><span class="dwgp-webgroup"></span></code> group instead of the default one
- `git`: the username of the new account

#### Strengthen login security

We then need to configure the git shell.  
In order to improve the security of unwanted logins and abort shell sessions, we create a file that is executed when the `git` user logs in the shell and that will abort the session.  

```bash
sudo mkdir /home/git/git-shell-commands
sudo nano /home/git/git-shell-commands/no-interactive-login
```

In the editor type in these commands:

```bash
#!/bin/sh
printf '%s\n' "Hi $USER! You've successfully authenticated, but I do not"
printf '%s\n' "provide interactive shell access."
exit 128
```

To save the new file hit `CTRL`+`o` then `ENTER`.  
To quit the editor, hit `CTRL`+`x`.

We finally have to make the file executable:

```bash
sudo chmod +x /home/git/git-shell-commands/no-interactive-login
```

#### Allow <code class="highlighter-rouge"><span class="dwgp-webuser"></span></code> impersonation

The `git` user needs to be able to publish files acting like <code class="highlighter-rouge"><span class="dwgp-webuser"></span></code>.  
In order to allow this, run this command:

```bash
sudo visudo
```

Go to the end of the editor contents and add these lines:

<div class="highlighter-rouge">
    <pre class="highlight"><code>Defaults!/usr/bin/git env_keep="GIT_DIR GIT_WORK_TREE"
git ALL=(<span class="dwgp-webuser"></span>) NOPASSWD: /usr/bin/git</code><span class="dwgp-x-composer-nocache-on" style="display: none">, /usr/local/bin/composer-without-cache</span></pre>
</div>

The first line tells the system that when the `git` command is executed with a `sudo`, we need to keep the two environment variables `GIT_DIR` and `GIT_WORK_TREE`.  
The second line tells the system that the `git` user can execute the `git` command acting as <code class="highlighter-rouge"><span class="dwgp-webuser"></span></code> without any further authentication.


## Authorized developers

Every developer that should be able to publish needs an SSH-2 RSA key pair.  
It's possible (and recommended) to use a different key for every developer.

### Create the keys under Windows

I order to create the key pair, you can use PuTTYgen.  
If you already installed [TortoiseGit](https://tortoisegit.org/) you should already have this command, otherwise you can [download it](http://www.chiark.greenend.org.uk/~sgtatham/putty/download.html).

So, open PuTTYgen and:

- in the `Parameters` page be sure to check `SSH-2 RSA` and `2048` for the `Number of bits in a generated key`
- Hit the `Generate` button and move randomly your mouse over the PuTTYgen window
- In the `Key comment` field enter `ServerName - DeveloperName` (where `ServerName` is the name of the server where the key will be used, and `DeveloperName` is the name of the developers owning the key)
- In the `Key passphrase` and `Confirm passphrase` fields enter a password of your choice
- Hit the `Save private key` button to save the private key to file
- Copy the contents of the `Public key for pasting into OpenSSH authorized_keys file` and save it: this is the public key that we'll need.


### Create the keys under *nix

Simply run this command:

```bash
ssh-keygen -t rsa -b 2048 -f key-for-git -C 'ServerName - DeveloperName'
```

Where:

- `-t rsa`: we want public/private key pair in the SSH-2 RSA format
- `-b 2048`: we want 2048 bits in the key
- `-f key-for-git`: this is the name of the file that will contain the private key
- `-C 'ServerName - DeveloperName'`: this is the comment to associate to the key
  (it's a good practice to specify the name of the server and the one the developer owning the key)

Once you run the `ssh-keygen` command and specified a password of your choice, you'll have these two files:

- `key-for-git`: contains the private key
- `key-for-git.pub`: contains the public key


### Allow the developer to publish to the server

Login to the server and run this command:

```bash
sudo mkdir /home/git/.ssh
sudo nano /home/git/.ssh/authorized_keys
```

Go to the end of the editor contents and add a new line containing the previusly generated public key.  

The public key is a single line that starts with `ssh-rsa `, followed by a quite long list of characters and ending with the `ServerName - DeveloperName` comments specified during the creation of the key.



## Create a new repository on the server

We'll end up with:

- Directory to be published: <code class="highlighter-rouge">/var/www/<span class="dwgp-reponame"></span></code>
- Repository directory: <code class="highlighter-rouge">/var/git/<span class="dwgp-reponame"></span></code>

First of all, we create the directory that will contain web site (it will be owned by the <code class="highlighter-rouge"><span class="dwgp-webuser"></span></code> user):

<div class="highlighter-rouge">
    <pre class="highlight"><code>sudo mkdir -p /var/www/<span class="dwgp-reponame"></span>
sudo chown -R <span class="dwgp-webuser"></span>:<span class="dwgp-webgroup"></span> /var/www/<span class="dwgp-reponame"></span>
sudo chmod u+rw -R /var/www/<span class="dwgp-reponame"></span></code></pre>
</div>

Then we create the directory that will contain the bare repository data:

<div class="highlighter-rouge">
    <pre class="highlight"><code>sudo mkdir -p /var/git/<span class="dwgp-reponame"></span>.git
cd /var/git/<span class="dwgp-reponame"></span>.git
sudo git init --bare
sudo git config core.sharedRepository group</code></pre>
</div>

The `core.sharedRepository group` option of the git repository is needed in order to grant write access to both the `git` and <code class="highlighter-rouge"><span class="dwgp-webuser"></span></code> users (they both belong to the same user group - <code class="highlighter-rouge"><span class="dwgp-webgroup"></span></code>).

And now the key concept of this whole approach: when someone pushes to this repository, we checkout the repository to the publish folder:

<div class="highlighter-rouge">
    <pre class="highlight"><code>sudo nano /var/git/<span class="dwgp-reponame"></span>.git/hooks/post-receive</code></pre>
</div>

In the editor type these lines:

<div class="highlighter-rouge">
    <pre class="highlight"><code>#!/bin/bash
currentUser=`whoami`
currentServer=`hostname`
repoDirectory=/var/git/<span class="dwgp-reponame"></span>.git
pubDirectory=/var/www/<span class="dwgp-reponame"></span>
echo "Hello! I'm $currentUser at $currentServer"
echo "I'm going to publish from"
echo "   $repoDirectory"
echo "to"
echo "   $pubDirectory"
rc=0
sudo -u <span class="dwgp-webuser"></span> git --git-dir=$repoDirectory --work-tree=$pubDirectory checkout master -f
if [ "$?" -ne "0" ]; then
    echo "GOSH! GIT FAILED!!!!"
    rc=1
fi
<div class="dwgp-x-composer-nocache-on" style="display: none">if [ $rc -eq 0 ]; then
    echo "Changing directory"
    pushd $pubDirectory
    if [ "$?" -ne "0" ]; then
        echo "GOSH! PUSHD FAILED!!!!"
        rc=1
    else
        echo "Running composer install"
        sudo -u <span class="dwgp-webuser"></span> composer-without-cache install \
            --prefer-dist --no-dev --no-progress --no-suggest --no-ansi --no-interaction
        if [ "$?" -ne "0" ]; then
            echo "GOSH! COMPOSER FAILED!!!!"
            rc=1
        else
            echo "Great! Composer succeeded!"
        fi
        popd
   fi
fi</div>
if [ $rc -eq 0 ]; then
    echo "Don't worry, be happy: everything worked out like a charm ;)"
fi
exit $rc
</code></pre>
</div>

We finally need to update the permissions of the newly created directory:

<div class="highlighter-rouge">
    <pre class="highlight"><code>sudo chown -R git:<span class="dwgp-webgroup"></span> /var/git/<span class="dwgp-reponame"></span>.git
sudo chmod -R ug+rwX /var/git/<span class="dwgp-reponame"></span>.git
sudo chmod -R ug+x /var/git/<span class="dwgp-reponame"></span>.git/hooks/post-receive
</code></pre>
</div>


## Push-to-publish

Everything is almost ready!
The only step required to deploy with a simple `git push` is to add the remote to your repository.

For instance, here's how to add a remote named `deploy` to your local repository:

<div class="highlighter-rouge">
    <pre class="highlight"><code>git remote add deploy git@<span class="dwgp-serveraddress"></span>:/var/git/<span class="dwgp-reponame"></span>.git</code></pre>
</div>

When you push to the `deploy` remote, the published directory willbe updated automatically.

Nice, isn't it?

{% include gitter.html handle="deploy-with-git-push" %}

<script>
$(document).ready(function() {
    var storage = (function() {
        var PREFIX = 'ml-dwgp-';
        var ok = window.localStorage && window.localStorage.setItem && window.localStorage.getItem;
        return {
            save: function (key, value) {
                if (ok) {
                    try {
                        window.localStorage.setItem(PREFIX + key, value);
                        return true;
                    } catch (e) {
                    }
                }
                return false;
            },
            load: function (key, defaultValue) {
                var result = defaultValue;
                if (ok) {
                    try {
                        var v = window.localStorage.getItem(PREFIX + key);
                        if (v !== null) {
                            return v;
                        }
                    } catch (e) {
                    }
                }
                return defaultValue;
            }
        };
    })();
    function Valorizer(key) {
        var me = this;
        me.currentValue = null;
        me.$input = $('#dwgp-' + key);
        me.type = 'text';
        me.saveEvent = 'blur';
        switch (key) {
            case 'reponame':
                me.normalize = function (v) { return v.replace(/[\\\/]+/g, '/').replace(/^\/|\/$/, '').replace(/[^\w\.\/]+/g, '-').replace(/^-+|-+$/g, ''); };
                break;
            case 'serveraddress':
                me.normalize = function (v) { return v.replace(/\s+/g, ''); };
                break;
            case 'webuser':
            case 'webgroup':
                me.normalize = function (v) { return v.replace(/[^\w\-]+/g, ''); };
                break;
            case 'x-composer-nocache':
                me.type = 'checkbox';
                me.saveEvent = 'change';
                break;
            default:
                me.normalize = function (v) { return v; };
                break;
        }
        switch (me.type) {
            case 'checkbox':
                me.$spans = {
                    on: $('.dwgp-' + key + '-on'),
                    off: $('.dwgp-' + key + '-off')
                };
                break;
            default:
                me.$spans = $('.dwgp-' + key);
                break;
           }
        me.$input
            .on('change keydown keypress keyup mousedown mouseup blur', function() {
                var newValue;
                switch (me.type) {
                    case 'checkbox':
                        newValue = me.$input.is(':checked') ? 'on' : 'off';
                        break;
                    default:
                        newValue = me.normalize(me.$input.val());
                        break;
                }
                if (newValue === '' || newValue === me.currentValue) {
                    return;
                }
                me.currentValue = newValue;
                switch (me.type) {
                    case 'checkbox':
                        me.$spans.off[newValue === 'off' ? 'show' : 'hide']();
                        me.$spans.on[newValue === 'on' ? 'show' : 'hide']();
                        break;
                    default:
                        me.$spans.text(newValue);
                        break;
                }
            })
            .on(me.saveEvent, function() {
                setTimeout(function() {
                    if (me.currentValue !== null) {
                        storage.save(key, me.currentValue);
                    }
                }, 0);
            })
        ;
        switch (me.type) {
            case 'checkbox':
                me.$input.prop('checked', storage.load(key, me.$input.is(':checked') ? 'on' :'off') === 'on');
                break;
            default:
                me.$input.val(storage.load(key, me.$input.val()))
                break;
        }
        me.$input.trigger('change');
    }
    for (var i = 0, L = ['reponame', 'serveraddress', 'webuser', 'webgroup', 'x-composer-nocache']; i < L.length; i++) {
        new Valorizer(L[i]);
    }
});
</script>
