## Initial Setup

1. Generate a database dump, see [this article](https://app.nuclino.com/Univers-Labs/Development/Creating-a-DB-dump-dbbc4fcc-b2ed-4eef-8538-475f27db86ca) for specific instructions. 
The current staging app is `financial-edge-staging` and the prod is `financial-edge-prod`. 
2a. Create a `docker-compose.override.yml` file with the following contents if you only want to run Financial Edge locally:

```
version: "3.2"
services:
  wordpress:
    ports:
      - 80:5000
    environment:
      HOME_URL: http://financialedge.test
      SITE_URL: http://financialedge.test/wp
      OVERRIDE_SERVER_NAME: financialedge.test
      DISABLE_OPCACHE: 1
      DEACTIVATE_SSO: 1
      GO_URL: https://devgo.fe.training
      FELIX_URL: https://fe-felix-staging.universlabs.io
  mysql:
    ports:
      - 3306:3306

```

Then add `financialedge.test` to your `/etc/hosts`.

2b. Create a `docker-compose.override.yml` file with the following contents if you want to run Financial Edge and Felix locally (retrieve the missing `<value>` from [passphrase](https://phabricator.univ.rs/K629)):


```
version: "3.2"
services:
  wordpress:
    ports:
      - 81:5000
    environment:
      HOME_URL: http://localhost:81
      SITE_URL: http://localhost:81/wp
      OVERRIDE_SERVER_NAME: localhost:81
      DISABLE_OPCACHE: 1
      DEACTIVATE_SSO: 0
      GO_URL: https://devgo.fe.training
      FELIX_URL: http://localhost:82
      IDP_X509_CERT:<value>
  mysql:
    ports:
      - 3307:3306
```
 
3. Save SSO certificate text from [passphrase](https://phabricator.univ.rs/K630) to a file called `./cert/idp.crt` [Only required if using SSO]
4. Run `./setup-docker.sh` to get everything installed.
5. Run `docker-compose up mysql` to import the WordPress database.
6. Run `docker-sync start` to sync files, this will take 5-10 minutes the first time depending on your system specs.
7. See "Running In Development" to start everything.

## Running In Development

To start, in separate terminal windows:

1. Run `docker-sync start`
2. Run `./up.sh`
3. Run `yarn watch`

### SSO - Follow this if you have a dump from prod or staging with SSO enabled

4. Set `DEACTIVATE_SSO: 1` in the env in `docker-compose.override.yml` and run `./up.sh` to disable the `wp-saml-auth` plugin
5. Then set it back to `DEACTIVATE_SSO: 0` and re-run `./up.sh` to allow re-activating the plugin
6. Log in using these [credentials](https://phabricator.univ.rs/K580) and in Plugins, enable `wp-saml-auth` and configure the plugin to point to the relevant hosting sites and to point to an SSO idp. 


Use the following inputs if you want to connect the Felix and FE sites locally.

```
### Service Provider Settings
Base URl: http://localhost:81/wp
Entity Id (Required): http://localhost:81
Assertion Consumer Service URL (Required): http://localhost:81/wp/wp-login.php

### Identity Provider Settings
Entity Id (Required): http://localhost:80/sso/saml2/idp/metadata.php
Single SignOn Service URL (Required): http://localhost:80/sso/saml2/idp/SSOService.php
Single Logout Service URL: http://localhost:80/sso/saml2/idp/SingleLogoutService.php?ReturnTo=http://localhost:81
```

You will also need to activate and deactivate the plugin on the Felix site for it to connect locally. 

*n.b you need to set `DEACTIVATE_SSO` back to `0` if want to enable it, otherwise it will not be able to be activated*


To stop:

1. Run `docker-compose down`
2. Run `docker-sync stop`
3. Exit `yarn watch`

Notes:

- If you can't see changes you are making reflected on the site, this might be due to docker-sync. To verify this, bring down your containers and then run `docker-compose up wordpress` to run them without docker-sync. If you can see the changes now, this means docker-sync needs to be reset, do this by running `docker-sync clean` and `docker-sync start`, then bringing everything back up again as normal.

## Building

1. We want to build with Advanced Custom Fields Flexible Content, so this means using Twig in an interesting way. When you create your flexible content fieldset, we have a function in `traits/theme-functions.php` defined that goes and gets the Twig stuff from the `views/acf-flexible-content` folder. In there is a sample of how to structure your files.
2. `page.php` will use the `ul_modules` function (described above) to add the flexible content modules to the context. The PHP files therein will compile the twig and return them to the output buffer, which will then get rendered when the loop through the modules is complete.
3. Sounds complicated but once you see it in action, it's really easy.

## WordPress & Plugins

WordPress and the majority of plugins are installed through Composer.

To add a new plugin:

1. Check it's available through [WordPress Packagist](https://wpackagist.org/). Search for the plugin and check the correct plugin is available.
2. If it is available, install it with `docker run --rm -it -v $(pwd):/app -w /app -u 1000:1000 composer composer require wpackagist-plugin/<plugin-name> --ignore-platform-reqs`
3. Sometimes plugins aren't available through Composer (e.g. if they're premium versions), in this case manually include the plugin to `web/app/plugins`, then add an exclusion to `.gitignore` so the directory gets checked in: `!web/app/plugins/<plugin-name>`


---

# Empty Wordpress

1. Run `./init.sh` to generate a new project. The script will prompt you for the name of the project. Once complete, set up a Bitbucket repository and push to it.
2. Edit `/etc/hosts` with the `HOME_URL` value you saved to the `docker-compose.override.yml` file.
3. Follow **Setup** instructions below. You can skip the "generate a database dump" step, as a basic clean version of the WordPress database is included with this repo.
4. [WordPress Login Credentials](https://phabricator.univ.rs/K554) - *make sure you change these*.
5. Replace "Empty Wordpress" in this file with the repo name, and remove this block of steps from the README.
6. Follow [this guide](https://app.nuclino.com/Univers-Labs/Development/Connecting-your-WordPress-site-to-a-Google-Cloud-Storage-bucket-0ffaa6a6-7050-402c-9e0f-0a372679d76e) to set up a new Google Cloud Platform project to be used for storage.

*To update the clean database file, run the following with the container id of the running mysql container:*
`docker exec CONTAINER_ID /usr/bin/mysqldump -u financialedge --password=financialedge financialedge | gzip > ./mysqldumps/clean-database.sql.gz`

