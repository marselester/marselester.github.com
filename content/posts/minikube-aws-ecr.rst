========================================
Minukube & Amazon EC2 Container Registry
========================================

:date: 2016-12-05
:tags: kubernetes, aws, minukube
:category: Infrastructure
:slug: minukube-aws-ecr

Minukube_ is an easy way to run Kubernetes locally.
When we want to build a Docker image in Minukube (so Kubernetes has an access to it),
we can configure our Docker client to communicate with the Minikube Docker daemon.

.. code-block:: console

    $ minikube start
    Starting local Kubernetes cluster...
    Kubectl is now configured to use the cluster.
    $ eval $(minikube docker-env)
    $ docker build -t fizz/bazz:latest .

What if we need to create a Kubernetes Deployment which pulls a Docker image from AWS ECR?

.. code-block:: console

    $ kubectl create -f - <<<'
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: my-deployment
    spec:
      replicas: 1
      template:
        metadata:
          labels:
            app: my-app-name
        spec:
          containers:
            - name: my-container-name
              image: 4338991606.dkr.ecr.eu-west-1.amazonaws.com/fizz/bazz:latest
    '

Kubernetes will fail to get the image due to authentication error (no credentials).
One of the solutions is to use imagePullSecrets_.
The following command creates a secret for use with a Docker registry.

.. code-block:: console

    $ kubectl create secret docker-registry my-docker-credentials \
    	--docker-server=DOCKER_REGISTRY_SERVER \
    	--docker-username=DOCKER_USER \
    	--docker-password=DOCKER_PASSWORD \
    	--docker-email=DOCKER_EMAIL

Firstly, let's obtain **DOCKER_USER** and **DOCKER_PASSWORD**.
For that, we'll need **awscli** Python library and environment variables
**AWS_ACCESS_KEY_ID** and **AWS_SECRET_ACCESS_KEY** set up.

.. code-block:: console

    $ pip install awscli
    $ export AWS_ACCESS_KEY_ID='AKIAI44QH8DHBEXAMPLE'
    $ export AWS_SECRET_ACCESS_KEY='je7MtGbClwBF/2Zp9Utk/h3yCo8nvbEXAMPLEKEY'

Log in to Amazon ECR registry.
This aws command will display **$ docker login** command. From its output we conclude that
**DOCKER_USER** is **AWS**, **DOCKER_PASSWORD** is **SomeVeryLongToken** and **DOCKER_REGISTRY_SERVER**
is **https://4338991606.dkr.ecr.eu-west-1.amazonaws.com** respectively.

.. code-block:: console

    $ aws ecr get-login --region eu-west-1
    docker login -u AWS -p SomeVeryLongToken -e none https://4338991606.dkr.ecr.eu-west-1.amazonaws.com

Now it is time to create a docker-registry secret and update the Deployment manifest.
Note that we've added **imagePullSecrets** which references our private registry.

.. code-block:: console

    $ kubectl apply -f - <<<'
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: my-deployment
    spec:
      replicas: 1
      template:
        metadata:
          labels:
            app: my-app-name
        spec:
          containers:
            - name: my-container-name
              image: 4338991606.dkr.ecr.eu-west-1.amazonaws.com/fizz/bazz:latest
          imagePullSecrets:
            - name: my-docker-credentials
    '

Kubernetes should be able to pull the image from Amazon Container Registry.

.. _Minukube: http://kubernetes.io/docs/getting-started-guides/minikube/
.. _imagePullSecrets: http://kubernetes.io/docs/user-guide/images/#specifying-imagepullsecrets-on-a-pod
