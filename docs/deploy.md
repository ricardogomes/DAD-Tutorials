# Deploy DAD Project - TESTE

**NOTE**: To run most of the command present in this tutorial we need to either be connected to our school network or use its [VPN](./vpn.md).

To deploy the project to our Kubernetes cluster we will need docker and `kubectl` (the Kubernetes cli) see the [tools tutorial](./tools.md) for installation instructions.

Each group has access to their own namespace on kubernetes so that the projects remain isolated.

> [!IMPORTANT]
> Given that you most likely already started working on our project and it might not have all the components testable, our suggestion is that you submit the provide code in the intermediate submission. Our goal is to make sure each group can deploy their project with all the components running. After the submission evaluation we can switch to using the our own code.

## Prerequites

Before we can publish our project there's some prerequisites we need to make sure we have. See the following sub-sections for instructions.

Add the config file you got via email on to `~/.kube/` and rename it to simply `config`. NOTE: if the .kube folder does not exist we need to create it.

Test you connection to the cluster.

```bash
kubectl get pods
```

## Prepare the Deployment

This tutorial assumes that we have our project in the following folder structure:

```bash
./                  -> Project Base
├── deployment      -> deployment files (Dockerfiles and Kubernetes Resources)
├── api             -> Laravel Project Root
├── api/deployment  -> Folder inside the Laravel project containing a Caddyfile
├── frontend        -> Vue Project Root
|── websockets      -> Node Web Sockets Project Root
```

The first task we need to do is find all the instances of the string `dad-group-x` replace the `x` with our group ID. This string is what allows our projects to be deployed to the proper [namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) on kubernetes.

### Tooling

The repository associated with these tutorials has all the commands defined in a `justfile`. [Justfile](https://github.com/casey/just) is a simplified version of a `Makefile` and we need to install if (instructions on the github page) we want to use it. Still on this tutorial we will see all the commands so that we can run them manually if we want.

Each group has access to their own namespace on kubernetes but they need to register the images and deployments with the proper reference to the group. That reference has the format **dad-group-X**, where X is the group number.

### Preparing Vue

In Vue's case we can take advantage of the `.env` files support of Vite to create a `.env` and a `.env.production` files with the following variables (again X to be replaced with the group number):

```ini
# .env
VITE_API_DOMAIN=localhost:8000
VITE_WS_CONNECTION=ws://localhost:3000


# .env.production
VITE_API_DOMAIN=api-dad-group-X-172.22.21.253.sslip.io
VITE_WS_CONNECTION=ws://ws-dad-group-X-172.22.21.253.sslip.io
```

## Deploy the code

To deploy our code to the cluster we need to build or container images locally, and push them to the container registry, that in our case lives at `registry-172.22.21.115.sslip.io`.

Present in the repository under the [code](https://github.com/ricardogomes/DAD-Tutorials/tree/main/code) folder is one called `deployment`. We are going to need those files.

Assuming we have a deployment folder in the same place as this tutorial repository we can run the next commands from the base of the full project. But check each command for the placement of the files and folders and change accordingly.

> [!IMPORTANT]
> One important **NOTE** is that Kubernetes will depend on us naming the versions of our apps to re-deploy them when we change our code, so keep in mind that we need to bump the version each time we want to push a new version of our applications.

### Configure Docker

Our Docker Registry does not use HTTPS so we ned to inform Docker that this registry is only available on port 80.

This is the configuration we need (this is a JSON property to be included into either existing configuration or a new JSON config by wrapping it in { }):

```json

    "insecure-registries" : [ "registry-172.22.21.115.sslip.io" ]

```

On Windows and MacOS the simplest way is to configure this via Docker Desktop.

![Docker Insecure Registries](assets/docker-insecure-registries.png)

On linux we can edit the file directly via `sudo nano /etc/docker/daemon.json` and restart the service via `sudo systemctl restart docker`.

### Push Images to Container Repository

Build the Laravel Image (replace group with your group id - dad-group-X and the version with the current version - 1.0.0):

```bash
    docker build -t registry-172.22.21.115.sslip.io/{{group}}/api:v{{version}} --platform linux/amd64 \
    -f ./deployment/DockerfileLaravel ./api
```

Push the Laravel Image (replace group with your group id - dad-group-X and the version with the current version - 1.0.0):

```bash
docker push registry-172.22.21.115.sslip.io/{{group}}/api:v{{version}}
```

Build the Vue Image (replace group with your group id - dad-group-X and the version with the current version - 1.0.0):

```bash
docker build -t registry-172.22.21.115.sslip.io/{{group}}/web:v{{version}} --platform linux/amd64 -f ./deployment/DockerfileVue ./frontend
```

Push the Vue Image (replace group with your group id - dad-group-X and the version with the current version - 1.0.0):

```bash
docker push registry-172.22.21.115.sslip.io/{{group}}/web:v{{version}}
```

Build the Node WebSockets Image (replace group with your group id - dad-group-X and the version with the current version - 1.0.0):

```bash
docker build -t registry-172.22.21.115.sslip.io/{{group}}/ws:v{{version}} --platform linux/amd64 -f ./deployment/DockerfileWS ./websockets
```

Push the Node WebSockets Image (replace group with your group id - dad-group-X and the version with the current version - 1.0.0):

```bash
docker push registry-172.22.21.115.sslip.io/{{group}}/ws:v{{version}}
```

### Deploy Resources to Kubernetes Cluster

Before we can publish our Kubernetes resources we need to replace the string 'dad-groupx' in each file with our actual group.

We can now deploy our resources:

```bash
kubectl apply -f deployment/
```

Check the deployment using:

```bash
kubectl get pods
```

Container may take a bit to get to the `healthy` state but after that would should be able to reach your application at:

- VUE: [http://web-dad-group-x-172.22.21.253.sslip.io](http://web-dad-group-x-172.22.21.253.sslip.io)
- Laravel: [http://api-dad-group-x-172.22.21.253.sslip.io](http://web-dad-group-x-172.22.21.253.sslip.io)

## Running commands

We sometimes need to run commands on the containers running in the cluster, one example are the Laravel commands (like migrate), we can do this by using these commands (X is the group number):

```bash

# get pod name
kubectl -n dad-group-X get pods -l app=laravel-app


kubectl -n dad-group-X exec -it <pod-name> -- php artisan migrate:fresh --seed

```

To redeploy the application stack you can run:

```bash

kubectl -n dad-group-X rollout restart deployment/laravel-app
kubectl -n dad-group-X rollout restart deployment/vue-app
kubectl -n dad-group-X rollout restart deployment/websocket-server
```

### Checking the Logs

To check the logs of a particular component we can run these commands:

```bash

kubectl get pods

# whit the full pod name

kubectl logs <full-pod-name>

```

## Changing the applications

When you need to change the application code a new container image, with a new version must be built.

If you are using the just command, you can change the version variable at the top of the file, if not you need to run the docker build and docker push commands with the new version.

After that the new version can be defined in the appropriate kubernetes yaml file and run the command `kubectl apply -f deployment/`.
