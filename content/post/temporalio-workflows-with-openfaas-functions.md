---
title: "Temporal.io workflow with Openfaas functions"
date: 2020-10-26T19:15:26+02:00
draft: false
thumbnail: "images/workflow.png"
tags: [
    "openfaas",
    "serverless",
    "k8s"
]
---

This is a simple setup to test the recently released workflow engine [Temporal](https://docs.temporal.io/docs/overview/) (from Uber's Cadence creators) with [Openfaas](https://docs.openfaas.com/) serverless framework in a local k3d kubernetes cluster.

<!--more-->

Openfaas is an awesome serverless framework project to spin up functions and api-services quickly in a variety of programing languajes focused (but not limited) to one-shot functions.

Temporal allows for complex workflows hiding most of the complexity behind building scalable distributed applications (from their website).


{{< youtube id="f-18XztyN6c" autoplay="false" >}}

## Install Arkade and required packages

Required packages are easy to install with [arkade](https://github.com/alexellis/arkade) from Alex Ellis, the creator and founder of Openfaas:

```shell
 curl -sLS https://dl.get-arkade.dev | sudo sh
 
 arkade get faas-cli
 arkade get k3d
 
 export PATH=$PATH:$HOME/.arkade/bin/
```

## Create k8s dev cluster

Set up a kubernetes locally from k3d with a traefik load balancer:

```shell
 k3d cluster create openfaas-temporalio --agents 2 --api-port 6443 --port "8080:80@loadbalancer"
 
 export KUBECONFIG=$(k3d kubeconfig write openfaas-temporalio)
 
 kubectl cluster-info
 kubectl get nodes
```

## Install Openfaas

Launch openfaas in k3d with arkade:

```shell
 arkade install openfaas --basic-auth=false --set=serviceType=ClusterIP --set=ingress.enabled=true --set=ingress.annotations."kubernetes\.io/ingress\.class"=traefik
```

> Default ingress hostname is 'gateway.openfaas.local'


## Install Temporal

https://github.com/temporalio/helm-charts#minimal-installation-with-required-dependencies-only

This is the minimal stack to use temporal (without metrics or Elastic Search):

```shell
 git clone https://github.com/temporalio/helm-charts
 
 cd helm-charts
 
 helm dependencies update
 
 # note: there is a pending PR in this repo for the ingress indentation
 helm upgrade --install \
     --set server.replicaCount=1 \
     --set cassandra.config.cluster_size=1 \
     --set prometheus.enabled=false \
     --set grafana.enabled=false \
     --set elasticsearch.enabled=false \
     --set kafka.enabled=false \
     --set web.ingress.enabled=true \
     --set web.ingress.annotations."kubernetes\.io/ingress\.class"=traefik \
     --set web.ingress.hosts[0]=dashboard.temporal.local \
     temporaltest . --timeout 15m
```

### Add ingress to /etc/hosts

Instead of using port-forward to access applications i will add the k3d traefik ingress rules to /etc/hosts with this snippet (requires [jq](https://github.com/stedolan/jq) to parse json):


```shell
 kubectl get ingress --all-namespaces -o json | jq -r ' "\(.items[].status.loadBalancer.ingress[].ip) \(.items[].spec.rules[].host)"' 2>/dev/null | uniq | sudo tee -a /etc/hosts
```

### Check Openfaas

Test openfaas is working with a sample function from the store:

```shell
 export OPENFAAS_URL=http://gateway.openfaas.local
 
 faas-cli store deploy Figlet
 faas-cli list
 echo "test openfaas" | faas-cli invoke figlet
 # expected output
 #  _            _                             __                 
 # | |_ ___  ___| |_    ___  _ __   ___ _ __  / _| __ _  __ _ ___ 
 # | __/ _ \/ __| __|  / _ \| '_ \ / _ \ '_ \| |_ / _` |/ _` / __|
 # | ||  __/\__ \ |_  | (_) | |_) |  __/ | | |  _| (_| | (_| \__ \
 #  \__\___||___/\__|  \___/| .__/ \___|_| |_|_|  \__,_|\__,_|___/
 #                          |_|                                   
```


### Check Temporal

There is a lot of samples in https://github.com/temporalio/samples-go to show the capabilities of Temporal.

Running the helloworld sample from the temporal docs:

* Create the default temporal namespace with the integrated CLI

```shell
 kubectl --namespace=default get pods -l "app.kubernetes.io/instance=temporaltest"
 kubectl exec -it services/temporaltest-admintools /bin/bash
 tctl namespace register default
```

* Create the worker for the helloworld workflow

```shell
 # this port-forward allows to use the samples directly from the repo (without edit the client.Options values)
 kubectl port-forward services/temporaltest-frontend-headless 7233:7233 &
 git clone https://github.com/temporalio/samples-go.git
 cd samples-go
 go run helloworld/worker/main.go
 # [expected output]
 # 2020/10/26 19:16:13 INFO  No logger configured for temporal client. Created default one.
 # 2020/10/26 19:16:13 INFO  Started Worker Namespace default TaskQueue hello-world WorkerID 10709@ws-steran@
```

* Another shell to execute the workflow activity

```shell
 cd /path/to/samples-go
 go run helloworld/starter/main.go
 # [expected output]
 # 2020/10/26 19:17:08 INFO  No logger configured for temporal client. Created default one.
 # 2020/10/26 19:17:08 Started workflow WorkflowID hello_world_workflowID RunID 2cab124b-0590-4072-b5d3-8b81763c0efc
 # 2020/10/26 19:17:08 Workflow result: Hello Temporal!
```

* Finally in the first shell close the worker and the port-forward

```shell
 ^C     # ctrl+c close worker
 fg 1   # get background task
 ^C     # ctrl+c close kubectl forward
```


## Hello world temporal sample with openfaas function

For testing purposes the openfaas function will:

- Register the workflow worker and activity at launch
- Starts a http server
- Expose a http endpoint to execute the workflow activity

Create a new function with the golang-middleware template from store. Golang and java are the two sdks available for temporal at this time:

```shell
 faas template store pull golang-middleware
 
 faas-cli new --lang golang-middleware temporalio-helloworld --prefix=kammin --gateway=gateway.openfaas.local
```

Add a goroutine to launch the worker in the main function on the template:

```golang
// template/golang-middleware/main.go
func main() {
	readTimeout := parseIntOrDurationValue(os.Getenv("read_timeout"), defaultTimeout)
	writeTimeout := parseIntOrDurationValue(os.Getenv("write_timeout"), defaultTimeout)

	s := &http.Server{
		Addr:           fmt.Sprintf(":%d", 8082),
		ReadTimeout:    readTimeout,
		WriteTimeout:   writeTimeout,
		MaxHeaderBytes: 1 << 20, // Max header of 1MB
	}

	go function.Workflow()

	http.HandleFunc("/", function.Handle)

	listenUntilShutdown(s, writeTimeout)
}
```

Write the function handler. This is the same code used before:

```golang
// temporalio-helloworld/handler.go
package function

import (
	"context"
	"fmt"
	"log"
	"os"

	"go.temporal.io/sdk/client"
	"go.temporal.io/sdk/worker"

	"net/http"

	"github.com/temporalio/samples-go/helloworld"
)

const (
	// hostport is the host:port which is used if not passed with options.
	hostport = "temporaltest-frontend-headless.default.svc.cluster.local:7233"
	// namespace is the namespace name which is used if not passed with options.
	namespace = "default"
)

func getEnv(key, fallback string) string {
	if value, ok := os.LookupEnv(key); ok {
		return value
	}
	return fallback
}

func Workflow() {

	c, err := client.NewClient(client.Options{HostPort: getEnv("temporal-dns", hostport)})
	if err != nil {
		log.Fatalln("Unable to create client", err)
	}
	defer c.Close()

	w := worker.New(c, "hello-world", worker.Options{})

	w.RegisterWorkflow(helloworld.Workflow)
	w.RegisterActivity(helloworld.Activity)

	err = w.Run(worker.InterruptCh())
	if err != nil {
		log.Fatalln("Unable to start worker", err)
	}

}

func Handle(rw http.ResponseWriter, r *http.Request) {

	// The client is a heavyweight object that should be created once per process.
	c, err := client.NewClient(client.Options{HostPort: getEnv("temporal-dns", hostport)})
	if err != nil {
		log.Fatalln("Unable to create client", err)
	}
	defer c.Close()

	workflowOptions := client.StartWorkflowOptions{
		ID:        "hello_world_workflowID",
		TaskQueue: "hello-world",
	}

	we, err := c.ExecuteWorkflow(context.Background(), workflowOptions, helloworld.Workflow, "Temporal")
	if err != nil {
		log.Fatalln("Unable to execute workflow", err)
	}

	log.Println("Started workflow", "WorkflowID", we.GetID(), "RunID", we.GetRunID())

	// Synchronously wait for the workflow completion.
	var result string
	err = we.Get(context.Background(), &result)
	if err != nil {
		log.Fatalln("Unable get workflow result", err)
	}
	log.Println("Workflow result:", result)

	rw.Header().Set("Content-Type", "application/json")
	rw.Write([]byte(fmt.Sprintf(`{"success":true, "message": "%s"}`, result)))

}
```

One command is all we need to build, push and deploy our new function:

```shell
 faas-cli up -f temporalio-helloworld.yml --build-arg GO111MODULE=on
```

> If you cant push the image to a remote registry change the openfaas deployment value to 'image_pull_policy=IfNotPresent'. Then do 'faas-cli build' and 'k3d image import' instead. This and using private registries on the openfaas docs.


Check the function is depoyed and the workflow registration in the function logs:

```shell
 faas-cli list
 # [expected output]
 # Function                      	Invocations    	Replicas
 # figlet                        	1              	1    
 # temporalio-helloworld         	0              	1   
```

```shell
 faas-cli logs temporalio-helloworld
 # [expected output]
 # 2020-10-26T20:04:47Z 2020/10/26 20:04:47 Started logging stderr from function.
 # 2020-10-26T20:04:47Z 2020/10/26 20:04:47 Started logging stdout from function.
 # 2020-10-26T20:04:47Z Forking - ./handler []
 # 2020-10-26T20:04:47Z 2020/10/26 20:04:47 OperationalMode: http
 # 2020-10-26T20:04:47Z 2020/10/26 20:04:47 Timeouts: read: 10s, write: 10s hard: 10s.
 # 2020-10-26T20:04:47Z 2020/10/26 20:04:47 Listening on port: 8080
 # 2020-10-26T20:04:47Z 2020/10/26 20:04:47 Writing lock-file to: /tmp/.lock
 # 2020-10-26T20:04:47Z 2020/10/26 20:04:47 Metrics listening on port: 8081
 # 2020-10-26T20:04:47Z 2020/10/26 20:04:47 stdout: 2020/10/26 20:04:47 INFO  No logger configured for temporal client. Created default one.
 # 2020-10-26T20:04:48Z 2020/10/26 20:04:48 stdout: 2020/10/26 20:04:48 INFO  Started Worker Namespace default TaskQueue hello-world WorkerID 14@temporalio-helloworld-77d4bc6b99-kt8hj@
```

Execute the function:

```shell
 curl http://gateway.openfaas.local/function/temporalio-helloworld
 # [expected output]
 # {"success":true, "message": "Hello Temporal!"}
```

Lot of information is provided in the [temporal UI](http://dashboard.temporal.local/namespaces/default):

![](/images/temporal-result1.png)
![](/images/temporal-result2.png)

My next move is to review the temporal docs for complex use cases like versioning workflows or saga patterns.


## Uninstall

To uninstall the lab delete the entries from /etc/hosts and the k3d cluster:

```shell
k3d cluster delete openfaas-temporalio
```