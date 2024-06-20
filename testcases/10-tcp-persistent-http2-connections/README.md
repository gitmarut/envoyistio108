## Purpose

This envoyfilter can help you configure http2 CONNECT tunneling for TCP workloads. More details [here](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/upgrades).

By default, the 'tcp_proxy' filter manages TCP connections between two Istio Sidecars.  
This implies that two Istio Sidecars will establish a new connection for each new connection made by the client app.

All connections made by the client application will be proxied through a few number of persistent connections between two Istio Sidecars when `HTTP/2 CONNECT} is used between the two Sidecars.  
  
The value of the {concurrency} setting in the client-side Istio Sidecar determines the number of persistent connections that can exist between two Istio Sidecars. Default value is 2.
## How to test it

**1. Install tcp-random-app and sleep**

Deploy tcp-random-app. And sleep app in the sleep namepsace.  Yaml is given. We will check tcp connections between these and see they are tunneling on http2 as depicted below.
```text
     >> (TCP)     (HTTP/2 CONNECT)                          >> (TCP)
Sleep ---> Sidecar -------------> Sidecar ---> tcp-random-app
```
**2. Change default envoy access logs format**

    istioctl install -f  istio-operator-access-log-format.yaml

An additional `[%CONNECTION_ID%]` filed is added to envoy access logs.

**3. Default behavior**
Get the ClusterIP of 'tcp-random-app' service.

    kubectl get svc -n tcp-random-app -o=jsonpath='{.items..spec.clusterIP}'; echo
    
Get 'sleep' pod name and get into the shell of it.

    sleeppod=$(kubectl get pods -n sleep -l app=sleep -o=jsonpath='{.items..metadata.name}' )
    
    kubectl exec -it -n sleep $sleeppod -- sh
Run the following command from sleep pod's shell

    nc 10.96.217.49 17777

On each press of return key(enter) at 'sleep' pod, you get a random number. It will ignore any text entered except "STOP". Server will gracefully terminate connection of entering "STOP" On pressing ctrl+C at client, it will error EOF and close connection to client.

At this you will get access logs in 'istio-proxy' container of the 'tcp-random-app' pod. Do the connect(nc) and STOP as given above multiple times.


    '[2024-06-20T00:55:29.159Z] [21] "- - -" 0 - - - "-" 664 1013 65881 - "-" "-" "-" "-" "10.244.0.45:17777" inbound|17777|| 127.0.0.6:48403 10.244.0.45:17777 10.244.0.44:40706 outbound_.17777_._.tcp-random-app.tcp-random-app.svc.cluster.local -'
    '[2024-06-20T00:56:48.597Z] [29] "- - -" 0 - - - "-" 661 937 24715 - "-" "-" "-" "-" "10.244.0.45:17777" inbound|17777|| 127.0.0.6:35963 10.244.0.45:17777 10.244.0.44:59970 outbound_.17777_._.tcp-random-app.tcp-random-app.svc.cluster.local -'
    '[2024-06-20T01:02:23.140Z] [53] "- - -" 0 - - - "-" 658 863 41755 - "-" "-" "-" "-" "10.244.0.45:17777" inbound|17777|| 127.0.0.6:37437 10.244.0.45:17777 10.244.0.44:45322 outbound_.17777_._.tcp-random-app.tcp-random-app.svc.cluster.local -'
    '[2024-06-20T01:03:12.681Z] [58] "- - -" 0 - - - "-" 660 901 6185 - "-" "-" "-" "-" "10.244.0.45:17777" inbound|17777|| 127.0.0.6:60755 10.244.0.45:17777 10.244.0.44:48756 outbound_.17777_._.tcp-random-app.tcp-random-app.svc.cluster.local -'
    '[2024-06-20T00:52:54.286Z] [9] "- - -" 0 - - - "-" 660 1167 912095 - "-" "-" "-" "-" "10.244.0.45:17777" inbound|17777|| 127.0.0.6:38681 10.244.0.45:17777 10.244.0.44:33434 outbound_.17777_._.tcp-random-app.tcp-random-app.svc.cluster.local -'
    '[2024-06-20T01:09:10.632Z] [84] "- - -" 0 - - - "-" 673 1390 29931 - "-" "-" "-" "-" "10.244.0.45:17777" inbound|17777|| 127.0.0.6:38873 10.244.0.45:17777 10.244.0.44:44512 outbound_.17777_._.tcp-random-app.tcp-random-app.svc.cluster.local -'

You can see the `%CONNECTION_ID%` in 2nd column. It keeps changing on each connect(nc). 

**5. Configuring HTTP/2 Connect**

There are yaml files given to do this, namely "new-svc-object.yaml", "vs-dr.yaml" and "envoyfilter10.yaml". 
In the new svc object we are defining an alternate port for the same service with the http2 as the protocol. Name of the port has to start with http2. 

Using the DR and VS we are redirecting the traffic on port 17777 to 17778.
And the envoyfilter will configure the server sidecar to accept HTTP2/CONNECT on port 17778

Apply the above objects.

    kubectl apply -f new-svc-object.yaml
    kubectl apply -f vs-dr.yaml
    kubectl apply -f envoyfilter10.yaml

**6. Test HTTP/2 CONNECT behaviour of TCP proxying**

From the 'sleep' pod's shell connect to the 'tcp-random-app'.

    nc 10.96.217.49 17777
Do the connect(nc) and STOP as given above multiple times.


    '[2024-06-20T01:29:57.087Z] [170] "CONNECT - HTTP/2" 200 - via_upstream - "-" 5 0 671 - "-" "-" "8da585d4-9fd5-4dfc-9f1e-74d9f8f37db8" "tcp-random-app.tcp-random-app.svc.cluster.local:17778" "127.0.0.1:17777" inbound_17777 127.0.0.1:47016 10.244.0.45:17778 10.244.0.44:48722 outbound_.17778_._.tcp-random-app.tcp-random-app.svc.cluster.local default'
    '[2024-06-20T01:30:06.687Z] [170] "CONNECT - HTTP/2" 200 - via_upstream - "-" 5 0 464 - "-" "-" "d0f70955-5789-442c-8717-ac6d96fb0a18" "tcp-random-app.tcp-random-app.svc.cluster.local:17778" "127.0.0.1:17777" inbound_17777 127.0.0.1:58136 10.244.0.45:17778 10.244.0.44:48722 outbound_.17778_._.tcp-random-app.tcp-random-app.svc.cluster.local default'
    '[2024-06-20T01:30:17.658Z] [175] "CONNECT - HTTP/2" 200 - via_upstream - "-" 5 0 4 - "-" "-" "7018e52d-07c1-41e7-8b2a-5c51bda9d886" "tcp-random-app.tcp-random-app.svc.cluster.local:17778" "127.0.0.1:17777" inbound_17777 127.0.0.1:39276 10.244.0.45:17778 10.244.0.44:40276 outbound_.17778_._.tcp-random-app.tcp-random-app.svc.cluster.local default'
    '[2024-06-20T01:30:25.144Z] [175] "CONNECT - HTTP/2" 200 - via_upstream - "-" 5 0 1847 - "-" "-" "3d8dff68-b6ac-4ed3-b8c6-5e434275f8a9" "tcp-random-app.tcp-random-app.svc.cluster.local:17778" "127.0.0.1:17777" inbound_17777 127.0.0.1:45994 10.244.0.45:17778 10.244.0.44:40276 outbound_.17778_._.tcp-random-app.tcp-random-app.svc.cluster.local default'
    '[2024-06-20T01:30:36.689Z] [175] "CONNECT - HTTP/2" 200 - via_upstream - "-" 5 0 1774 - "-" "-" "eddda32d-fd9f-4e3c-9de3-7ad652bc6ed1" "tcp-random-app.tcp-random-app.svc.cluster.local:17778" "127.0.0.1:17777" inbound_17777 127.0.0.1:40888 10.244.0.45:17778 10.244.0.44:40276 outbound_.17778_._.tcp-random-app.tcp-random-app.svc.cluster.local default'
    '[2024-06-20T01:30:43.626Z] [175] "CONNECT - HTTP/2" 200 - via_upstream - "-" 5 0 2821 - "-" "-" "7a4b6f8d-7a2a-453b-adc8-73e7cb9fe558" "tcp-random-app.tcp-random-app.svc.cluster.local:17778" "127.0.0.1:17777" inbound_17777 127.0.0.1:49534 10.244.0.45:17778 10.244.0.44:40276 outbound_.17778_._.tcp-random-app.tcp-random-app.svc.cluster.local default'
    '[2024-06-20T01:30:56.679Z] [170] "CONNECT - HTTP/2" 200 - via_upstream - "-" 7 0 1 - "-" "-" "cdab6b28-2e8f-4e29-add0-130ea6005eee" "tcp-random-app.tcp-random-app.svc.cluster.local:17778" "127.0.0.1:17777" inbound_17777 127.0.0.1:46922 10.244.0.45:17778 10.244.0.44:48722 outbound_.17778_._.tcp-random-app.tcp-random-app.svc.cluster.local default'
    '[2024-06-20T01:30:56.907Z] [170] "CONNECT - HTTP/2" 200 - via_upstream - "-" 8 114 2741 - "-" "-" "8e6d6a7c-9592-45c0-a77d-ec00c521a8fb" "tcp-random-app.tcp-random-app.svc.cluster.local:17778" "127.0.0.1:17777" inbound_17777 127.0.0.1:46924 10.244.0.45:17778 10.244.0.44:48722 outbound_.17778_._.tcp-random-app.tcp-random-app.svc.cluster.local default'
    '[2024-06-20T01:31:08.483Z] [175] "CONNECT - HTTP/2" 200 - via_upstream - "-" 5 0 3701 - "-" "-" "5c34fdf0-900d-4a77-8520-d1a35681b3d5" "tcp-random-app.tcp-random-app.svc.cluster.local:17778" "127.0.0.1:17777" inbound_17777 127.0.0.1:44132 10.244.0.45:17778 10.244.0.44:40276 outbound_.17778_._.tcp-random-app.tcp-random-app.svc.cluster.local default'
    '[2024-06-20T01:32:01.740Z] [175] "CONNECT - HTTP/2" 200 - via_upstream - "-" 16 417 4677 - "-" "-" "c988ff0e-3a24-4649-8904-be18a9964d1e" "tcp-random-app.tcp-random-app.svc.cluster.local:17778" "127.0.0.1:17777" inbound_17777 127.0.0.1:38980 10.244.0.45:17778 10.244.0.44:40276 outbound_.17778_._.tcp-random-app.tcp-random-app.svc.cluster.local default'
    '[2024-06-20T01:32:09.968Z] [175] "CONNECT - HTTP/2" 200 - via_upstream - "-" 15 375 2878 - "-" "-" "7dac2dfc-38a8-4cd1-babd-ae2eac76a93c" "tcp-random-app.tcp-random-app.svc.cluster.local:17778" "127.0.0.1:17777" inbound_17777 127.0.0.1:57426 10.244.0.45:17778 10.244.0.44:40276 outbound_.17778_._.tcp-random-app.tcp-random-app.svc.cluster.local default'
    '[2024-06-20T01:32:21.225Z] [170] "CONNECT - HTTP/2" 200 - via_upstream - "-" 10 189 2471 - "-" "-" "77c128f9-c16f-418c-a988-a7fa21563bc1" "tcp-random-app.tcp-random-app.svc.cluster.local:17778" "127.0.0.1:17777" inbound_17777 127.0.0.1:54186 10.244.0.45:17778 10.244.0.44:48722 outbound_.17778_._.tcp-random-app.tcp-random-app.svc.cluster.local default'
    '[2024-06-20T01:32:27.371Z] [175] "CONNECT - HTTP/2" 200 - via_upstream - "-" 5 0 1692 - "-" "-" "9758d76e-2081-4bd2-b4d0-1fb763d1b78d" "tcp-random-app.tcp-random-app.svc.cluster.local:17778" "127.0.0.1:17777" inbound_17777 127.0.0.1:54190 10.244.0.45:17778 10.244.0.44:40276 outbound_.17778_._.tcp-random-app.tcp-random-app.svc.cluster.local default'
    '[2024-06-20T01:32:35.380Z] [170] "CONNECT - HTTP/2" 200 - via_upstream - "-" 5 0 1835 - "-" "-" "b5d2fcda-0fc9-41b6-81bf-591d226b21af" "tcp-random-app.tcp-random-app.svc.cluster.local:17778" "127.0.0.1:17777" inbound_17777 127.0.0.1:44938 10.244.0.45:17778 10.244.0.44:48722 outbound_.17778_._.tcp-random-app.tcp-random-app.svc.cluster.local default'
    '[2024-06-20T01:32:45.544Z] [175] "CONNECT - HTTP/2" 200 - via_upstream - "-" 5 0 2048 - "-" "-" "877a314b-7a64-4a8a-8045-4ba46e7388d9" "tcp-random-app.tcp-random-app.svc.cluster.local:17778" "127.0.0.1:17777" inbound_17777 127.0.0.1:54896 10.244.0.45:17778 10.244.0.44:40276 outbound_.17778_._.tcp-random-app.tcp-random-app.svc.cluster.local defau

See that `%CONNECTION_ID%` has only 2 values. [170] and [175]

This indicates that one of the two persistent connections formed between two Istio Sidecars serves as a proxy for each new connection made by the client app ({sleep}).
