# Kubernetes-intro

![](https://github.com/dabble-be/kubernetes-intro/raw/main/.assets/images/kubernetes-logo.500x485.png =300x300)

### System Requirements

This exercise has been constructed using the following software versions. These are not hard requirements.
However; This exercise does assume a working Kubernetes cluster.

  - [BASH](https://www.gnu.org/software/bash/) | `5.0.17(1)-release` | Bash is the GNU Project's shell.
  - [curl](https://curl.se/) | `7.68.0` | command line tool and library for transferring data with URLs.
  - [git](https://git-scm.com/) | `2.25.1` | Git is a free and open source distributed version control system designed to handle everything from small to very large projects with speed and efficiency.
  - [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/) | `v1.22` | lets you control Kubernetes clusters.
  - [tee](https://www.gnu.org/software/coreutils/manual/html_node/Introduction.html) | `8.30` | The tee command copies standard input to standard output and also to any files given as arguments.

## Running a web application

Let's dive straight in and start creating our first Kubernetes manifest file:

```sh
cat <<EOF | tee hello-server.deployment.yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-server
  labels:
    app: hello-server
EOF
```

This first chunk in `hello-server.deployment.yml` is where we define that our manifest is of type `Deployment` (which is part of `apiVersion: apps/v1`). We add a `name`, `namespace` and some labels.

```sh
cat <<EOF | tee --append hello-server.deployment.yml
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-server
EOF
```

`spec` is the bread and butter of our Kubernetes resource files. It's where we specify resource-specific options. Here we start of adding the amount of `replicas` we want to run for our deployment. We also add selector labels, this makes it easier to find resources created by this deployment.

```sh
cat <<EOF | tee --append hello-server.deployment.yml
  template:
    metadata:
      labels:
        app: hello-server
EOF
```

Next up, still inside of the deployments `spec:`, we add a `template` option, where we'll add all values we want to be passed into the generated pods.

We are adding the same labels here, that will make it easy to create a load balancer later on.

```sh
cat <<EOF | tee --append hello-server.deployment.yml
    spec:
      containers:
      - name: hello-app
        image: us-docker.pkg.dev/google-samples/containers/gke/hello-app:1.0
EOF
```

Next up, let's add a pod-`spec:` to the `template:`, also to ensure our pods get the correct parameters set.
The snippet adds 1 container with name `hello-app` and image `.../hello-app:1.0`.

```sh
cat <<EOF | tee --append hello-server.deployment.yml
        resources:
          limits:
            cpu: 200m
            memory: 200Mi
EOF
```

We do want to limit the resources available to this pod. For this intro's sake, let's add resource limit's. These will limit the amount of resources available to the created pods, or kill those who exceed it.

- CPU: set to 200 millicpu (or 1%)
- Memory: set to 200 mebibyte (about 210 Megabyte)

```sh
kubectl apply -f hello-server.deployment.yml
```


_output:_

```log
deployment.apps/hello-server created
```

Get the name of the resulting pod:

```sh
kubectl get pods
```

_output:_

```log
dabbler@shell:~$ kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
shell                           1/1     Running   0          55m
hello-server-5997b67bfd-j5dbm   1/1     Running   0          4s
```

For the sake of this intro, I'll be copying `hello-server-5997b67bfd-j5dbm` and using it in the following commands. Let's put the name in a variable for now:

```sh
targetPod="hello-server-5997b67bfd-j5dbm"
```

Now that we have stored our pod's name, let's have a look at its logs:

```sh
kubectl logs "$targetPod"
```

_output:_

```log
2021/11/29 15:33:22 Server listening on port 8080
```

Seems like our app is running properly, and listening on port `8080`. This pod is running deep within our Kubernetes cluster though. How are we able to access the service?

```sh
targetIP="$( kubectl get pod "$targetPod" --template='{{.status.podIP }}' )"; echo "$targetIP"
```

_output:_

```log
10.42.0.30
```

The above command stores the IPv4 address for our pod in variable `targetIP`. From within the cluster (and the same name space), we can access this pod's port directly:

Let's have a look at the response, with `curl`:

```sh
curl -i "http://$targetIP:8080"
```

_output:_

```log
HTTP/1.1 200 OK
Date: Mon, 29 Nov 2021 15:59:46 GMT
Content-Length: 69
Content-Type: text/plain; charset=utf-8

Hello, world!
Version: 1.0.0
Hostname: hello-server-5997b67bfd-j5dbm
```

Note that the `Hostname:` in the output is the name of our pod.


Good thing we managed to validate our application is running properly. We don't want to lookup the IP address of a node each time we want to access the service. Let's add some service discovery.

Introducing the `service` resource. Let's start constructing it:

```sh
cat <<EOF | tee hello-server.service.yml
apiVersion: v1
kind: Service
metadata:
  name: hello-server
  labels:
    app: hello-server
EOF
```

This time, we are creating a resource of kind `Service`. Metadata is very similar to the one we used in our `deployment` earlier. We name this service after our app, and add the common labels.

```sh
cat <<EOF | tee --append hello-server.service.yml
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
EOF
```

We begin the `spec`ification of our service resource by adding a port to its `ports:` array. In this case, our app is listening on port 8080, and will be made available to the cluster (it receives a `clusterIP`) on port 80.
Now that our service knows which ports it's needs, we need to add a way to find our `pods`.

```sh
cat <<EOF | tee --append hello-server.service.yml
  selector:
    app: hello-server
EOF
```

We do that by adding a `selector:`. This selector accepts labels that must match existing pods. This way, it can build an internal list of addresses to forward traffic to.

Let's test the results of our `service`'s query before we add it to our Kubernetes cluster:

```sh
kubectl get pods --selector app=hello-server
```

_output:_

```log
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-5997b67bfd-j5dbm   1/1     Running   0          84m
```

That looks about right. We only did start one replica.

Apply the service we just created a manifest for:

```sh
kubectl apply -f ./hello-server.service.yml
```

_output:_

```log
service/hello-server created
```

To have a look at the details that have been created by the `service`, `describe` it:

```sh
kubectl describe service hello-server
```

_output:_

```log
Name:              hello-server
Namespace:         demo
Labels:            app=hello-server
Annotations:       <none>
Selector:          app=hello-server
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.43.6.202
IPs:               10.43.6.202
Port:              <unset>  80/TCP
TargetPort:        8080/TCP
Endpoints:         10.42.0.30:8080
Session Affinity:  None
Events:            <none>
```

Copy the IP in the output:

```log
IP:                10.43.6.202
```

We can use this IP from within the cluster to access the service:

```sh
curl -iL http://10.43.6.202
```

```log
HTTP/1.1 200 OK
Date: Mon, 29 Nov 2021 17:04:11 GMT


Hello, world!
Version: 1.0.0
Hostname: hello-server-5997b67bfd-j5dbm
```

That works great! However, we don't want to be searching for an IP each time. Let's figure out what this service offers us in terms of service discovery:

```sh
curl -iL http://hello-server
```

_output:_

```log
HTTP/1.1 200 OK
Date: Mon, 29 Nov 2021 17:12:18 GMT
Content-Length: 69
Content-Type: text/plain; charset=utf-8

Hello, world!
Version: 1.0.0
Hostname: hello-server-5997b67bfd-j5dbm
```

> If you want to access this DNS entry from another Kubernetes namespace, you can add an apex to the address. The address will be of the following form: `<servicename>.<namespace>.svc.cluster.local`.
> In the example, this would be: `hello-server.demo.svc.cluster.local`

From within the same namespace, we can simply use the name of the service we created to resolve the correct IP.
Now let's increase it's usefulness even more; By scaling the deployment:

```sh
kubectl scale deployment hello-server --replicas=5
```

_output:_

```log
deployment.apps/hello-server scaled
```

The `deployment` will now aim to have 5 replicas available of our `hello-server` at all times. You can follow the state of the deployment using:

```sh
kubectl rollout status deployment hello-server
```

```log
Waiting for deployment "hello-server" rollout to finish: 1 of 5 updated replicas are available...
Waiting for deployment "hello-server" rollout to finish: 2 of 5 updated replicas are available...
Waiting for deployment "hello-server" rollout to finish: 3 of 5 updated replicas are available...
Waiting for deployment "hello-server" rollout to finish: 4 of 5 updated replicas are available...
deployment "hello-server" successfully rolled out
```

Now we can taste the fruits of our labor; Perform a few requests to the service address we used earlier:

```sh
for i in $( seq 2 ); do curl -iL http://hello-server; done
```

_output:_

```log
HTTP/1.1 200 OK
Date: Mon, 29 Nov 2021 19:56:32 GMT
Content-Length: 69
Content-Type: text/plain; charset=utf-8

Hello, world!
Version: 1.0.0
Hostname: hello-server-5997b67bfd-7ms94
HTTP/1.1 200 OK
Date: Mon, 29 Nov 2021 19:56:32 GMT
Content-Length: 69
Content-Type: text/plain; charset=utf-8

Hello, world!
Version: 1.0.0
Hostname: hello-server-5997b67bfd-4vzpk
```

We can see now that we are getting 2 different `Hostname:` (`hello-server-5997b67bfd-7ms94` and `hello-server-5997b67bfd-4vzpk`). Feel free to try this a few times and check the output.
What we are seeing here is the beauty of services. As the service detected more pods labeled: `hello-server` after scaling, it added the new pods to the service's `endpoint` list.

This service is getting along fine. It's a good time to see if we can make the service available to the outside world. Kubernetes proposes a resource for this: `ingress`. This is yet another API object: Aimed at managing access to the services in a cluster.

Create the ingress manifest step by step:

```sh
cat <<EOF | tee hello-server.ingress.yml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-server
EOF
```

```sh
cat <<EOF | tee --append hello-server.ingress.yml
spec:
  rules:
  - http:
      paths:
      - backend:
          service:
            name: hello-server
            port:
              number: 80
EOF
```

And add the last part to the ingress manifest:

```sh
cat <<EOF | tee --append hello-server.ingress.yml
        path: /http/$STUDENT_ID
        pathType: Prefix
EOF
```

This will add the last parameters we need. In case of this example, the service will be available here;

Open another browser tab at: `https://lab.dabble.be/http/demo/`. (in case of dabble labs)

> (Replace `demo` in the URL with your environments name; Run `echo "$STUDENT_ID"` to get your environment name on dabble-shell)
> `echo "https://lab.dabble.be/http/$STUDENT_ID"` should do the trick.
> If you are using the dabble lab, you can use the above URI else, you'll need to refer to the documentation of your platform.
> Ingress resources are a standard though, so it shouldn't differ between platforms.

Refresh a few times to see the value of `Hostname:` differ like when we used `curl` earlier to access our service directly.

Good. We've set up and application, scaled it and exposed it to the outside world.

After developing hard, a new version of our application has been released: `hello-app:2.0`.

`Deployments` perform rolling-release style deployments by default, so let's take a look at what happens when we change our image's tag from `1.0` to `2.0` in our `hello-server.deployment.yml`.

```sh
sed -i 's/hello-app:1.0/hello-app:2.0/' hello-server.deployment.yml
sed -i 's/replicas: 1/replicas: 5/' hello-server.deployment.yml
```

Take a last look at your existing pods:

```sh
kubectl get pods -l app=hello-server
```

_output:_

```log
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-5997b67bfd-tc57l   1/1     Running   0          3m
hello-server-5997b67bfd-vchwx   1/1     Running   0          3m
hello-server-5997b67bfd-qfhmj   1/1     Running   0          3m
hello-server-5997b67bfd-g697k   1/1     Running   0          3m
hello-server-5997b67bfd-ghh2d   1/1     Running   0          3m
```

Now, apply the changes to the deployment file:

```sh
kubectl apply -f hello-server.deployment.yml
```

_output:_

```log
deployment.apps/hello-server configured
```

This will gradually increase the amount of running replicas for the new version of the deployment, until it reaches the requested amount.

If you are hungry for more exercises, please check out the `Further dabbling` section.

# Conclusion

In this intro, we observed the most basic resources to build stateless applications on Kubernetes. I hope you've gotten a taste of this amazing software, and enjoyed it.

Kubernetes is more complex than, for example, docker swarm. However, it's far more extensive and extendable. It's a framework for automating application deployment, scaling and management.

# Further dabbling

## More
More coming soon!
