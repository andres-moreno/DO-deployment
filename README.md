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

We first need to be able to have Node applications running on a
server, detached from a user, and proxied for security. Below are some
links to tutorials for Digital Ocean with my notes.

How to Set Up Nginx
-------------------

[Here][nginxDOsetup] is a writeup for how to set up Nginx for use on
Digital Ocean. And [here][NginxHTTPS] is where they tell us how to
secure our Nginx server.

What I did:

* Installed `nginx` with `sudo apt install nginx`
* Adjusted the firewall
    * `sudo ufw app list`: list of application profiles
    * `sudo ufw allow 'Ningx Full'`
    * `sudo ufw status`
* Check that all is well with `ngix`: `systemctl status nginx`
* Checked to see that my server was avaiable out in the internets
  (need to have `curl` installed but this works: `curl -4
  icanhazip.com`). Then try: `http://<server ip>`
* The DO nginx setup instructions give us a list of the commands one
  might neeed to manage nginx:
     * `sudo systemctl stop nginx`: stop the server
     * `sudo systemctl start nginx`: start the server
     * `sudo systemctl restart nginx`: restart the service
     * `sudo systemctl reload nginx`: reload without dropping
       connections
     * `sudo systemctl disable nginx`: disable enabling `nginx` at
       boot
     * `sudo systemctl enable nginx`: enable `nginx` at boot
* Set up a server block for `andresmoreno.me`. This is what it looks
  like out of the box without any changes (the default):
  
```bash
server {
        listen 80;
        listen [::]:80;

        root /var/www/example.com/html;
        index index.html index.htm index.nginx-debian.html;

        server_name example.com www.example.com;

        location / {
                try_files $uri $uri/ =404;
        }
}
```

Remarks:

1. This Nginx file listens on port 80, which means it only supports
   `http`.
2. It listens for `example.com` and `www.example.com`.
3. It serves files from `root`, which is set up to be
   `/var/www/example.com/html;` If it doesn't find the file or an
   `index` file per the line below the `root` configuration, it
   returns a `404`.
   
[nginxDOsetup]: https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-18-04 "Nginx setup for DO"
[NginxHTTPS]: https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-18-04 "Nginx HTTPS setup"

Locking Things Down
-------------------

1. We first need to install `certbot`:

     * Add respository:`sudo add-apt-repository ppa:certbot/certbot`
     * Install`sudo apt install python-certbot-nginx`
     * Test our `nginx` configuration: `sudo nginx -t`

