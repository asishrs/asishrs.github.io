# Proxyless gRPC load balancing in Kubernetes


<img class="cp t u fz ak" src="/images/posts/proxyless-grpc/main.png" width="3921" height="2185" role="presentation"/>

In this post, I will show how you can build a proxy-less load balancing for your gRPC services using the new xDS load balancing.

You can find the complete code for this experiment in [asishrs/proxyless-grpc-lb](https://github.com/asishrs/proxyless-grpc-lb)


Why is load balancing in gRPC difficult?
========================================

If you are building gRPC based applications, you may already be aware of the usage of HTTP2 in gRPC. If you are unfamiliar with that, please read below

*   [https://grpc.io/blog/grpc-load-balancing/](https://grpc.io/blog/grpc-load-balancing/)
*   [https://kubernetes.io/blog/2018/11/07/grpc-load-balancing-on-kubernetes-without-tears/](https://kubernetes.io/blog/2018/11/07/grpc-load-balancing-on-kubernetes-without-tears/)

At a high-level, I want you to understand two points.

1.  gRPC is built on HTTP/2, and HTTP/2 is designed to have a single long-lived TCP connection.
2.  To do gRPC load balancing, we need to shift from connection balancing to _request_ balancing.

What are the options?
=====================

There used to be two options to load balance gRPC requests in a Kubernetes cluster

*   Headless service
*   Using a Proxy (example Envoy, Istio, Linkerd)

Recently gRPC announced the support for [_xDS based load balancing_](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md), and as of this time, the gRPC team added support in _C-core, Java, and Go_ languages. This is an essential feature as this will open a third option for load balancing in gRPC, and I will show how to do that in a Kubernetes cluster. gRPC will be moving from its original _grpclb protocol_ to the new _xDS protocol._

xDS API
=======

[xDS API](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol) is a suite of APIs becoming popular and evolving into a standard used to configure various data plane software.

In the xDS API flow, the client uses the following main APIs:

*   **Listener Discovery Service (LDS)**: Returns [Listener](https://github.com/envoyproxy/envoy/blob/9e83625b16851cdc7e4b0a4483b0ce07c33ba76b/api/envoy/api/v2/listener.proto#L27) resources. Used basically as a convenient root for the gRPC client’s configuration. Points to the RouteConfiguration.
*   **Route Discovery Service (RDS)**: Returns [RouteConfiguration](https://github.com/envoyproxy/envoy/blob/9e83625b16851cdc7e4b0a4483b0ce07c33ba76b/api/envoy/api/v2/route.proto#L24) resources. Provides data used to populate the gRPC [service config](https://github.com/grpc/grpc/blob/master/doc/service_config.md). Points to the Cluster.
*   **Cluster Discovery Service (CDS)**: Returns [Cluster](https://github.com/envoyproxy/envoy/blob/9e83625b16851cdc7e4b0a4483b0ce07c33ba76b/api/envoy/api/v2/cluster.proto#L34) resources. Configures things like load balancing policy and load reporting. Points to the ClusterLoadAssignment.
*   **Endpoint Discovery Service (EDS)**: Returns [ClusterLoadAssignment](https://github.com/envoyproxy/envoy/blob/9e83625b16851cdc7e4b0a4483b0ce07c33ba76b/api/envoy/api/v2/endpoint.proto#L33) resources. Configures the set of endpoints (backend servers) to load balance across and may tell the client to drop requests.

gRPC load balancing using xDS API
=================================

To leverage xDS load balancing, the gRPC client needs to connect to the xDS server. Clients need to use xds resolver in the target URI used to create the gRPC channel.

The diagram below shows the sequence of API calls.

<img alt="Image for post" class="t u v cy ai" src="/images/posts/proxyless-grpc/xDS-api.png" width="1502" height="824" />

xDS API calls

The xDS server is responsible for discovering the EndPoints for the gRPC server and communicate that to the client. The client then asks for updates in a periodic interval.

xDS EndPoint Discovery in Kubernetes cluster
============================================

In this example, I am using k8s _client-go_ to discover the EndPoints and Envoy [go-control-plane](https://github.com/envoyproxy/go-control-plane) as the xDS server. In my implementation, the xDS server [polls the Kubernetes endpoints](https://github.com/asishrs/proxyless-grpc-lb/blob/master/xds-server/internal/app/resources.go#L51) every minute and updates the xDS snapshot with the gRPC server IP address and port. Since I am using a minute interval for polling, clients may take up to a minute to reflect the Endpoint updates.

<img alt="Image for post" class="t u v cy ai" src="/images/posts/proxyless-grpc/endpoint-discovery.png" width="1554" height="284"/>

**_Note:_** _Envoy go-control-plane doesn’t support multiple gRPC client requests. I am using a forked version_ [_with a fix for this_](https://github.com/asishrs/go-control-plane/blob/master/pkg/cache/v2/simple.go#L165)_. I have an_ [_issue open_](https://github.com/envoyproxy/go-control-plane/issues/349) _to discuss the same with the Envoy team._

Example Application
===================

I am using two gRPC services (hello and howdy) and clients connecting to those in a load-balanced way in this example application. The below diagram shows the setup.

<img alt="Image for post" class="t u v cy ai" src="/images/posts/proxyless-grpc/application.png" width="1676" height="746"/>

See all in Action
=================

To test the repository quickly, I am going to run all commands via [_Go Task_](https://github.com/go-task/task). You can look into the all commands in the _Taskfile.yml_ (or _Taskfile-\*.yml_) files.

**Deploy the components**
-------------------------

```shell
# XDS Server

# Build xDS Server 
task xds:build
# Deploy xDS Server 
task xds:deploy
# You can run above in a single command 
task xds:build xds:deploy


# gRPC Server

# Build gRPC Server
task server:build
# Deploy gRPC Server
task server:deploy
# You can run above in a single command
task server:build server:deploy

# gRPC Client

# Build gRPC Client
task client:build
# Deploy gRPC Client
task client:deploy
# You can run above in a single command
task client:build client:deploy
```

**Optional Step — Verify all components**

Check all the components are running.

Run `kubectl get deployments.apps` to check all deployments. You must see five deployments in the list.

```shell
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
hello-client   1/1     1            1           4d23h
howdy-client   1/1     1            1           4d23h
hello-server   2/2     2            2           4d23h
howdy-server   2/2     2            2           4d23h
xds-server     1/1     1            1           5d
```

Run `kubectl get svc` to check all services. You must see four services on the list.

```shell
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
kubernetes     ClusterIP   10.43.0.1       <none>        443/TCP     14d
xds-server     ClusterIP   10.43.113.60    <none>        18000/TCP   5d
hello-server   ClusterIP   10.43.78.196    <none>        50051/TCP   4d23h
howdy-server   ClusterIP   10.43.228.201   <none>        50052/TCP   4d23h
```

Run `kubectl get pods` to check all pods. You must see seven pods on the list. The pod suffix (after the last -) is going to be different each time.

```shell
NAME                            READY   STATUS    RESTARTS   AGE
howdy-client-659b8f99f5-q6fvx   1/1     Running   1          4d23h
hello-client-5f7bdc5f68-2vkfs   1/1     Running   1          4d23h
xds-server-86c5f6bfcf-f8wks     1/1     Running   1          5d
hello-server-b9ff98f5b-cgvk7    1/1     Running   1          4d23h
howdy-server-7d79c584f6-xbv4l   1/1     Running   1          4d23h
hello-server-b9ff98f5b-l7dmc    1/1     Running   1          4d23h
howdy-server-7d79c584f6-d54zs   1/1     Running   1          4d23h
```

Validate Proxyless (xDS) load balancing
=======================================

The server responds to the **gRPC** requests in the code by adding the server host IP. We can use this to identify the target server responding to the client request.

Let’s check the logs from _hello client_ by running `kubectl logs -f deployment/hello-client`

There are two essential things in the logs.

**xDS calls**

You can see logs initiating connections with the xDS server and getting the server’s xDS protocol response. This is the point where the client-side load balancer picks the available connections.

Example response

```shell
INFO: 2020/09/07 01:31:28 [xds] [eds-lb 0xc0002ce820] Watch update from xds-client 0xc0001502a0, content: {Drops:[] Localities:[{Endpoints:[{Address:10.42.0.234:50052 HealthStatus:1 Weight:0} {Address:10.42.1.138:50052 HealthStatus:1 Weight:0}] ID:my-region-my-zone- Priority:0 Weight:1000}]}
```

**gRPC Server Response**

You can see the **gRPC** server responding to the client requests. Using the IP address in the gRPC response, you can see that the response comes from both replicas of _hello-grpc-server_ apps.

Example response

```shell
{"level":"info","caller":"client/client.go:51","msg":"Hello Response","Response":"message:\"Hello gRPC Proxyless LB from host howdy-server-7d79c584f6-xbv4l\""}
{"level":"info","caller":"client/client.go:51","msg":"Hello Response","Response":"message:\"Hello gRPC Proxyless LB from host howdy-server-7d79c584f6-d54zs\""}
{"level":"info","caller":"client/client.go:51","msg":"Hello Response","Response":"message:\"Hello gRPC Proxyless LB from host howdy-server-7d79c584f6-xbv4l\""}
{"level":"info","caller":"client/client.go:51","msg":"Hello Response","Response":"message:\"Hello gRPC Proxyless LB from host howdy-server-7d79c584f6-d54zs\""}
```

Similarly, you can check the _howdy-client_ logs `kubectl logs -f deployment/howdy-client`

[Watch the commands!](https://asciinema.org/a/358829)

**Final Diagram**

Here is the diagram representing, the whole setup.

<img alt="Image for post" class="t u v cy ai" src="/images/posts/proxyless-grpc/final.png" width="2092" height="734"/>

Next Steps
==========

What about scaling the deployments? You can scale (up and down) the **gRPC server** (both _hello_ and _howdy_) deployments and see how the xDS load balancing will behave. It may take up to a minute for the client to pick up the changes.

```
\# Scale hello-server to 5 replicas
kubectl scale — replicas=5 deployments.apps/hello-server\# Scale howdy-server to 5 replicas
kubectl scale — replicas=5 deployments.apps/howdy-server
```

Bonus Diagram
=============

Here is the diagram with every call in this experiment.

<img alt="Image for post" class="t u v cy ai" src="/images/posts/proxyless-grpc/bonus.png" width="2200" height="1210"/>

References
==========

*   [https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds\_protocol](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol)
*   [https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md)
*   [https://github.com/envoyproxy/data-plane-api/tree/master/envoy/api/v2](https://github.com/envoyproxy/data-plane-api/tree/master/envoy/api/v2)
*   [https://github.com/grpc/grpc-go/blob/master/examples/features/xds/README.md](https://github.com/grpc/grpc-go/blob/master/examples/features/xds/README.md)
*   [https://github.com/envoyproxy/go-control-plane/issues/349](https://github.com/envoyproxy/go-control-plane/issues/349)
*   [https://medium.com/@salmaan.rashid/grpc-xds-loadbalancing-a05f8bd754b8](https://medium.com/@salmaan.rashid/grpc-xds-loadbalancing-a05f8bd754b8)