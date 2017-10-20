---
author:
  name: Linode Community
  email: docs@linode.com
description: 'Wercker allows you to set up automation pipelines for your apps with only a single configuration file. This guide will explain the basics of the wercker.yml file and demonstrate several basic workflows.'
keywords: 'wercker,docker,development'
license: '[CC BY-ND 4.0](https://creativecommons.org/licenses/by-nd/4.0)'
published: 'Tuesday, October 17th, 2017'
modified: 'Wednesday, October 18th, 2017'
modified_by:
  name: Linode
title: 'How to develop and deploy your applications using Wercker'
contributor:
  name: Damaso Sanoja
external_resources:
 - '[Wercker Developer Documentation](http://devcenter.wercker.com/docs/home)'
---

*This is a Linode Community guide. [Write for us](/docs/contribute) and earn $300 per published guide.*

---

Wercker is a software automation tool that aims to improve the Continuous-Integration and Continuous-Delivery (CI/CD) process of developers and organizations. It enables the creation of automated workflows, or *pipelines*, which specify a series of tasks or commands that are run on your code whenever a change is pushed to the source repository.

This guide will use three example Go apps to demonstrate the basics of Wercker setup and configuration, and show how these can be used to create different kinds of workflows.

## Prerequisites

In order to complete this guide, you will need the following, or equivalent:

  - A Github account (similar procedures should work on Bitbucket or any git variant)
  - A Docker account (other services might work with minor changes)
  - Ubuntu 16.04

## Before You Begin

1.  Complete the [Getting Started](/docs/getting-started) guide.

2.  Follow the [Securing Your Server](/docs/security/securing-your-server/) guide to create a standard user account, harden SSH access and remove unnecessary network services.

    This guide will use `sudo` wherever possible.

3.  Update your packages:

        sudo apt update && sudo apt upgrade

