# PipelinePlayground

Experimenting with setting up a minimal pipeline (locally)
Jenkins running in docker, with builds triggered by commits.

Tools used:
- Git
- Jenkins
- Docker

## Jenkins in Docker
Want access to Docker, so we can run Docker in the Jenkins container.

#### Dockerfile

    FROM jenkins/jenkins:lts
    
    USER root
    RUN apt-get update -qq \
        && apt-get install -qqy apt-transport-https ca-certificates curl gnupg2 software-properties-common 
    RUN curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add -
    RUN add-apt-repository \
    "deb [arch=amd64] https://download.docker.com/linux/debian \
    $(lsb_release -cs) \
    stable"
    RUN apt-get update  -qq \
        && apt-get install docker-ce=17.12.1~ce-0~debian -y
    RUN usermod -aG docker jenkins
_Adapted from [source]()_

#### Build the image
    docker image build -t jenkins-docker .

#### Start container with mounted docker daemon and volume for project

    docker container run -d -p 8080:8080 -v /var/run/docker.sock:/var/run/docker.sock -v C:/"Path to some (local) repository":/var/projects jenkins-docker

___

## Set up Jenkins

Create a pipeline and set it to trigger builds remotely. Here I use a token: _mytoken_.

    "URL of Jenkins"/job/"Name of pipeline"/build?token=mytoken

This URL will trigger a build. In my case it looks like this:

    http://localhost:8080/job/MyPipeline/build?token=mytoken

I also set Jenkins to allow anyone to trigger builds, as this is run locally anyway, and makes things a bit simpler (at least for me).

In the pipeline section, set it to _Pipeline script from SCM_, and specify path to your project under _Repository URL_. The path should be the path on the mounted volume, e.g. _/var/projects/_ as above, with _file://_ prepended. It should look like this:

    file:///var/projects/

Under _Script Path_ write the name of your Jenkinsfile, in my case it is simply _Jenkinsfile_.

It does not matter if there actually is a Jenkinsfile in the mounted volume right now, because you can just commit it, and it will be added to the mounted volume.

### Jenkinsfile

For this experiment I wanted to try a few things, and the Jenkinsfile will reflect that. Within Jenkins the things that are interesting to look at is to use Docker (while Jenkins itself is run in Docker), and to do something with the repository. With these goals, I wrote the following pipeline script.

    pipeline {
        agent { docker { image 'python:3.5.1' } }
        stages {
            stage ('build') {
                sh 'python /var/projects/some_python_file.py'
            }
        }
    }

Here we create a python agent using Docker, and runs a python program from the project, specifying its path on the mounted volume.

___

## Set up Git

In the local project specified above there will be a **.git** folder containing a **hooks** folder. The hooks folder contains examples of _git-hooks_, which are examples of scripts describing behavior triggered by git commands.

Create a new file called _post-commit_, and add your URL for triggering Jenkins, so that it looks like this:

    #!/bin/sh
    curl http://localhost:8080/job/Apipeline/build?token=mytoken

Now a build should trigger in Jenkins everytime a commit is done.


