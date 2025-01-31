# How to test it

Fork this repo:

https://github.com/amioranza/todomvc

This repo have the Github Actions Workflow, by default it is disable, you need to click on Actions an then click in the green button to activate the Github Actions.

Every push to the master branch will trigger a new build.

## Pipeline Documentation

## When it runs

This pipeline will be triggered at any push to the master branch as described here:

```
on:
  push:
    branches:
      - master
```

It means that every line of code pushed to the master branch will generate a new package (docker image) ready to deploy.

## What it does

First it defines two global env vars to use on all steps of this pipeline:

```
env:
  CI: true
  GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

After the environment variables it has the jobs section, where the job called prepare-build-publish was created. The name of the job must be as descriptive as possible to makes it easy to understand its purpose.

This job have steps that effectively prepare (clone the repository) , build (build the multi stage docker image) and finally publish (publish the docker image) using two tags: the first is the SHA256 hash of the commit and the second is the tag `latest` to make it easy to use on the yaml and command line.


```
jobs:
  prepare-build-publish:
```

The directive runs-on defines the operating system of the job runner, in this case ubuntu-latest.

```
    runs-on: ubuntu-latest
```

The first step/action calls a default function to checkout the repository code and the `@master` defines the branch to be checked out.

```
    steps:

    - uses: actions/checkout@master
```

The next step is the Github info output just to show off the vairables avaliable in the environment, it calls the run directive where bash commands will be executed.

```
    - name: Github Info
      run: |
        echo "GITHUB_ACTION: $GITHUB_ACTION"
        echo "GITHUB_ACTOR: $GITHUB_ACTOR"
        echo "GITHUB_REF: $GITHUB_REF"
        echo "GITHUB_HEAD_REF: $GITHUB_HEAD_REF"
        echo "GITHUB_BASE_REF: $GITHUB_BASE_REF"
        echo "github.event_name: ${{ github.event_name }}"
        cat $GITHUB_EVENT_PATH
```

The build step is the most important step, it will build the application using the Dockerfile that resides in the examples/vue directory of the repo. If some change is needed it have to be done on the Dockerfile itself. The build step will always execute the build of the container image considering a clean environment. It builds the first container tag with the SHA256 hash of the commit and repeat the command to add a second tag to the container defined as `latest`.

```
    - name: build
      run: |
        cd $GITHUB_WORKSPACE/examples/vue
        export LOWERCASE_REPOSITORY=$(echo "${GITHUB_REPOSITORY}" | tr "[:upper:]" "[:lower:]")
        export IMAGE_NAME="docker.pkg.github.com/${LOWERCASE_REPOSITORY}/todomvc"
        export IMAGE_TAG="v${GITHUB_SHA}"
        docker build -t $IMAGE_NAME:$IMAGE_TAG .
        docker build -t $IMAGE_NAME:latest .
```

The final step of this Workflow is to publish the generated images in the GitHub Docker registry. It will show the package in the project page where you can pull or run this image after login on the Github Container registry.

```
    - name: publish
      run: |
        export OWNER="${GITHUB_REPOSITORY%/*}"
        export LOWERCASE_REPOSITORY=$(echo "${GITHUB_REPOSITORY}" | tr "[:upper:]" "[:lower:]")
        export IMAGE_NAME="docker.pkg.github.com/${LOWERCASE_REPOSITORY}/todomvc"
        export IMAGE_TAG="v${GITHUB_SHA}"
        echo "owner: ${OWNER}"
        echo "token: ${GITHUB_TOKEN}"
        echo "repo: ${GITHUB_REPOSITORY}"
        echo "version: ${VERSION}"
        docker login -u ${OWNER} -p ${GITHUB_TOKEN} docker.pkg.github.com
        docker push $IMAGE_NAME:$IMAGE_TAG
        docker push $IMAGE_NAME:latest

