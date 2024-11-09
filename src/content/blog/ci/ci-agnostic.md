---
title: CI Agnostic
author: Tommy Tran Duc Thang
pubDatetime: 2024-11-02T08:02:27Z
slug: ci-agnostic
featured: true
# ogImage: ""
draft: false
tags:
  - CI/CD
  - CI Agnostic
  - dagger.io
  - earthly
description: This blog is for sharing knownledge and how to archive CI Agnostic
# canonicalURL: https://example.org/my-article-was-already-posted-here
---

## Table of contents

## I. Recommended Background Knowledge

- You are DevOps or Developer
- Familiar with CI
- Known at least 1 CI tool/platform (e.g: Gitlab CI, Jenkins, Github Action,
  Jenkins, ...)
- Familiar with Docker - Containerization

## II. Introduction

This is my first technical blog, so it may have some errors here and there, so
please create a **_Suggest Changes_** If you feel like changing any part of this
blog. Really appreciate it!

As a DevOps/Developer or even for someone who is a Tester/QA I'm pretty sure you
already get use to the term CI/CD and you may already using something like
Jenkins, Gitlab CI, Github Action, ... But this blog is not just focusing on
DevOps point of view, but rather for the experience of everyone who is using
CI/CD.

Have you ever faced the issue where, when you try to run some automation/deploy
your application but after awhile, just randomly it throw some nonsense error
after a long run that is not related to any of the application. Congratulation,
most likely you faced the issue of inconsistent CI.

Or if you are a DevOps enginner or anyone has some experience on developing
CI/CD then you will know that, each CI Platform will have different way to
setup, e.g:

| CI/CD Name       | Format                | Personal Opinion                                                                                                                         |
| ---------------- | --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Gitlab CI        | .gitlab-ci.yml (Yaml) | Very Good                                                                                                                                |
| Tekton CI        | K8s manifest (Yaml)   | Not rcm if you don't have indept K8s knowledge and high level skill because this requires a lot of ducktaping to work                    |
| Github Action    | ./github (Yaml)       | Good                                                                                                                                     |
| Jenkins          | ClickOps/Java-Groovy  | FUCK OFF - For me Jenkins only good initially - fast but not reliable and dependency versioning is like ass - Please stay away from this |
| AWS CodePipeline | buildspec.yaml (Yaml) | OK but for some unique cases you may need to do more tricky stuff                                                                        |

> Disclaimer: This is only my opinion. But trust me, I used all of them
> extensively with a deep understanding.

And all of them are really different compare to each other. Then with a big
system, it will be very hard for us to migrate from this platform to another,
and to my experience, I had to a lot of project related with migrating CI
platform before so it's a pain in the a\*\*. Let me give an example:

Let's say a **_SIMPLE_** CI for running lint -> build -> push image with NodeJs.
Here is how it is going to looks like for Jenkins and Gitlab CI:

Jenkins Example:

```groovy
pipeline {
    agent any
    environment {
        DOCKER_IMAGE = "${env.REGISTRY_URL}/${env.JOB_NAME}:${env.BUILD_NUMBER}"
    }
    stages {
        stage('Lint') {
            agent {
                docker {
                    image 'node:18'
                    args '-v $HOME/.npm:/root/.npm' // Cache NPM dependencies if needed
                }
            }
            when {
                branch 'main'
                expression { return env.CHANGE_ID != null } // Run on merge requests or main branch
            }
            steps {
                script {
                    sh 'npm install'
                    sh 'npx eslint .'
                }
            }
        }

        stage('Build') {
            agent {
                docker {
                    image 'node:18'
                    args '-v $HOME/.npm:/root/.npm'
                }
            }
            when {
                branch 'main'
                expression { return env.CHANGE_ID != null } // Run on merge requests or main branch
            }
            steps {
                script {
                    sh 'npm install'
                    sh 'npm run build'
                }
            }
            post {
                success {
                    archiveArtifacts artifacts: 'dist/**', allowEmptyArchive: true
                }
            }
        }

        stage('Push Image') {
            agent {
                docker {
                    image 'docker:20.10.7'
                    args '--privileged -v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            when {
                branch 'main'
            }
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-credentials-id', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh 'echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin'
                        sh "docker build -t $DOCKER_IMAGE ."
                        sh "docker push $DOCKER_IMAGE"
                    }
                }
            }
        }
    }
}

```

Gitlab-CI Example:

