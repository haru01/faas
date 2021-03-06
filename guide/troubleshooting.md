# Troubleshooting guide

## Healthcheck

Most problems reported via GitHub or Slack stem from a configuration problem or issue with a function. Here is a checklist of things you can try before digging deeper:

Checklist:
* [ ] All core services are deployed: i.e. gateway
* [ ] Check functions are deployed and started
* [ ] Check request isn't timing out at the gateway or the function level

## Docker Swarm

### List all functions

```
$ docker service ls
```

You are looking for 1/1 for the replica count of each service listed.

### Find a function's logs

```
$ docker service logs --tail 100 FUNCTION
```

### Find out if a function failed to start

```
$ docker service ps --no-trunc=true FUNCTION
```

### Stop and remove OpenFaaS

```
$ docker stack rm func
```

If you have additional services / functions remove the remaining ones like this:

```
$ docker service ls -q | xargs docker service rm
```

*Use with caution*

## Kubernetes

### List all functions

```
$ kubectl get deploy
```

### Find a function's logs

```
$ kubectl logs deploy/FUNCTION
```

### Find out if a function failed to start

```
$ kubectl describe deploy/FUNCTION
```

### Remove the OpenFaaS deployment

```
$ kubectl delete -f faas.yml,monitoring.yml,rbac.yml
```

If you're using the async stack remove it this way:

```
$ kubectl delete -f faas.async.yml,monitoring.yml,rbac.yml,nats.yml
```

## Timeouts

Default timeouts are configured at the HTTP level for the gateway and watchdog. You can also enforce a hard-timeout for your function.

For watchdog configuration see the [README](https://github.com/openfaas/faas/tree/master/watchdog).

For the gateway set the following environmental variables:

* read_timeout, write_timeout - the default for both is "8" - seconds.

## Watchdog

### Debug your function without deploying it

Here's an example of how you can deploy a function without using an orchestrator and the API gateeway. It is especially useful for testing:

```
$ docker run --name debug-alpine \
  -p 8081:8080 -ti functions/alpine:latest sh
# fprocess=date fwatchdog &
```

Now you can access the function with one of the supported HTTP methods such as GET/POST etc:

```
$ curl -4 localhost:8081
```

### Edit your function without rebuilding it

You can bind-mount code straight into your function and work with it locally, until you are ready to re-build. This is a common flow with containers, but should be used sparingly.

Within the CLI directory for instance:

Build the samples:

```
$ git clone https://github.com/openfaas/faas-cli && \
  cd faas-cli
$ faas-cli -action build -f ./samples.yml
```

Now work with the Python-hello sample, with the code mounted live:

```
$ docker run -v `pwd`/sample/url-ping/:/root/function/ \
  --name debug-alpine -p 8081:8080 -ti alexellis/faas-url-ping sh
$ touch ./function/__init__.py
# fwatchdog
```

Now you can start editing the code in the sample/url-ping folder and it will reload live for every request.

```
$ curl localhost:8081 -d "https://www.google.com"
Handle this -> https://www.google.com
https://www.google.com => 200
```

Now you can edit handler.py and you'll see the change immediately:

```
$ echo "def handle(req):" > sample/url-ping/handler.py
$ echo '    print("Nothing to see here")' >> sample/url-ping/handler.py
$ curl localhost:8081 -d "https://www.google.com"
Nothing to see here
```