4. This guide requires Docker installed in your Linode. See our [How to Install Docker and Pull Images for Container Deployment](https://www.linode.com/docs/applications/containers/how-to-install-docker-and-pull-images-for-container-deployment) guide for more information.

## Set Up Demo Applications and Version Control

1.  Install `git` on both your local machine and on your Linode:

        sudo apt install git

2.  Sign in to Github and fork the following repositories:

    * [jClocksGMT](https://github.com/mcmastermind/jClocksGMT), a very basic jQuery collection of digital and analog clocks.
    * [golang/example](https://github.com/golang/example), a minimal set of `Go` examples made by `golang` Project.
    * [Getting Started golang](https://github.com/wercker/getting-started-golang), a sample `Go` application for Wercker.

3.  Clone your forks on your local machine:

        git clone https://github.com/<GITHUB_USER>/jClocksGMT.git
        git clone https://github.com/<GITHUB_USER>/example.git
        git clone https://github.com/<GITHUB_USER>/getting-started-golang.git


4.  Install Go on your system and make sure it is in your $PATH:

        sudo apt install golang-go
        go version

## Set Up a Wercker Account

1.  In a web browser, navigate to the [Wercker main page](http://www.wercker.com/) and sign up for a free account.

    ![Wercker Main Page](/docs/assets/wercker-main.png)

2.  The easiest way to register is using your GitHub account.

    ![Wercker Registration](/docs/assets/wercker-registration.png)

## Configuring wercker.yml

One of the advantages of Wercker is its simplicity. You only need one configuration file: `wercker.yml`. This file describes your automation pipelines through one or more steps. You can think of steps as calls to action processes and pipelines as collections of one or more steps.

### jClocksGMT Example

This example will demonstrate using Wercker to update the source code on a remote server whenever the repository on Github is updated. This is a common use case for static web sites: whenever you push to Github from your local machine, the code on the server where the site is hosted will automatically be updated as well.

1.  Create a `wercker.yml` file in the root of the `jClocksGMT` directory and paste in the content below. Replace `192.0.2.0` with the public IP address of your Linode, and update the last line to use the correct username and file path.

{: .file}
/path/to/jClocksGMT/wercker.yml
:   ~~~ yml
    box: debian
    # Build definition
    build:
      # The steps that will be executed on build
      steps:
        # Installs openssh client and server.
        - install-packages:
            packages: openssh-client openssh-server
        # Adds Linode server to the list of known hosts.
        - add-to-known_hosts:
                hostname: 192.0.2.0
                local: true
        # Adds the Wercker SSH key.
        - add-ssh-key:
                keyname: linode
        # Custom code to be executed on remote Linode
        - script:
            name: Update code on remote Linode
            code: |
              ssh username@<Linode IP or hostname> git -C /path/to/jClocksGMT pull
    ~~~

{:.caution}
> Make sure that you do not use any tabs inside your `wercker.yml` file; all indentention must be done with spaces.

Whenever a Wercker run is triggered (by a push to the repository), Wercker will load a Docker image and run the steps specified from that image. This is why all commands to be run on your Linode are prefaced with an `ssh` command. In this case, the `wercker.yml` file contains the following steps:

- `box`: Defines the Docker image to be used. In this case a global Debian image is specified.
- `install-packages`: this step is a shortcut to `apt-get install`. All packages listed will be installed in your container.
- `add-to-known_hosts`: this is a self-explanatory step that adds the Linode IP or domain name to the known hosts file.
- `add-ssh-key`: this step adds the Wercker generated Public SSH key to your container.
- `script`: scripts are custom steps that can execute almost any command, in this case on your remote Linode.

Put together, Wercker will do the following every time it is run:

1. A Debian image is loaded in the container.
2. The necessary packages `openssh-client` and `openssh-server` are installed.
3. The SSH connection between the Wercker container and your Linode is set up.
4. The Debian container runs a `git pull` command from your remote Linode.

In other words, the `build` pipeline updates the remote repository on your Linode.

### Hello.go Example

This example demonstrates a more complicated pipeline with both `build` and `deploy` pipelines. This time, Wercker will build a simple Go application and deploy it to DockerHub, then deploy the image from DockerHub to your remote Linode.

1.  Go to the root folder of your `example` fork and copy the `hello.go` file there:

        cp ./hello/hello.go .

2.  Create the `wercker.yml` file on the same root folder:

  {: .file}
  /path/to/example/wercker.yml
  :   ~~~ yml
      box: google/golang

      build:

          steps:
          # Sets the go workspace and places you package
          # at the right place in the workspace tree
          - setup-go-workspace

          # Build the project
          - script:
              name: Build application
              code: |
                  go get github.com/<user>/example
                  go build -o myapp

          - script:
              name: Copy binary
              code: |
                cp myapp "$WERCKER_OUTPUT_DIR"

      ### Docker Deployment
      deploy:

          #This deploys to DockerHub
          steps:
          - internal/docker-scratch-push:
              username: $DOCKER_USERNAME
              password: $DOCKER_PASSWORD
              repository: <docker-username>/myapp
              cmd: ./myapp

      ### Linode Deployment from Docker
      linode:

          steps:
          # Installs openssh client and other dependencies.
          - install-packages:
              packages: openssh-client openssh-server
          # Adds Linode server to the list of known hosts.
          - add-to-known_hosts:
              hostname: 192.0.2.0
              local: true
          # Adds SSH key created by Wercker
          - add-ssh-key:
              keyname: linode
          # Custom code to pull image
          - script:
              name: pull latest image
              code: |
                  ssh username@192.0.2.0 docker pull <docker-username>/myapp:latest
                  ssh username@192.0.2.0 docker tag <docker-username>/myapp:latest <docker-username>/myapp:current
                  ssh username@192.0.2.0 docker rmi <docker-username>/myapp:latest
      ~~~

This configuration file is more complex than the last one. There are three pipelines:

1. `build`: the obligatory pipeline that is used in this case to build your application. Since this second example uses a `Go` language the most convenient `box` is the official `google/golang` that comes with the necessary tools configured. The steps performed by this pipeline are:
- `setup-go-workspace`: prepares your `Go` environment.
- `Build application`: runs the actual building process for your sample application named `myapp`. The application is saved in the corresponding workspace.
- `Copy binary`: remember that you are working on a temporary pipeline, this step saves your application binary as a predefined environmental variable called `$WERCKER_OUTPUT_DIR` so that it is available for use in the next pipeline.

2. `deploy`: this pipeline will take your binary from `$WERCKER_OUTPUT_DIR` and then push it to your Docker account.
- `internal/docker-scratch-push`: this special step makes all the magic, using the environmental variables `$DOCKER_USERNAME` and `$DOCKER_PASSWORD` saves your binary to a light-weight `scratch` image. The `repository` parameter specifies the desired Docker repository to use.

3. `linode`: the fist three steps, `install-packages`, `add-to-known_hosts` and `add-ssh-key` were explained in the previous example, they are responsible for the SSH communication between your pipeline's container and your Linode. The custom script `pull latest image` is actually more interesting to analyze:
- Second line pulls (from Docker Hub) your most recent image build. By default this image is tagged "latest" unless specified otherwise.
- Third line clones your latest image and tag it "current".
- Four line removes the pulled image tagged "latest" in preparation for your next update.
That is one simple way to have a "current" application running all the time. Remember this is only an example you can define your own process.

{: .note}
>
>This example uses a very basic application, remember that statically building your application could be needed in real life production.

### Getting Started golang Example

The last example will introduce the **Wercker CLI**. This tool requires that Docker be installed on your local machine. You can use the same guide as for your Linode [here](https://www.linode.com/docs/applications/containers/how-to-install-docker-and-pull-images-for-container-deployment)

1.  Install Docker Machine:

        curl -L https://github.com/docker/machine/releases/download/v0.12.2/docker-machine-`uname -s`-`uname -m` >/tmp/docker-machine && chmod +x /tmp/docker-machine && sudo cp /tmp/docker-machine /usr/local/bin/docker-machine

2.  Install the Wercker CLI:

        sudo curl -L https://s3.amazonaws.com/downloads.wercker.com/cli/stable/linux_amd64/wercker -o /usr/local/bin/wercker
        sudo chmod 777 /usr/local/bin/wercker

3.  Check that the CLI is correctly installed:

        wercker version

4.  If the above command fails, you may also need to add `/usr/local/bin` to your `$PATH` variable:

        echo 'PATH=/usr/local/bin:$PATH' >> ~/.bash_profile

5.  Navigate to your fork root directory:

        cd /path/of/your/getting-started-golang

  A `wercker.yml` file should already be present:

  {: .file}
  /path/to/getting-started-golang/wercker.yml
  :   ~~~ yml
      box:
      id: golang
      ports:
          - "5000"

      dev:
      steps:
      - internal/watch:
          code: |
            go build ./...
            ./source
          reload: true

      # Build definition
      build:
          # The steps that will be executed on build
          steps:

          # golint step
              - wercker/golint

          # Build the project
              - script:
                  name: go build
                  code: |
                    go build ./...

      # Test the project
      - script:
          name: go test
          code: |
            go test ./...
      ~~~


  Only two pipelines are defined in this `yml` file: `dev` and `build`. Note that in this example port 5000 is exposed.

  - `dev`: this is an special type of pipeline, that can only be used locally, and serves the purpose of application testing.
  - `internal/watch` this step is watching for source code changes, and if any occurs then builds your application again (reload: true). This is very useful for debugging processes.
  - `build`: this was usually your first step for building your application. But now you can use it to check your code for any errors during the `build` step (wercker/golint).

## Adding your applications to Wercker

As the name implies, Wercker applications correspond to each of your projects. Because of its versioning scheme, you can think of each application as a specific working repository.

1.  Register the three forked repositories. Click the "plus" button in the upper right:

  ![Application creation](/docs/assets/wercker-app-button.jpg)

2.  Select the first example repository: <GITHUB_USER>/jClocksGMT, and click the button.

  ![Choose repository](/docs/assets/wercker-choose-repository.png)

3.  Configure access to your repository. If the project doesn't use submodules, the recommended option is usually the best choice.

  ![Configure access](/docs/assets/wercker-conf-access.jpg)

4.  Choose whether your application should be private (default) or public. Mark the example as public and click the Finish button.

  A greeting message indicates you are almost ready to start building your application and offers the option to start a wizard to help you create the application `wercker.yml` file, but that won't be necessary because you already did that in the previous section.

  ![YML Wizard](/docs/assets/wercker-yml-wizard.jpg)

5.  Repeat the same procedure for the other two example projects.

### Configure Applications

#### jClocks Example
If you remember your configuration files, you have several environmental variables to setup.

1.  For the first example you need a SSH key pair for communication with your Linode. Click on the "Environment" tab:

  ![jClocksGMT variables](/docs/assets/wercker-global-environment-variables.jpg)

2.  Click on "Generate SSH Keys". A pop-up window will appear asking for a key name (you need to use the same name used in your `wercker.yml` file: "linode"). Choose 4096 bit encryption:

![jClocksGMT SSH Keys](/docs/assets/wercker-ssh-key-creation.jpg)

  This will generate a key pair: `linode_PUBLIC` and `linode_PRIVATE`. The suffix is automatically added and is not needed in the configuration file.

3.  Copy the `linode_PUBLIC` key into your Linode. If your terminal application supports copy and paste, you can use CTL-C and CTL-V to copy the text from the Wercker dashboard into `~/.ssh/authorized_keys` on your Linode. If not, you can copy the key to your local machine and copy it from there to your remote server:

        cat ~/.ssh/jclock.pub | ssh root@<Linode IP or hostname> "mkdir -p ~/.ssh && cat >>  ~/.ssh/authorized_keys"

4.  Your first example is ready to be deployed: the application is configured on Wercker and your local repository contains the `wercker.yml` file explaining the steps to be executed. To trigger the automation you just need to commit your changes and watch then working on Wercker's **Runs** tab. Run the following command from the root of the jClocksGMT fork:

        git add . && git commit -m "initial commit" && git push origin master

5.  An animation will show each step's progress, allowing you to debug any problems. In this case, the build will fail:

  ![jClocksGMT build error](/docs/assets/wercker-jclocks-error-01.jpg)

  The hint says "Update code on remote Linode failed." Click on the build pipeline for more information:

  ![jClocksGMT build error](/docs/assets/wercker-jclocks-error-02.jpg)

6. This shows that the process failed at the step "Update code on remote Linode." This is due to the fact the repository was not cloned on the remote Linode in the first place. Connect to your Linode and clone the repository in the appropriate location, then return to the Wercker dashboard and click the "Retry" button:

  ![jClocksGMT build error](/docs/assets/wercker-jclocks-retry.jpg)

  This time the run should succeed, and your remote Linode repository will be updated. This basic example can be useful for many scenarios. If you are hosting a static website you can configure Wercker for updating your remote server each time you commit a change (an article) taking from you the manual work of doing so.

#### Hello.go Example

Click on "Workflows" tab in the Wercker dashboard. The editor will show a single pipeline, `build`, created automatically by Wercker. This example requires additional pipelines which you will have to create manually.

1. Click on the "Add new pipeline" button:

    ![Add pipeline](/docs/assets/wercker-add-pipeline.jpg)

2.  Fill in the fields as follows:

    - Name: You can use any name to describe your pipeline, for this example `deploy-docker` and `deploy-linode` will be used.
    - YML Pipeline name: This is the name declared in the `wercker.yml` file. You need to fill in with `deploy` and `linode` respectively.
    - Hook type: We will use the default behavior which is to "chain" this pipeline to another. You can select "Git push" when you want to run different pipelines in parallel each time a push is commited.

3.  Once you configure your pipelines you can start chaining them. Click on the "Plus button" to the right of the `build` pipeline:

    ![Chain pipeline](/docs/assets/wercker-chain-pipeline.jpg)

    You have the option to define an specific branch (or branches) to trigger the pipeline. By default Wercker will monitor all branches and start executing the steps if any commit is made, this is the case for our example. Select `deploy-docker` in the drop-down list and then click Add.

4.  Repeat this procedure to chain `deploy-linode` after this pipeline. The end result will look similar to this:

    ![Workflow screen](/docs/assets/wercker-workflow-01.jpg)

5.  Next you need to define the environmental variables, but this time you will do it inside each pipeline and not globally. Still on Workflows tab click on `deploy-docker` pipeline at the botton of the screen. Here you can create the variables. There are two variables from this example's `wercker.yml` that must be defined here: `DOCKER_USERNAME` and `DOCKER_PASSWORD`. Create them and don't forget to mark the password as "protected".

6.  Select the `deploy-linode` pipeline and create an SSH key pair, similar to the last example. Remember to copy the Public key to your remote server.

7.  Everything is set. Test your workflow by committing to your local fork:

        git add . && git commit -m "initial commit" && git push origin master

    You will end up with a result similar to:

    ![example workflow](/docs/assets/wercker-example-final.jpg)

8.  Test your application on the remote server by logging in remotely and running `docker images`:

    ![Docker images](/docs/assets/wercker-hello-images.jpg)

    Only an image tagged "current" is present.

9.  Run the application with the command:

        docker run <docker-username>/myapp:current

10. If you want to test your automation a bit further, edit your `hello.go` inside the `/example` folder. Add some text to the message (remember is reversed). Commit your changes and wait for the Wercker automation to run.

11. Run your application again. You should see the modified message:

  ![Changed text](/docs/assets/wercker-hello-run-02.jpg)

#### Wercker CLI Example

The final example demonstrates the Wercker CLI.

1.  Navigate to the root directory of the `getting-started-golang` fork.

2.  Run the following command to start Wercker:

        wercker build

  ![Wercker CLI build](/docs/assets/wercker-cli-build.jpg)

  The output should be similar to the logs you previously were able to view on the Wercker dashboard. The difference now is that you can check each step locally and detect any errors early in the process. The Wercler CLI replicates the SaaS behavior: it downloads specified images, builds, tests and shows errors. Since the CLI is a development tool intended to facilitate local testing, you will be unable to deploy the end result remotely.

3.  Build the application with Go:

        go build

  The application `getting-started-golang` is built in the root directory.

4.  Run the program:

        ./getting-started-golang

5. Navigate to `localhost:5000/cities.json` in your browser. You should see a list of cities displayed:

 ![Cities JSON](/docs/assets/wercker-cities-JSON.png)

6.  Close the program with Ctrl-C. Run the command:

        wercker dev --expose-ports

  ![Wercker Dev](/docs/assets/wercker-dev.jpg)

  This command starts the auto-build function in the `dev` pipeline. It builds the application within the Docker container and serves from there. If you make any changes to the app, the container will rebuild to reflect the changes.

7.  Open the `main.go` file in a text editor and add an entry to the list of cities. Refresh your browser and you should see the updated list.

## Next Steps

From a developer's point of view, there are infinite possibilities to play with when using Wercker:

* You can specify "local boxes", meaning you can use specialized images depending on your pipeline goals. DockerHub has many images ready to use for this purpose.
* You are not limited to "chained" workflows; you can start pipelines in parallel (as many as needed) and chain only when necessary. This is useful if you need to build a complex application that takes a long time to compile. You can start the compilation pipeline early in parallel with other tasks or better yet you can divide your application into several pipelines to reduce the time of each process and isolate problems.
* Wercker is language agnostic, process agnostic and platform agnostic. Your imagination is the limit for the possible uses of the Wercker tool.