```yaml
stages:
  - lint
  - build
  - push

# Linting Stage: Run ESLint
lint:
  image: node:18 # Use a Node.js Docker image
  stage: lint
  script:
    - npm install # Install dependencies (if ESLint is a dependency)
    - npx eslint . # Run ESLint on the codebase
  only:
    - merge_requests
    - main # Optionally, restrict to certain branches

# Build Stage: Install dependencies and build the app
build:
  image: node:18 # Use a Node.js Docker image
  stage: build
  script:
    - npm install # Install project dependencies
    - npm run build # Run the build script (adjust if you have a custom build script)
  artifacts:
    paths:
      - dist/ # Optionally save build artifacts, like compiled files
    expire_in: 1 hour
  only:
    - merge_requests
    - main # Optionally, restrict to certain branches

# Push Image Stage: Build and push Docker image to registry
push_image:
  image: docker:20.10.7 # Use the Docker image to build and push the image
  stage: push
  services:
    - docker:20.10.7-dind # Enable Docker-in-Docker for building images
  variables:
    DOCKER_DRIVER: overlay2 # Docker storage driver (optional, depending on the CI environment)
  script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER"
      --password-stdin $CI_REGISTRY # Log in to GitLab's registry
    - docker build -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_REF_NAME" . # Build Docker image
    - docker push "$CI_REGISTRY_IMAGE:$CI_COMMIT_REF_NAME" # Push image to GitLab registry
  only:
    - main # Push image only on the `main` branch or change this as per your needs
  when: on_success
```

You already can see they have really different declaration type/syntax/structure
not to mention other stuff like system integration, and this is only a very very
simple pipeline. Imagine the workload required when you need to migrate 3000
pipelines and each pipeline uses different script files sum up to around avg
1000 script lines PER PIPELINE. Yes, and yes, I been to those messy project, and
there is no way you can migrate them smoothly :D.

Maybe this is too long for an introduction but please understand for me. This is
the first time a write a technical blog. He he ^^.

## III. Problems To Solve

From the introduction we can list out the problems really common for CI systems:

- Inconsistent between environment (Local/Other Environment)
- Hard to maintain when the system grows and having bad engineers doing nonsense
  spaghetti code
- Almost impossible to change to other CI platform when the system grew to a
  certain level

## IV. Solution

So this blog I will present a term for you which will solve all of the problems
above with **_CI AGNOSTIC_**.

### 1. What is CI Agnostic?

CI Agnostic basically embracing the value of Containerization. Instead of
leaving the CI declaration to CI Platform we do it by declaring the steps it
gonna do in a container.

With the old traditional way:
![Traditional Way of doing CI](@assets/images/ci-agnostic/blog-ci-agnostic.png)

> We will declare CI in the CI Platform specific language (e.g: .gitlab-ci.yml,
> buildspec.yml, Jenkinsfile, ...)

Agnostic way:
![Agnostic Way of doing CI](@assets/images/ci-agnostic/blog-ci-agnostic-new-approach.png)

> We will declare CI in containerized environment. And both local machine and
> Remote CI platform can just execute this containerized environment.

This will solve the problem of inconsistent CI between environments because all
of them will run in containers. Now Developers can also run the CI directly on
their local machine to!!! which means there won't be 5000 commits on the
repository just to do some CI testing.

Able to run CI on local and knowing that it will run exactly the same on other
env -> less prone to error, faster feed back loop -> FASTER DELIVERY TIME

Not only That, it will be very easy to move our CI workload to a different
platform with ease. That's why it's called **_CI Agnostic_**

But that is not easy solve just buy putting everything into the Dockerfile (You
still can do this but it will have a lot problem with performance/caching and
hard to handle hard logic). Fortunately, on the market right now, there are a
few tools support us to achieve CI Agnostic.

### 2. Available Tools And Technologies
All of the tools I listed here are opensource :D btw.

| Name      | Complexity   | Declaration Language                   | Community Support | Personal Opinion                                                                                                                                              |
| --------- | ------------ | -------------------------------------- | ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Dagger.io | Medium       | Go/TS/Python                           | Good              | It's good but not easy to get on if the team skill level is not high                                                                                          |
| Earthly.dev   | Low - Medium | Earthly syntax (Similar to Dockerfile) | Good              | It's good, easier to catch on compare to Dagger                                                                                                               |
| Batect    | Easy         | Yaml                                   | Not Maintained    | I like the idea and the way this was implemented really similar to taskfile.dev and the simplicity of it. Anyone can understand without having much knowledge |

I would recommend Dagger.io or Earthly since they have more support from the community and more functionalities/features.

### 3. How Do They Work?

For the case of Dagger.io and Earthly, both of them use Buildkit behind the scence and having same approach and only just slightly different in the declaration way:
- Dagger.io uses Go/TS/Python to declare the steps/functions
- Earthly uses its own declarative way which is really easy to catch on (They said Earthly is like makefile and Dockerfile have a baby - And I think it's true)

Both of them for every execution will be able to execute in a docker container. And they can utilize buildkit to better utilizing caching and performance. For the context of the blog explaining buildkit will be too long. So just don't care about it right now :D. I may create another blog just for that topic

### 3. Demo example
The demo I gonna do today is from my another repo I'm working on to write a K8s Controller. Maybe it's 

[K8s Controller Pod Cloud Role Identity](https://github.com/TranThang-2804/k8s-pod-identity-controller)

1. Prerequisites:
- Golang (v1.22.3)
- Earthly (latest)

2. Clone the repository
```sh
git clone https://github.com/TranThang-2804/k8s-pod-identity-controller.git
```

3. You will notice that in the repository, there already an Earthfile so you don't have to do anything.
Here is the content of the Earthfile:



## V. Reference:

