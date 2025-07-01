========================
Prometheus on Kubernetes
========================

:date: 2016-11-13
:tags: prometheus, kubernetes, monitoring, golang, Google Container Engine
:category: Infrastructure
:slug: prometheus-on-kubernetes

`Prometheus <https://prometheus.io/>`_ is a monitoring toolkit.
Let's set it up on Kubernetes and test how it works by scraping HTTP request metrics
from `hello web application <https://github.com/marselester/prometheus-on-kubernetes>`_
which also runs in the same cluster.

First of all, we need Kubernetes cluster running. It's easy to bootstrap one via `Google Container Engine <https://console.cloud.google.com>`_.

.. code-block:: console

    $ gcloud container clusters create my-k8

The command above might complain that zone is not currently set.
Choose the one you like and try to create a cluster again.

.. code-block:: console

    $ gcloud compute zones list
    $ gcloud config set compute/zone asia-east1-b

You should now have a Kubernetes cluster.
If there are any issues, check out `Udacity course <https://www.udacity.com/course/scalable-microservices-with-kubernetes--ud615>`_ which helped me run Kubernetes in GKE.

First Start
-----------

Prometheus is available as Docker image and can be run locally quickly.

.. code-block:: console

    $ docker run -p 9090:9090 prom/prometheus:v1.2.1

When a container is started, the Prometheus expression browser should be accessible on `http://localhost:9090`.
Now let's achieve the same results with Kubernetes.

.. code-block:: console

    $ kubectl run prometheus-deployment --image=prom/prometheus:v1.2.1 --port=9090

The command above created `Kubernetes Deployment <http://kubernetes.io/docs/user-guide/deployments/>`_
that runs `prometheus` Docker image.

By default deployed applications are visible only inside the Kubernetes cluster.
To see whether Prometheus started up, we can run a proxy between terminal and Kubernetes cluster.

.. code-block:: console

    $ kubectl proxy
    Starting to serve on 127.0.0.1:8001

In another cloud shell, we can call Kubernetes API.

.. code-block:: console

    $ kubectl get pods
    NAME                                     READY     STATUS    RESTARTS   AGE
    prometheus-deployment-2234044252-of29y   1/1       Running   0          8m

    $ curl http://127.0.0.1:8001/api/v1/proxy/namespaces/default/pods/prometheus-deployment-2234044252-of29y:9090/metrics

Another option is to run a port forwarding:

.. code-block:: console

    $ kubectl port-forward prometheus-deployment-2234044252-of29y 8080:9090
    $ curl http://127.0.0.1:8080/metrics

Check Prometheus container logs:

.. code-block:: console

    $ kubectl logs -f po/prometheus-deployment-2234044252-of29y
    time="2016-10-29T05:02:38Z" level=info msg="Starting prometheus (version=1.2.1, branch=master, revision=dd66f2e94b2b662804b9aa1b6a50587b990ba8b7)" source="main.go:75"

And finally we can run a shell inside the Pod's container and have a look at Prometheus config file.

.. code-block:: console

    $ kubectl exec prometheus-deployment-2234044252-of29y -it /bin/sh
    root$ cat /etc/prometheus/prometheus.yml

Its advisable to describe Deployments in configuration files,
so we can have a better visibility and version control over our cluster.
The following Deployment manifest is similar to what we achieved with `kubectl run` command.

.. code-block:: yaml

    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: prometheus-deployment
    spec:
      replicas: 1 # tells deployment to run 1 pod matching the template below
      template: # crete pods using pod definition in this template
        metadata:
          labels: # these key value pairs will be attached to pods
            app: prometheus-server
        spec:
          containers:
            - name: prometheus
              image: prom/prometheus:v1.2.1
              ports:
                - containerPort: 9090 # port we open in the container

Let's delete `prometheus-deployment` we created via `kubectl run` command

.. code-block:: console

    $ kubectl delete deployment prometheus-deployment

and re-create the Deployment from a file (it is available in git repository):

.. code-block:: console

    $ git clone https://github.com/marselester/prometheus-on-kubernetes.git
    $ cd ./prometheus-on-kubernetes/
    $ kubectl create -f kube/prometheus/deployment-v1.yml
    $ kubectl get deployments
    NAME                    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    prometheus-deployment   1         1         1            0           40s

Prometheus Service
------------------

We have a Prometheus Pod running.
Now we need `Kubernetes Service <http://kubernetes.io/docs/user-guide/services/>`_ to let external clients access it.

.. code-block:: console

    $ kubectl expose deployment prometheus-deployment --type=NodePort --name=prometheus-service

The assigned port can be found in `NodePort` output of Service description:

.. code-block:: console

    $ kubectl describe service prometheus-service
    # ...
    NodePort:               <unset> 32514/TCP
    # ...

