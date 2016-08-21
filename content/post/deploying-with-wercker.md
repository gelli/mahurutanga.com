+++
categories = []
date = "2016-08-20T17:49:32+02:00"
tags = ["code"]
title = "Deploying Hugo websites with Wercker"

+++

I'm a big fan of "Done means it's online". To make sure this is also true for this blog, I wanted each commit to automatically trigger a deployment of the whole static page. Instead of using WordPress for this page I decided to go with [Hugo](https://gohugo.io), a static site generator. In this blog article I'm going to show how I set up [Wercker](https://wercker.com) to build the website and deploy it to my shared hosting webspace at 1&1.
<!-- more -->

## Prerequisites
In order for [Wercker](https://wercker.com) to pick up your code it needs to be hosted on either GitHub or BitBucket. Since I host all my code on GitHub this is what I'm using in this article.

You also need a [Wercker](https://wercker.com) account. Signing up is for free and you can either create one from scratch or use your GitHub account to sign up.

## Creating a new application in Wercker
After you logged in click on [Create Application](https://app.wercker.com/applications/create). Choose the GitHub repository that holds the source of your website.

![Create a new application: Step 1](/images/deploying-with-wercker/wercker-step-1.png)

On step 2, if you host your website in a public repository use the first option and let Wercker check out the repository without using SSH. In case you are using a private repository you need to choose one of the other options and add an SSH key to your GitHub account in order for Wercker to be able to access the repo.

![Create a new application: Step 2 and 3](/images/deploying-with-wercker/wercker-step-2-and-3.png)

Whether you make your app public or not on the last step is kind of a personal preference. I didn't find this option to be explained well so I left the checkbox unchecked.

Press finish to create the application.

## Adding a wercker.yml configuration
Wercker uses the build configuration added to a file called wercker.yml in your repository. Since this blog is open source you can find the [full wercker.yml](https://github.com/gelli/mahurutanga.com/blob/master/wercker.yml) for it in the [repository on GitHub](https://github.com/gelli/mahurutanga.com)

Wercker creates a new build environment (or box) on each run of the build. So the first step in the configuration is to choose a base box to start with. Since the build needs some additional git related stuff I'm not using the default "debian" box.

```yaml
box: switchback/git-debian
```

Next we're configuring the build steps. Wercker provides a [registry](https://app.wercker.com/explore) to search for predefined build steps to use in your wercker.yml. There is already one for building Hugo websites called "arjen/hugo-build" which can be used. Adding this step to the build configuration will take care of starting Hugo and building the static files from the sources.

![Wercker Registry](/images/deploying-with-wercker/wercker-registry.png)

If you want do build or deploy other applications with wercker, the registry on the website provides a search interface for various boxes and build steps from Webpack to Golang.

Since for this blog I'm using a git submodule to include the theme, I added a "script" step that makes sure that this submodule is present during the next build steps. Here's the build part of the wercker.yml:

```yaml
build:
  steps:
    - script:
        name: initialize git submodules
        code: |
            git submodule update --init --recursive
    - arjen/hugo-build@1.11.0
```

## Triggering the first build
Commit the wercker.yml and push it to GitHub. Wercker should automatically start a build. Alternatively you can start a new build on the website.
![Wercker Build](/images/deploying-with-wercker/wercker-build.png)

## Deploying to a shared hosting webspace
Your Hugo website should be generated and the first build finish successfully. Now we need to push the generated files to the webspace. I'm currently still hosting this blog at 1&1 on a shared hosting package so I'm going to push the files via SSH and SCP to the target location. This will require a couple of manual steps to make sure Wercker has access to the webspace.

### Generate SSH keys
First, open your application on the Wercker website and go to Environment. Generate a new SSH key pair and name it "DEPLOY". If you prefer to choose a different name make sure to also adjust it in the wercker.yml in the later steps.

Add the public key of the generated key pair to your .ssh/authorized_keys file on the target server.
![Create an SSH key](/images/deploying-with-wercker/wercker-ssh-key.png)

### Environment Variables
You can configure additional environment variables that Wercker has access to during the build steps. In order to keep the wercker.yml free from confidential data, we're going to add environment variables for the server hostname, user and deployment directory to the application settings that will be used in the wercker.yml deploy step.

* HOSTNAME: the server hostname where to push the files to
* USER: the username to use when connecting via SSH
* DEPLOY_PATH: The directory on the server where the files should be copied to

![Environment variables](/images/deploying-with-wercker/wercker-env-vars.png)

## Add deploy configuration to wercker.yml

The deploy configuration requires several steps to prepare copying the files via ssh. First, since every build happens in a new box, the target server needs to be added to the ssh known hosts. Here I'm using the environment variables that I configured in the last step to add the server hostname.

```
deploy:
    steps:
        - add-to-known_hosts:
            hostname: $HOSTNAME
```

Now we're copying the generated ssh key into a temporary file so ssh can use it to connect to the server. Make sure to enter the correct environment variable for the content parameter, if you used a different one in the ssh key step.
```
- mktemp:
    envvar: PRIVATEKEY_PATH
- create-file:
    name: write key
    filename: $PRIVATEKEY_PATH
    content: $DEPLOY_PRIVATE
    overwrite: true
```

With the ssh key in place the script steps can access the server, delete the old files and sync the new files from the current build.

```
- script:
    name: remove old files
    code: ssh -i $PRIVATEKEY_PATH -l $USER -o UserKnownHostsFile=/dev/null \ 
        -o StrictHostKeyChecking=no $HOSTNAME rm -rf $DEPLOY_PATH/*
- script:
    name: transfer application
    code: |
        scp -r -i $PRIVATEKEY_PATH public/* $USER@$HOSTNAME:$DEPLOY_PATH/
```

That's it. Commit the wercker.yml and push it to GitHub. You can find the [full wercker.yml](https://github.com/gelli/mahurutanga.com/blob/master/wercker.yml) for this website in the [repository on GitHub](https://github.com/gelli/mahurutanga.com)

### Add deploy step to build pipeline
The last step is to add the configured deploy step to the Wercker workflow pipline. Open your application on the Wercker website, click on "Workflows" and add a new pipeline. Choose a name and use "deploy" as YML Pipeline name.

![Wercker Pipeline](/images/deploying-with-wercker/wercker-pipeline.png)

Add the created deploy step to the workflow pipeline.

![Wercker Pipeline](/images/deploying-with-wercker/wercker-workflow.png)

That's it. Push an update of your website to GitHub to trigger a new build. Wercker will pick it up, generate the static files and automatically copy them over to your server.

It's not perfect, yet since e.g. the website is unreachable for a short time after deleting the old files and copying the new. I will switch the deployment to rsync on a later step and maybe add a real atomic switch via symlinks.
