
# Modern application full lifecycle

## Description

This repository contains an example how to implement full CI/CD pipeline, from containerising an application to its deployment to a Kubernetes cluster. The goal is to show how it can be done in a simplistic way and not to pose as a guide.

## Usage

### Installing

It's expected that you have already installed on your system the following support applications:
  - [Docker](https://www.docker.com/)
  - [Minikube](https://minikube.sigs.k8s.io/docs/start/)
  - [git](https://git-scm.com/)
  - [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

### Folder structure

Here's project's folder structure :

```
/     			    # Root directory.
|- docker/app       # Folder used to store application Dockerfile.
|- docker/jk        # Folder used to store Jenkins Dockerfile.
|- kubernetes/app   # Folder used to store application k8s manifests.
|- kubernetes/jk    # Folder used to store Jenkins k8s manifests.
|- src/             # Application folder.
|- Jenkinsfile      # Jenkins pipeline file.
```

### The application

The application is a simple http server echo written in javascript that was selected randomly from GitHub. These conditions were taken in consideration while choosing the project:
- not containerised
- need to handle dependency
- set of tests that could be used on Jenkins pipeline

Link to original project: 
[https://github.com/watson/http-echo-server](https://github.com/watson/http-echo-server).

### Jenkins pipeline process

To illustrate the usage of Jenkins as a CI/CD tool to handle application lifecycle, the following steps are implemented:

- checkout application repository
- dependency installation (support packages needed to build the application)
- run batch of tests
- build docker image
- push docker image to image repository
- deploy to Kubernetes

There are other steps that would be nice to have:
- static code analyser, tool that checks code complexity and code smells
- linter, tool that flags syntax errors, syntax not following set standards and bugs
- check if the deployment was successfully executed and rollback in case of failure
- fire a smoke test after deployment with rollback in case of failure

#### Infrastructure setup
It is expected that you have a Minikube environment running locally with configuration context set to use Minikube cluster. All commands are expected to run on project root, unless asked otherwise.

##### Create application namespace on Kubernetes
```sh
kubectl apply -f ./kubernetes/app/namespace.yaml
```
##### Setup Jenkins:
- Step 1 - Build Jenkins custom image
This Dockerfile will include needed tools (kubectl and docker) and install all the required plugins
```sh
docker build -t xicoria/jk-k8s -f ./docker/jk/Dockerfile ./docker/jk
docker push xicoria/jk-k8s
```
######  (please change image name to reflect your docker hub user and remember to set the image public on docker hub. This image xicoria/jk-k8s is available, this step can be skipped if you don't want to build the image)
- Step 2 - Apply Jenkins manifest
```sh
kubectl apply -f ./kubernetes/jk/manifest.yaml
```
- Step 3 - Retrieve initial admin password
```sh
kubectl -n jenkins exec -ti `kubectl -n jenkins get pods -o jasonpath="{.items[0].metadata.name}"` cat /var/jenkins_home/secrets/initialAdminPassword
```
- Step 4 - Finish Jenkins installation
Use this command to open Jenkins service on your default browser
```sh
minikube service -n jenkins jenkins
```
Enter admin password, install suggested plugins, create admin user, restart and log in.

- Step 5 - Create credential to access Docker Hub
Create a Username with password credential with a valid user on Docker Hub using the id: docker_hub_login

- Step 6 - Create credential to access Kubernetes cluster
Create a Kubernetes configuration credential, with id: kube_config and fill the kubeconfig (Enter directly) content textarea with the output of the command:
```sh
kubectl config view --context=minikube --flatten
``` 
- Step 7 - Create credential to access GitHub
Create a Username with password credential with a valid Github user and create a personal access token to use as password. Use the id: github_login
(Personal access token can be found on Github settings > Developer Settings > Personal Access Token. Give permissions to repo and admin:repo_hook)

- Step 8 - Create pipeline
-- Create new item as a multibranch pipeline
-- Add source GitHub and choose appropriate credential
-- Paste application repository https address on Repository URL (https://github.com/Xicoria/echo.git)

After saving, it will trigger a Scan Repository process that will automatically start building master branch of the project. 

###### Repositories on Docker Hub are created as Private, you will need to change it to public if you want Kubernetes pod to fetch your image.
###### If you want to use your own images, there are two places where they are hardcoded. ./kubernetes/app/manifest.yaml and ./kubernetes/jk/manifest.yaml.

## Enhancements

#### How to ensure the application can handle heavy load
By observing pod metrics we can make use of Kubernetes HorizontalPodAutoscaler to react on specific load conditions and scale the number of pods available based on defined rules on min/max number of pods to scale to. 
Same care should be used monitoring all the nodes where the application can reside, appropriate node autoscaling can be used to broaden the resources of the cluster when appropriate.
Using benchmarking tools as [k6](https://k6.io/) or [wrk](https://github.com/wg/wrk)  to simulate heavy load, we can understand how the application reacts and from there we can set the application autoscaling rules and ensure that there is enough resource available on the cluster.
One thing to have in mind is that giving more resource or scaling horizontally is not always the best solution, as improvements on the application code or structure can have a huge influence on the amount of access it can handle. Identify the bottleneck and act informed is the best way to tackle heavy load problems.

#### Tools/strategy for logging
- Logs should be accessible, a stack such ELK provide the needed easy access to logs and filter information quickly
- Logs should be informative and logged to the right channel. Instruct developers to channel log information appropriately allow creation of alerts that will be triggered under undesired conditions (using for example Kibana Actions)
- Tools as [Unnomaly](https://unomaly.com/) can be used as well to add machine learning on log handling (react over abnormal logs)

To log from Kubernetes, there are two strategies that can be used to ingest logs that are:
- sidecar on application pod, that permits a more fine grained configuration of the log ingestor, but requires more maintenance.
- daemonset cluster-wide that will ingest all node logs to the log server

#### How to monitor the application to ensure constant uptime
To ensure constant uptime, Kubernetes is armed with unhealthy checks that will remove unhealthy pods from services and replace with new ones.
Another important aspect of constant uptime is to maintain disaster recovery procedures in a central knowledge base that can be used in the stressful moments of system failures.

#### Security concerns that would need to be addressed
Access should be given in a minimal fashion. An application or user must have only access to what they need.
For example, CI/CD tool keys for repository access (code and images) should be given by project and restricted to the data they need to access.
On application level the usage of a API Gateway, like [kong](https://konghq.com/), can handle user authentication, web firewalls, token verification, request/response rewrite.
Inside the cluster, network policies can be used to isolate pods, creating rules to allow access from/to only what is needed by the service.
Credentials and environment variables can be handled by a key management service like [Vault](https://www.vaultproject.io/) both to centralise where passwords/information are stored but also teams to take care of their application credentials.
Docker images should be scanned for known security breaches, so an inventory of images used on projects can reduce the impact of vulnerabilities. 

#### How to handle if the application needed a DB
If an application needs database, the best option is to allow access to a database on VM or a cloud database like RDS. 
The application would receive the credentials on pod creation from a key management service and appropriate network rules on AWS would restrict the access.

#### Automated tests approach
Every application must have Unit tests that can be easily integrated to the deployment pipeline. Not only Unit tests, but static code analyser, code linter and tools alike to keep code quality and ensure standards and best practices are followed.
On staging environment deployment pipeline, another set of tests can be triggered, such load test, test application for security issues, integration tests.
Deploying code to production would trigger smoke tests to ensure that the application is running and important environment variables/credentials are set and in place.

#### List of infrastructure tools that I would use 
- prometheus/grafana for monitoring/alert
- load test tool (k6, wrk) for benchmarking
- api gateway to impose security in front of services and to allow more control over incoming traffic and traffic distribution
- secret management tool to centralise key management and to permit teams have specific access to their application credentials
- selenium or another tool to automated integration tests
- elk stack for log
- jaeger for distributed tracing