We have exposed the Service on an external port `32514` on all nodes in our cluster.
Now create a firewall rule to allow external traffic.

.. code-block:: console

    $ gcloud compute firewall-rules create prometheus-nodeport --allow=tcp:32514

Let's use one of the external IPs from our Kubernetes cluster instances to see Prometheus expression browser.

.. code-block:: console

    $ gcloud compute instances list

You should be able to access Prometheus on `http://<EXTERNAL_IP>:32514`.
Let's delete the Service and create it via Service config file.

.. code-block:: console

    $ kubectl delete service prometheus-service
    $ kubectl create -f kube/prometheus/service-v1.yml

Since the Prometheus Pod exposes `9090` port and has `app: prometheus-server` label,
our config should be as following:

.. code-block:: yaml

    apiVersion: v1
    kind: Service
    metadata:
      name: prometheus-service
    spec:
      selector: # exposes any pods with the following labels as a service
        app: prometheus-server
      type: NodePort
      ports:
        - port: 80 # this Service's port (cluster-internal IP clusterIP)
          targetPort: 9090 # pods expose this port
          # Kubernetes master will allocate a port from a flag-configured range (default: 30000-32767),
          # or we can set a specific port number (in our case).
          # Each node will proxy 32514 port (the same port number on every node) into this service.
          # Note that this Service will be visible as both NodeIP:nodePort and clusterIp:port
          nodePort: 32514

Prometheus Config
-----------------

So far we've been using the default Prometheus config which is part of a Docker image.
For sure we will need to update it so Prometheus can collect metrics from our example app.
Let's take the default config as a starting point and store it in
`Kubernetes ConfigMap <http://kubernetes.io/docs/user-guide/configmap/>`_.
The config can be copied from the running container or from the git repository.

.. code-block:: console

    $ kubectl exec prometheus-deployment-2234044252-of29y -it cat /etc/prometheus/prometheus.yml

Next we need to create a ConfigMap entry for the `prometheus.yml` file:

.. code-block:: console

    $ kubectl create configmap prometheus-server-conf --from-file=prometheus.yml=kube/prometheus/config-v1.yml

Now let's mount `prometheus-server-conf` ConfigMap volume to our Prometheus Pod

.. code-block:: console

    $ kubectl apply -f kube/prometheus/deployment-v2.yaml

and store metrics in emptyDir volume, so we don't lose them when a container in the Pod crashes.

.. code-block:: console

    $ kubectl apply -f kube/prometheus/deployment-v3.yml

Sending App Metrics
-------------------

We have a Prometheus running in Kubernetes but we have no application to monitor.
I wrote `hello-app/v1 <https://github.com/marselester/prometheus-on-kubernetes/blob/master/hello-app/v1/main.go>`_ web app that exposes `/hello` HTTP endpoint.

`Björn Rabenstein in his talk <https://youtu.be/HkEZ1LJ7kzQ?list=PLDWZ5uzn69eyh791ZTkEA9OaTxVpGY8_g>`_
explains how to instrument your code to expose metrics to Prometheus.
Our application is not forced to use Prometheus client to expose metrics.
We can create `/metrics` HTTP endpoint manually in the following text format:

.. code-block:: text

    # HELP http_requests_total Number of HTTP requests.
    # TYPE http_requests_total counter
    http_requests_total{code="200",method="get"} 2384

But it's much easier to use a library
(see `hello-app/v2 <https://github.com/marselester/prometheus-on-kubernetes/blob/master/hello-app/v2/main.go>`_).

.. code-block:: go

    import "github.com/prometheus/client_golang/prometheus/promhttp"
    // ...
    http.Handle("/metrics", promhttp.Handler())

Now the app has `/metrics` endpoint with Go runtime metrics (number of goroutines, GC statistics).

Our app exposes an important endpoint `/hello`.

.. code-block:: go

    func helloHandler(w http.ResponseWriter, r *http.Request) {
        status := doSomeWork()
        w.WriteHeader(status)
        w.Write([]byte("Hello, World!\n"))
    }

Let's instrument `helloHandler()` to count HTTP requests and their durations.
First, we need to define metrics.

.. code-block:: go

    import "github.com/prometheus/client_golang/prometheus"

    var (
        // How often our /hello request durations fall into one of the defined buckets.
        // We can use default buckets or set ones we are interested in.
        duration = prometheus.NewHistogram(prometheus.HistogramOpts{
            Name:    "hello_request_duration_seconds",
            Help:    "Histogram of the /hello request duration.",
            Buckets: []float64{0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10},
        })
        // Counter vector to which we can attach labels. That creates many key-value
        // label combinations. So in our case we count requests by status code separetly.
        counter = prometheus.NewCounterVec(
            prometheus.CounterOpts{
                Name: "hello_requests_total",
                Help: "Total number of /hello requests.",
            },
            []string{"status"},
        )
    )

    // init registers Prometheus metrics.
    func init() {
        prometheus.MustRegister(duration)
        prometheus.MustRegister(counter)
    }

