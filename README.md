Digital Ocean Deployment Notes
==============================

Objective
---------

Streamline the deployment of Node and React applications.

Requirements
------------

* Support local (development) and a remote (production) deployment
* Provide automatic restart in both environments
* Play well with Git
* Keep it simple: using a single small droplet
* Secure: lock the system tight and use standard tools as much as possible

Architecture Choices
--------------------

* Web server: `Nginx`, used for redirection (HTTPS & React apps), reverse-proxy of
  Node apps, serving static content, and potentially for load
  balancing
* Code deployment: `Git` to bare repositories on the VPS. Leverage Git
  Hooks to build React apps and to install Node dependencies for both
  React and Node apps
* Process management: `PM2` to support automatic restart of apps on
  reboot as well as restart on code changes for Node apps
* JS Package Manager: `yarn`
  
Recipe
======

We break down the process into 4 pieces

Node applications code deployment
---------------------------------

My current `NodeJS` serving model has me deploying code to a directory
in my home directory and using `pm2` to start the server with reverse
proxy provided by Nginx.

We got started by following the DO Community tutorial
[here][DoGitDeploy]. The writeup below is a summary of that tutorial
in a way that makes sense to me. The setup phase involves both the
remote and local machines:

*Remote Machine Setup*:

1. Create a `git` repository for `myapp` in the desired directory (say
   `/home/afm/myapp`), create a git repository under it, and
   initialize this git repository to `bare` with 
   
   ```bash
   mkdir myapp && cd myapp
   mkdir myapp.git && cd myapp.git && 
   git init --bare
   ```

2. Now create a `post-receive` file in the `hooks` directory of
   `myapp.git` with the following content 
   
   ```bash
   #!/bin/sh
   git --work-tree=/home/afm/myapp --git-dir=/home/afm/myapp/myapp.git checkout -f
   cd /home/afm/myapp
   yarn install
   ```

The above script will be run after the `git push` command to push out
new code and will result on the working code being placed in the
work-tree directory and installed

Make sure to `chmod +x post-receive` so that the script above can be
executed.

*Local Machine*

1. We need a good working `git` repository. For now, I use the
   `master` branch when pushing out code but maybe I'll push a `prod`
   branch to production and have the hook check it out on the VPS
2. Add a remote: 

   `git remote add live ssh://afm@andresmoreno.me/home/afm/myapp/myapp.git`
   
   and check to make sure it's there with `git remote -v`
3. Now we are ready: we can push code changes with `git push live
   master`. Optionally, we can add a deploy script to our
   `package.json` file with the above command in order to be able to
   type at the command line `yarn deploy`
   
At this point we should be able to update the code on the remote
machine whenever we want.

[DoGitDeploy]: https://www.digitalocean.com/community/tutorials/how-to-set-up-automatic-deployment-with-git-with-a-vps "Git DO Deploy"
