=========================
Ambassador as API Gateway
=========================

:date: 2019-03-13
:tags: ambassador, kubernetes, api gateway, auth, rate limiting
:category: Infrastructure
:slug: apigate-ambassador

`API gateway <https://docs.microsoft.com/en-us/azure/architecture/microservices/design/gateway>`_
acts as a reverse proxy, routing API requests from clients to services.
Usually it also performs authentication and rate limiting, so the services behind the gate don't have to.
In this short tutorial we'll see how to achieve that with `Ambassador <https://getambassador.io/>`_.

The demo is based on a dummy `Traveling project <https://traveling.docs.apiary.io/>`_
where we have services to rent a car and book a hotel.

Firstly, we shall install Ambassador on Minikube_
by following `Ambassador user guide <https://www.getambassador.io/user-guide/getting-started/>`_.

.. code-block:: console

    $ minikube start
    Starting local Kubernetes v1.13.2 cluster...
    $ kubectl apply -f https://www.getambassador.io/yaml/ambassador/ambassador-rbac.yaml

Next, create Kubernetes service for Ambassador deployment of type NodePort, so it's easily accessible outside of the cluster.
It will be the entry point of our API gateway.

.. code-block:: console

    $ kubectl apply -f - <<<'
    apiVersion: v1
    kind: Service
    metadata:
      name: ambassador
    spec:
      selector:
        service: ambassador
      type: NodePort
      ports:
        - port: 80
      # Propagate the original source IP of the client.
      externalTrafficPolicy: Local
    '
    $ AMBASSADORURL=$(minikube service --url ambassador)

Note, there is a diagnostic web UI at `$AMBASSADORURL/ambassador/v0/diag/`.
You should restrict access to it from your custom auth service.

Travel Apps
-----------

Let's deploy the travel apps on Kubernetes. For simplicity, you can use the images uploaded to Docker Hub.
If you prefer to build Docker images yourself, make sure they are accessible from Minikube:
configure Docker client to communicate with the Minikube Docker daemon.
Note, you can restore your local Docker environment with `eval $(minikube docker-env --unset)` later on.

.. code-block:: console

    $ git clone https://github.com/marselester/apigate.git
    $ cd ./apigate/
    $ eval $(minikube docker-env)
    $ docker build --tag=marselester/travel-hotel:v1.0.0 --file=docker/hotel.Dockerfile .
    $ kubectl apply -f k8s-ambassador/hotel/deployment.yml
    $ kubectl apply -f k8s-ambassador/hotel/service.yml

The hotel app should be running:

.. code-block:: console

    $ kubectl get pods -l app=hotel
    hotel-api-8d7d59b69-bcsvh
    $ kubectl port-forward hotel-api-8d7d59b69-bcsvh 8000
    $ curl localhost:8000/v1/hotels/bookings/7b4fc183-ee67-494d-9715-3510c6d8f2ef
    {
        "id": "7b4fc183-ee67-494d-9715-3510c6d8f2ef",
        "hotel_id": "046d471d-70c7-4595-80cc-266d3e6e07fa",
        "status": "confirmed"
    }

Now, it's time to put the `hotel` Service behind the gateway by adding annotations to the service.

.. code-block:: yaml

    apiVersion: v1
    kind: Service
    metadata:
      name: hotel
      annotations:
        getambassador.io/config: |
          ---
          apiVersion: ambassador/v1
          kind: Mapping
          name: hotel_mapping
          prefix: /v1/hotels
          rewrite: ""
          service: hotel
    spec:
      selector:
        app: hotel
      type: NodePort
      ports:
        - port: 80
          targetPort: 8000

.. code-block:: console

    $ kubectl apply -f k8s-ambassador/hotel/service-gate.yml

By default, Ambassador would rewrite `/v1/hotels` prefix to `/`.
With `rewrite: ""` directive we configure Ambassador to not change the prefix as
it forwards a request to the `hotel` service.
Let's confirm that hotel booking requests are routed to the hotel app:

.. code-block:: console

    $ curl $AMBASSADORURL/v1/hotels/bookings/7b4fc183-ee67-494d-9715-3510c6d8f2ef
    {
        "id": "7b4fc183-ee67-494d-9715-3510c6d8f2ef",
        "hotel_id": "046d471d-70c7-4595-80cc-266d3e6e07fa",
        "status": "confirmed"
    }

The car rental app is deployed similarly.