Second, measure a request duration in seconds and increase the counter in the `helloHandler()` function.

.. code-block:: go

    func helloHandler(w http.ResponseWriter, r *http.Request) {
        var status int

        defer func(begun time.Time) {
            duration.Observe(time.Since(begun).Seconds())

            // hello_requests_total{status="200"} 2385
            counter.With(prometheus.Labels{
                "status": fmt.Sprint(status),
            }).Inc()
        }(time.Now())

        status = doSomeWork()
        w.WriteHeader(status)
        w.Write([]byte("Hello, World!\n"))
    }

`hello-app/v3 <https://github.com/marselester/prometheus-on-kubernetes/blob/master/hello-app/v3/main.go>`_
is used in further examples.

Hello App Demo
--------------

Next step is to run the web app in Kubernetes.
A Docker image of the `hello-app/v3` is available on Docker Hub.

.. code-block:: console

    $ kubectl apply -f kube/hello/deployment-v1.yml
    $ kubectl port-forward hello-deployment-1471727270-eaknp 8000:8000
    $ curl localhost:8000/hello
    Hello, World!
    $ curl localhost:8000/metrics
    ...
    # HELP hello_request_duration_seconds Histogram of the /hello request duration.
    # TYPE hello_request_duration_seconds histogram
    hello_request_duration_seconds_bucket{le="0.01"} 0
    hello_request_duration_seconds_bucket{le="0.025"} 0
    hello_request_duration_seconds_bucket{le="0.05"} 0
    hello_request_duration_seconds_bucket{le="0.1"} 1
    hello_request_duration_seconds_bucket{le="0.25"} 1
    hello_request_duration_seconds_bucket{le="0.5"} 1
    hello_request_duration_seconds_bucket{le="1"} 1
    hello_request_duration_seconds_bucket{le="2.5"} 1
    hello_request_duration_seconds_bucket{le="5"} 1
    hello_request_duration_seconds_bucket{le="10"} 1
    hello_request_duration_seconds_bucket{le="+Inf"} 1
    hello_request_duration_seconds_sum 0.083953974
    hello_request_duration_seconds_count 1
    # HELP hello_requests_total Total number of /hello requests.
    # TYPE hello_requests_total counter
    hello_requests_total{status="500"} 1

The Service creation is similar to what we have already done before.
We use `32515` NodePort here.

.. code-block:: console

    $ kubectl apply -f kube/hello/service-v1.yml
    $ gcloud compute firewall-rules create hello-nodeport --allow=tcp:32515

Now it is possible to see the app's metrics from Internet.

.. code-block:: console

    $ curl http://<EXTERNAL_IP>:32515/metrics

Hello Prometheus
----------------

Since the web app is run on Kubernetes, we can configure Prometheus
to scrape metrics from `/metrics` HTTP endpoint of hello-app Pods.

.. code-block:: yaml

    global:
      scrape_interval: 5s
      evaluation_interval: 5s

    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
          - targets: ['localhost:9090']

      - job_name: 'hello'
        # The information to access the Kubernetes API to discover targets.
        kubernetes_sd_configs:
          - api_servers:
            - 'https://kubernetes.default.svc'
            # Prometheus assumes it is being run inside a Kubernetes pod.
            in_cluster: true
            # Only pods should be discovered.
            role: pod
        # Prometheus collects metrics from pods with "app: hello-server" label.
        # Prometheus gets 'hello_requests_total{status="500"} 1'
        # from hello:8000/metrics and adds "job" and "instance" labels, so it becomes
        # 'hello_requests_total{instance="10.16.0.10:8000",job="hello",status="500"} 1'.
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_app]
            regex: hello-server
            action: keep

Update a ConfigMap of Prometheus config and re-create a Prometheus Pod so it picks up changes.

.. code-block:: console

    $ kubectl create configmap prometheus-server-conf \
        --from-file=prometheus.yml=kube/prometheus/config-v2.yaml \
        -o yaml \
        --dry-run | kubectl replace -f -

Finally, the app's metrics should show up in Prometheus expression browser `http://<EXTERNAL_IP>:32514`.

You can try to query `hello_requests_total` which shows how many requests we've served since the beginning.
Let's see how many requests we've served in last 5 minutes normalised per second (QPS)
with `rate(hello_requests_total[5m])` query.

Here I gave an example of how to run Prometheus in Kubernetes cluster
and collect metrics from a simple web app.
However recently I have encountered CoreOS `kube-prometheus <https://github.com/coreos/kube-prometheus>`_
which makes it easier. Have a look at `The Prometheus Operator: Managed Prometheus setups for Kubernetes <https://coreos.com/blog/the-prometheus-operator.html>`_ for more details.