```

## What the Dockerfile does

This is a multi stage build, it means that the build stage will not be sent to the final image keeping it clean and as small as possible.

The `FROM` directive defines the base image to use, in this case node:lts-alpine. The `as build-stage` is the name used for this stage to be referred by the next stage, more bellow.

```
FROM node:lts-alpine as build-stage
```

The `WORKDIR` is a metadata directive, it does not create any additional layer, it only changes the current working dir inside the container to the specified path.

```
WORKDIR /app
```

The `RUN` directive will run the commands listed. In this case it uses a chained sequence of commands where if some command fails all the chain fail. It avoids some broken builds. The commands used here install all the node dependencies, create a new vue.js project and build the project.

```
RUN npm install -g @vue/cli todomvc todomvc-app-css && \
  vue create -d -f -n todomvc && \
  cd todomvc && \
  npm run build
```

The next three `COPY` directives copies the artifacts from the repository to the container filesystem.

```
COPY index.html /app/todomvc/dist/index.html
COPY js /app/todomvc/dist/js
COPY node_modules /app/todomvc/dist/node_modules
```

To make the container smaller in size and more secure it uses the multi stage build where only the relevant artifacts generated by the container named as `build-stage` should be sent to the `production-stage` in this case the second container defined in the Dockerfile. The second `FROM` directive is the final artifact that will be sent to the Docker Registry to be used as the image of the service. This `FROM` directive uses the nginx:stable-alpine as the production image.

```
FROM nginx:stable-alpine as production-stage
```

The `COPY` directive uses the paramter `--from=build-stage` to tell the docker that the artifacts have to be copied from that stage to the current stage. It will copy everything inside the dist directory from the `build-stage` to the nginx default directory in the `production-stage`.

```
COPY --from=build-stage /app/todomvc/dist/ /usr/share/nginx/html
```

The `EXPOSE` directive defines the public port of the container, it will be used as the published port when running the container via docker or any other container orchestrator.

```
EXPOSE 80
```

The `CMD` directive is the default command line of that container. It means that all the time it runs it will run `nginx` with `-g` and `daemon off;` parameters. It can be overrided by specifying commands after the container image in the command line.

```
CMD ["nginx", "-g", "daemon off;"]
```

## The complete flow overview

```
                         +---------------------------------+
                         |                                 |
                         |    FLOW STEP: Checkout Master   |
                         |                                 |
                         +---------------------------------+
                                          |
                                          |
                                          |
                         +---------------------------------+
                         |                                 |
                         |        FLOW STEP: Build         |---------------------|
                         |                                 |                     |
                         +---------------------------------+                     |
                                                                                 |
                                                                +--------------------------------+
                                                                |                                |
                                                                |    DOCKERFILE: build-stage     |
                                                                |                                |
                                                                +--------------------------------+
                                                                                  |
                                                                                  |
                                                                                  |
                                                                +--------------------------------+
                                                                |                                |
                                                                |  DOCKERFILE: production-stage  |
                                                                |                                |
                                                                +--------------------------------+                                     |
                        +---------------------------------+                      |
                        |                                 |                      |
                        |        FLOW STEP: Publish       |----------------------|
                        |                                 |
                        +---------------------------------+
```

## How to run it on Docker

You can run the generated docker image with the steps bellow:

```
docker login -u <github_user> -p <github_personal_token> docker.pkg.github.com
docker run -it -d -p 3000:80 --name todomvc docker.pkg.github.com/<github_user>/todomvc/todomvc:latest
```

Access http://localhost:3000 and you can access the todomvc vue.js version

## How to deploy to Kubernetes

First you need to create the secret with your credentials to the Github Docker Registry

```
kubectl create secret docker-registry registry --docker-server=docker.pkg.github.com --docker-username=<github_user> --docker-password=<gu=ithub_personal_token>
```

To deploy on kubernetes you can run the commands bellow:

```
kubectl apply -f k8s/todomvc.yml
```

To access the access the aplication you need to discover at least one of the worker nodes IP and the service port to access the application:

```
kubectl get nodes -o wide
```

Choose any IP of nodes with the worker role, then get the service port of the deployment:

```
kubectl get svc todomvc -o json | jq '.spec.ports[]  | .nodePort'
```

Then access the url like this `http://<discovered_node_ip>:<service_port>`

The yaml fille applied to the cluster will create the Horizontal Pod Autoscalling object. It will monitor the use o CPU of the container and if it hits the 50% threshold it will fire a new repolica of the same application. It will scale out and scale in the application based on the CPU consumption.