.. code-block:: yaml

    apiVersion: v1
    kind: Service
    metadata:
      name: car
      annotations:
        getambassador.io/config: |
          ---
          apiVersion: ambassador/v1
          kind: Mapping
          name: car_mapping
          prefix: /v1/cars
          rewrite: ""
          service: car
    spec:
      selector:
        app: car
      type: NodePort
      ports:
        - port: 80
          targetPort: 8000

.. code-block:: console

    $ docker build --tag=marselester/travel-car:v1.0.0 --file=docker/car.Dockerfile .
    $ kubectl apply -f k8s-ambassador/car/deployment.yml
    $ kubectl apply -f k8s-ambassador/car/service-gate.yml

You can see that the car booking is available at the gateway.

.. code-block:: console

    $ curl $AMBASSADORURL/v1/cars/bookings/9e0d65f5-9de2-4428-9bee-1f3967f05129
    {
        "id": "9e0d65f5-9de2-4428-9bee-1f3967f05129",
        "car_id": "cfb6f7a5-4591-4f5c-8b17-9a1b10f98ada",
        "status": "confirmed"
    }

Authentication Service
----------------------

API requests must be authenticated before reaching the Travel apps.
This work is delegated to `travelauth` Service that performs HTTP Basic authentication
and returns a username in `X-Travel-User` header if credentials matched.
We shall start from deploying it on the cluster.

.. code-block:: console

    $ docker build --tag=marselester/travel-auth:v1.0.0 --file=docker/auth.Dockerfile .
    $ kubectl apply -f k8s-ambassador/auth/deployment.yml
    $ kubectl apply -f k8s-ambassador/auth/service.yml

If the auth server is properly deployed, it will prompt for username/password.

.. code-block:: console

    $ kubectl get pods -l app=auth
    auth-api-556685f658-h9qb4
    $ kubectl port-forward auth-api-556685f658-h9qb4 8000
    $ curl -i -u bob:bob localhost:8000/v1/hotels
    HTTP/1.1 200 OK
    X-Travel-User: bob

Let's tell Ambassador to
`forward all requests to the auth server <https://www.getambassador.io/reference/services/auth-service/>`_
and copy `X-Travel-User` response header from the auth server to the routed request.

.. code-block:: yaml

    apiVersion: v1
    kind: Service
    metadata:
      name: travelauth
      annotations:
        getambassador.io/config: |
          ---
          apiVersion: ambassador/v1
          kind: AuthService
          name: gate_auth
          # This is k8s service name; all API requests are sent there. For example,
          # API request /v1/hotels will be sent to http://travelauth:80/v1/hotels.
          auth_service: travelauth
          proto: http
          # The travelauth service adds a username into a header after successful authentication,
          # so all the other services know who the user is (ratelimit, hotel, car services).
          allowed_authorization_headers:
            - "X-Travel-User"
    spec:
      selector:
        app: auth
      type: NodePort
      ports:
        - port: 80
          targetPort: 8000

.. code-block:: console

    $ kubectl apply -f k8s-ambassador/auth/service-auth.yml

With updated config Ambassador should enforce authentication on API gateway.

.. code-block:: console

    $ curl -i $AMBASSADORURL/v1/hotels/bookings/7b4fc183-ee67-494d-9715-3510c6d8f2ef
    HTTP/1.1 401 Unauthorized
    $ curl -i -u bob:bob $AMBASSADORURL/v1/hotels/bookings/7b4fc183-ee67-494d-9715-3510c6d8f2ef
    HTTP/1.1 200 OK
    {
        "id": "7b4fc183-ee67-494d-9715-3510c6d8f2ef",
        "hotel_id": "046d471d-70c7-4595-80cc-266d3e6e07fa",
        "status": "confirmed"
    }

Rate Limiting Service
---------------------

With Ambassador, individual requests can be annotated with metadata, called labels.
These labels can then be passed to a third party rate limiting service through a gRPC interface
which implements actual rate limit logic.
Check out `Node.js <https://www.getambassador.io/user-guide/rate-limiting-tutorial/>`_ and
`Java <https://blog.getambassador.io/designing-a-rate-limiting-service-for-ambassador-f460e9fabedb>`_ based examples.

In order to build such service, it must support Envoy's ratelimit.proto_ interface.
The protocol buffer definition of RateLimitService service is described in `./internal/pb/ratelimit.proto`.
You need to `install <https://grpc.io/docs/quickstart/go.html>`_ protoc compiler and protoc plugin for Go
(beware of `ProtoPackageIsVersion3 issue <https://github.com/golang/protobuf/issues/763>`_)
to generate gRPC service code.

