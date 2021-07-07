# Kubernetes "Lab" Exercise: Deploy a web api

## July 7, 2021

## Prerequisites

We'll use a Docker repository to store & pull the web api image. To keep it simple for now, we'll use a public repository so k8s can pull the image without needing login credentials.

* If you don't already have one, create a [dockerhub](https://hub.docker.com/) account or equivalent.

* Create a public Docker image repository there called "todoapi".

## Lab

### 1. Start up your k8s cluster

* Set up your cloud (EKS) k8s cluster like we did [here](https://github.com/us-learn-and-devops/2021_05_26/blob/main/README.md).
  * But this time, don't deploy the nginx `service` and `deployment` that we used last time. We'll deploy a todo app instead.

### 2. Build and push the app image

While you're waiting for your cluster...

* Check out the todo app [here](https://github.com/us-learn-and-devops/2021_07_07/todoapi). It's a REST API that stores a todo list.

  * Note that the server is configured to listen on port 8080 by default. (This will be relevant both now, if you want to try running it locally, and later when we deploy to k8s.)

  * If you have Go and `make` installed, try running it locally on localhost:8080 by running the command `make run`.

    * You'll get a simple welcome message via a GET to `localhost:8080/`.

    * You can add to the list via a POST to `localhost:8080/todo` with a JSON body that includes a "name" field and optional "description" field, e.g.

          {
              "name": "shopping",
              "description": "get milk and eggs"
          }

    * You can view the list via a GET to `localhost:8080/list`.

    * This initial version of the todo list app is primitive and only saves the list to memory, so the list will be "forgotten" if you stop the server.

      * We'll iterate on this app in future demos/exercises. This will give us an opportunity to add persistent storage and configure backups using a k8s volume claim.

  * There is a 2-stage Dockerfile included. You can use that to create an image for the web app.

* Use Docker locally to [build the Dockerfile](https://docs.docker.com/engine/reference/commandline/build/#tag-an-image--t) and tag the image.

* [Login](https://docs.docker.com/engine/reference/commandline/login/) to your Docker Hub (or equivalent) account and [push](https://docs.docker.com/engine/reference/commandline/push/) the tagged image to your "todoapi" respository.

### 3. Create a `deployement`

When your cluster is ready and you've configured kubectl access...

* Test that your cluster is up and running. (How can you do this using kubectl?)

* Refer to the [`deployment` template](https://github.com/us-learn-and-devops/2021_05_26/blob/main/webapp-deployment.yaml) from May 28 and/or the [kubernetes docs](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#creating-a-deployment); create a similar deployment for your todo app using the image you just pushed.

  * Note: Instead of deploying 3 pods, deploy just 1 pod for now. We'll scale up in a few minutes.

* How can you reach your web api to test a REST request?

  * Try using port-forwarding from localhost to one of the deployment pods. Check out the [README](https://github.com/us-learn-and-devops/2021_06_09) from June 9 for a refresher.

    * Don't forget that, unless you've changed it, the server is listening on port 8080.

    * Once you've got port-forwarding working, you should be able to add to the list using a POST to the `/todo` endpoint, and view the list via a GET to the `list` endpoint -- like described above if you tried running the server locally.

      * (Use Ctrl+C to "stop" the port-forwarding when you're done.)
  
### 4. Create a load-balancer `service`

Add more sophisticated access to the deployed pods:

* Refer to the [`service` template](https://github.com/us-learn-and-devops/2021_05_26/blob/main/webapp-svc.yaml) from May 28 and/or [kubernetes docs](https://kubernetes.io/docs/concepts/services-networking/service/#defining-a-service); create a similar service for your todo app.

  * Note that because the api server is listening on port 8080, you'll need to map port 80 (default http port) for the load balancer to port 8080 for the service endpoints. Check out the above link to the k8s docs to find the way to configure this.

* How can you reach your web api to test a REST request now that you've got a load-balancer service?

### 5. Scale the deployment

* Scale the deployment to 3 pods.

* How can you check to see if you've really got 3 pods now?

* Add a couple of `todo`s to the list, then GET the list. Make sure everything seems to be working.

  * Note: This early version of the todo app only saves to local memory. Because the pods don't share state, you could get different results from a GET if you retry it -- but you probably won't unless you mess with the pods (e.g. by scaling) because the load balancer keeps track of your client session.

* Attempt to scale up to 6 pods.

  * What happened?

* We've gotten into trouble. You could delete the deployment and then redeploy it. (How do you do that?) Can you think of anything else to fix the situation?

  * When you redeploy, scale back down to 1 pod again.

## *6. More trouble*

* What happens if a deployment can't launch pods?

  * Sabotage your deployment config file by misspelling the image name. Try a redeploy.

  * What happens?

## *7. Health checks*

* Refer to the [kubernetes docs](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-liveness-http-request) and add a "liveness" health check to your deployment.
  * What is a "liveness" probe?
  * The todo api has a root endpoint ("/") that you can use for a simple `httpGet` liveness probe.
  * Note you'll need to consider what port and `periodSeconds` check interval to specify.

* (We'll check out readiness probes when the todo app gets more sophisticated. Feel free to read about them now, though.)

* What happens if you misconfigure the liveness probe? Try making the `httpGet` path something that doesn't exist, like "/doesntexist". What happens?

## *8. Resource limits*

* Refer to [kubernetes docs](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#resource-units-in-kubernetes) on resource limits.

  * Try specifying CPU and memory resource/limit configuration in your deployment. We are using t3.micro instances for our worker nodes. How might you take that into consideration here?

## (Suggested) Command Reference (using bash)

### *2. Build and push the app image*

* `docker build -t mydockerhubacct/todoapi:latest .`

* `export REGISTRY_PASSWORD=xzy`

  `echo $REGISTRY_PASSWORD | docker login -u myusername --password-stdin`

* `docker push mydockerhubacct/todoapi:latest`

### *3. Create a `deployement`*

* `kubectl get svc -n default`

  `kubectl get pods -n kube-system`

* `kubectl apply -f ./deployment.yaml`

* `kubectl get pods --selector="env=dev"`

  `kubectl port-forward todoapi-7dbb7c8d8-v2zfd 8080:8080`, then Ctrl+C to exit

### *4. Create a load-balancer `service`*

* `kubectl apply -f ./service.yaml`

* `curl http://aa6cc89ee10e149e38695cd4c2c8965c-1807306611.us-west-2.elb.amazonaws.com/list`

### *5. Scale the deployment*

* Update the relica count in deployment.yaml and then `kubectl apply -f ./deployment.yaml`

* `kubectl get pods --selector="env=dev"`

  `kubectl describe deploy todoapi -n default`

* Update the relica count in deployment.yaml and then `kubectl apply -f ./deployment.yaml`

  `kubectl get pods -n default --watch`