2. Update `ufw`

     * `sudo ufw status`: check to see what's permitted
     * `sudo ufw allow 'Nginx Full'` (We did this already so we don't
       have to change anything
    
3. Obtain an SSL Certificate: we use the `nginx` plugin:

      `sudo certbot --nginx -d example.com -d www.example.com`

      Make sure to enable redirects to `https`.

4. Test to make sure the renewal of the certificate is working:

       `sudo certbot renew --dry-run`

Nginx Configuration File after Lock Down
----------------------------------------

After installing `certbot`, the file now looks like this:

```bash
server {

        root /var/www/andresmoreno.me/html;
        index index.html index.htm index.nginx-debian.html;

        server_name andresmoreno.me www.andresmoreno.me;

        location / {
                rewrite ^/blip/(.*)$ /blip/build/ last;
                rewrite ^/blop/(.*)$ /blop/build/ last;
                rewrite ^/test-app/(.*)$ /test-app/build/$1 last;
                try_files $uri $uri/ =404;
        }

        location /blip/build {
                try_files $uri $uri/ =404;
        }       

        location /blop/build {
                try_files $uri $uri/ =404;
        }       

        location /test-app/build {
                try_files $uri $uri/ =404;
        }       

    listen [::]:443 ssl ipv6only=on; # managed by Certbot
    listen 443 ssl; # managed by Certbot
    # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/andresmoreno.me/fullchain.pem; 
    # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/andresmoreno.me/privkey.pem; 
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
server {
    if ($host = www.andresmoreno.me) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    if ($host = andresmoreno.me) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


        listen 80;
        listen [::]:80;

        server_name andresmoreno.me www.andresmoreno.me;
    return 404; # managed by Certbot

}
```

The most salient things above are:

* Serving up certificates in the SSL configuration section
* The use of port 443 for *everything*: the second server listens on
  port 80 and then redirects all requests to `https`.
  
Serving React and NodeJS Applications
-------------------------------------

Below are some modifications to support reverse proxy of Node
applications and `rewrite` directives to support the deployment of
`React` applications built on the server and being served out of the
`build` directory

```bash
    location /apis {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
    location / {
        rewrite ^/test-app/(.*)$ /test-app/build/$1 last;
        try_files $uri $uri/ =404;
    }

    location /weather {
        try_files $uri $uri/ =404;
    }

    location /test-app/build {
        try_files $uri $uri/ =404;
    }
        
```

The `NodeJS` proxy line above enables the support of a `Node`
application being served out of port 3000 with routes under `/apis`
(e.g., `https://andresmoreno.me/apis/loc`)

The `rewrite` lines above address a particular set of requirements:

  * We want the `root` of our static server to be the
    `/var/www/andresmoreno.me/html` directory, i.e., we want to be
    able to deploy static web sites as subdirectories there and have
    everything work.
  * But we also want to be able to deploy `React` applications in
    their build directory (it is much easier to build them *in situ*),
    which means supporting a URI like
    `https://andresmoreno.me/my-app/` which will in turn be resolved
    as `/var/www/andresmoreno.me/html/my-app/build`, which corresponds
    to our `root` path with the name of the app, followed by
    `build`. But this is exactly what the line below does:

    `rewrite ^/test-app/(.*)$ /test-app/build/$1 last`


Serving NodeJS Applications in a DO Droplet
-------------------------------------------

We need to install `node`, `pm2`, and set up `nginx` with a reverse
proxy pointing to our `node` application (which we did above). The
detailed instructions can be found [here][DONodeApp]

[DONodeApp]: https://www.digitalocean.com/community/tutorials/how-to-set-up-a-node-js-application-for-production-on-ubuntu-18-04 "NodeJS app on DO droplet"

Install `node` with `nvm`
------------------------

The first step is to install `node` and `npm`. The best way to do this
is with `nvm`. All the gory details for `nvm` can be found
[here][nvm]. Look [here][nvmInstall] for clear installation
instructions. We can then manage our node versions with `nvm`

[nvm]: https://github.com/creationix/nvm/blob/master/README.md "NVM docs"
[nvmInstall]: https://nodesource.com/blog/installing-node-js-tutorial-using-nvm-on-mac-os-x-and-ubuntu/ "Install Node with nvm"

Install *pm2*
-------------

Use `npm` to install PM2:

`sudo npm install pm2@latest -g`

Note that we might not need the `sudo` for `node` that was installed
with `npm`. Annoying.

We now can start our application with

`pm2 start ./bin/www`

This will cause the execution of the `nodeJS` app in the
background. We can set up our droplet to run `pm2` at startup with

`pm2 startup systemd`

and we can *freeze* the list of jobs that `pm2` runs with

`pm2 save`

To remove `pm2` from startup, we type

`pm2 unstartup systemd`

DO has a page with more information on [working with services][SystemdServices]

[SystemdServices]: https://www.digitalocean.com/community/tutorials/systemd-essentials-working-with-services-units-and-the-journal "Systemd Services"


Serving NodeJS Applications in a DO Droplet
-------------------------------------------

We need to install `node`, `pm2`, and set up `nginx` with a reverse
proxy pointing to our `node` application. The detailed instructions
can be found [here][DONodeApp].

[DONodeApp]: https://www.digitalocean.com/community/tutorials/how-to-set-up-a-node-js-application-for-production-on-ubuntu-18-04 "NodeJS app on DO droplet"

Install `node` with `nvm`
------------------------

The first step is to install `node` and `npm`. The best way to do this
is with `nvm`. All the gory details for `nvm` can be found
[here][nvm]. Look [here][nvmInstall] for clear installation
instructions. We can then manage our node versions with `nvm`

[nvm]: https://github.com/creationix/nvm/blob/master/README.md "NVM docs"
[nvmInstall]: https://nodesource.com/blog/installing-node-js-tutorial-using-nvm-on-mac-os-x-and-ubuntu/ "Install Node with nvm"

Install *pm2*
-------------

Use `npm` to install PM2:

`sudo npm install pm2@latest -g`

Note that we might not need the `sudo` for `node` that was installed
with `npm`. Annoying.

We now can start our application with

`pm2 start ./bin/www`

This will cause the execution of the `nodeJS` app in the
background. We can set up our droplet to run `pm2` at startup with

`pm2 startup systemd`

and we can *freeze* the list of jobs that `pm2` runs with

`pm2 save`

To remove `pm2` from startup, we type

`pm2 unstartup systemd`

DO has a page with more information on [working with services][SystemdServices]

[SystemdServices]: https://www.digitalocean.com/community/tutorials/systemd-essentials-working-with-services-units-and-the-journal "Systemd Services"

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

Supporting Automatic Restarting of Node Applications
----------------------------------------------------

We want to have our Node applications automatically restarted on
reboot and whenever we push a new version of our code to the
server. We use `PM2` for this purpose. Incidentally, I really like
`nvm` for managing Node. 