.. code-block:: console

    $ brew install protobuf
    $ go get -u github.com/golang/protobuf/protoc-gen-go
    $ cd $GOPATH/src/github.com/golang/protobuf/protoc-gen-go
    $ git checkout v1.2.0
    $ go install

The arguments tell protoc to use `ratelimit.proto` definition,
search for imports in `./internal/pb/` dir, generate Go code using gprc plugin,
and place the result in `./internal/pb/` dir.

.. code-block:: console

    $ protoc ratelimit.proto -I internal/pb/ --go_out=plugins=grpc:internal/pb/

We now have a newly generated gRPC server and client code in `./internal/pb/ratelimit.pb.go`.
Our ratelimit server already implements `RateLimitServiceServer` interface.

.. code-block:: go

    // ShouldRateLimit must respond to the request with an OK or OVER_LIMIT code.
    func (s *server) ShouldRateLimit(ctx context.Context, req *pb.RateLimitRequest) (*pb.RateLimitResponse, error) {
        // Descriptors is a list of labels on which the rate limit service can base
        // its decision to accept or reject the request.
        log.Printf("%s: %+v", req.GetDomain(), req.GetDescriptors())

        resp := pb.RateLimitResponse{
            OverallCode: pb.RateLimitResponse_OVER_LIMIT,
        }
        return &resp, nil
    }

Let's deploy it on the cluster and send a few requests using
`grpcurl <https://github.com/fullstorydev/grpcurl>`_.

.. code-block:: console

    $ docker build --tag=marselester/travel-ratelimit:v1.0.0 --file=docker/ratelimit.Dockerfile .
    $ kubectl apply -f k8s-ambassador/ratelimit/deployment.yml
    $ kubectl apply -f k8s-ambassador/ratelimit/service.yml
    $ kubectl get pods -l app=ratelimit
    ratelimit-api-5554898589-4gmv5
    $ kubectl port-forward ratelimit-api-5554898589-4gmv5 5000

As we can see the server exposes RateLimitService.

.. code-block:: console

    $ grpcurl -plaintext localhost:5000 list
    grpc.reflection.v1alpha.ServerReflection
    pb.lyft.ratelimit.RateLimitService

Its ShouldRateLimit method should return `OK` or `OVER_LIMIT` reply.

.. code-block:: console

    $ grpcurl -d '{"domain":"envoy"}' -plaintext localhost:5000 pb.lyft.ratelimit.RateLimitService/ShouldRateLimit
    {
      "overallCode": "OVER_LIMIT"
    }

Now we shall introduce the ratelimit service to Ambassador.

.. code-block:: yaml

    apiVersion: v1
    kind: Service
    metadata:
      name: travelratelimit
      annotations:
        getambassador.io/config: |
          ---
          apiVersion: ambassador/v1
          kind: RateLimitService
          name: gate_ratelimit
          service: travelratelimit
    spec:
      selector:
        app: ratelimit
      type: NodePort
      ports:
        - port: 80
          targetPort: 5000

.. code-block:: console

    $ kubectl apply -f k8s-ambassador/ratelimit/service-rate.yml

Finally, I am going to add `labels` to attach rate limiting descriptors as shown in
`Rate Limiting tutorial <https://www.getambassador.io/user-guide/rate-limiting-tutorial/>`_.

.. code-block:: yaml

    apiVersion: v1
    kind: Service
    metadata:
      name: hotel
      annotations:
        getambassador.io/config: |
          ---
          apiVersion: ambassador/v1
          kind: Mapping
          name: hotel_mapping
          prefix: /v1/hotels
          rewrite: ""
          service: hotel
          labels:
            ambassador:
              - request_label_group:
                - x-ambassador-test-allow:
                    header: "x-ambassador-test-allow"
                    omit_if_not_present: true
    spec:
      selector:
        app: hotel
      type: NodePort
      ports:
        - port: 80
          targetPort: 8000

.. code-block:: console

    $ kubectl apply -f k8s-ambassador/hotel/service-rate.yml

If a request made to the hotel API has header `X-Ambassador-Test-Allow`,
it should be eligible for rate limiting.

.. code-block:: console

    $ curl -H "x-ambassador-test-allow: probably" -i -u bob:bob $AMBASSADORURL/v1/hotels
    HTTP/1.1 200 OK

Unfortunately I have not been able to make rate limiting work yet.
Here is the corresponding `GitHub issue <https://github.com/datawire/ambassador/issues/1144>`_.

.. _Minikube: http://kubernetes.io/docs/getting-started-guides/minikube/
.. _ratelimit.proto: https://github.com/datawire/ambassador/blob/master/ambassador/common/ratelimit/ratelimit.proto
