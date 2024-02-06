## Purpose

This envoyfilter limits connection to a particular port in your istio workload. Will be useful if workload pods are not able to handle more than a certain number of connections for whatever reasons - may be a code limitation or the CPU/MEM resources given to it is just enough for a certain number of clients.

## Explanation
Example envoyfilter allows only one connection to bookinfo productpage app port 9080.

## How to test it

Apply the envoyfilter given in this directory to your bookinfo namespace. Expose the bookinfo productpage as  a loadbalancer. (Since I am using a kind cluster with MetalLB in my linux VM, I am not short of LB addresses. You can also do with your node port as well)

Now you can run a "while loop" like this to see there are no errors.

    while true; do curl  -I  -s http://172.18.64.2:9080  ; done;
Output will be 200.

    gitmarut@gitmarut:~/go/src/envoyistio108/testcases/limit_max_connections_on_a_port$ while true; do curl  -I  -s http://172.18.64.2:9080  ; done;
    HTTP/1.1 200 OK
    server: istio-envoy
    date: Tue, 06 Feb 2024 01:16:57 GMT
    content-type: text/html; charset=utf-8
    content-length: 1683
    x-envoy-upstream-service-time: 1
    x-envoy-decorator-operation: productpage.bookinfo1.svc.cluster.local:9080/*

 Now we will add a persistent connection to productpage by tricking curl. Curl by default closes the connection when it exits each time in the above loop. But we will trick it with adding nc.
From a second terminal, execute

    nc -l -p 8080 &

Now curl two URLs from second terminal. Since second URL won't respond curl cannot exit, and also occupy the only allowed connection in productpage.

    gitmarut@gitmarut:~$ curl -v -I  -s http://172.18.64.2:9080   localhost:8080
    *   Trying 172.18.64.2:9080...
    * Connected to 172.18.64.2 (172.18.64.2) port 9080 (#0)
    > HEAD / HTTP/1.1
    > Host: 172.18.64.2:9080
    > User-Agent: curl/7.81.0
    > Accept: */*
    >
    * Mark bundle as not supporting multiuse
    < HTTP/1.1 200 OK
    HTTP/1.1 200 OK
    < server: istio-envoy
    server: istio-envoy
    < date: Tue, 06 Feb 2024 01:23:01 GMT
    date: Tue, 06 Feb 2024 01:23:01 GMT
    < content-type: text/html; charset=utf-8
    content-type: text/html; charset=utf-8
    < content-length: 1683
    content-length: 1683
    < x-envoy-upstream-service-time: 2
    x-envoy-upstream-service-time: 2
    < x-envoy-decorator-operation: productpage.bookinfo1.svc.cluster.local:9080/*
    x-envoy-decorator-operation: productpage.bookinfo1.svc.cluster.local:9080/*
    
    <
    * Connection #0 to host 172.18.64.2 left intact
    *   Trying 127.0.0.1:8080...
    *   Trying ::1:8080...
    * connect to ::1 port 8080 failed: Connection refused

Now run the above while loop with a "-v"

    while true; do curl  -v -I  -s http://172.18.64.2:9080  ; done;
Output will be 

    gitmarut@gitmarut:~/go/src/envoyistio108/testcases/limit_max_connections_on_a_port$ while true; do curl -v -I  -s http://172.18.64.2:9080  ; done;
    *   Trying 172.18.64.2:9080...
    * Connected to 172.18.64.2 (172.18.64.2) port 9080 (#0)
    > HEAD / HTTP/1.1
    > Host: 172.18.64.2:9080
    > User-Agent: curl/7.81.0
    > Accept: */*
    >
    * Recv failure: Connection reset by peer
    * Closing connection 0

Now if you kill the tricky curl in second terminal by pressing "ctrl + c", this while loop above will start getting http code 200.

### Checking envoy stats in productpage app.
See stats of istio sidecar in the productpage  app, edit deployment for workload with istio injection enabled. 
Spec > template > metadata > annotations 

    template:
      metadata:
        annotations:
          prometheus.io/path: /metrics
          prometheus.io/port: "9080"
          prometheus.io/scrape: "true"
          proxy.istio.io/config: |
            proxyStatsMatcher:
             inclusionRegexps:
              - ".*downstream_.*"
              - ".*upstream_.*"

productpage pod will restart and you can port-forward 15000 port of the pod to local host.

    kubectl  port-forward -n bookinfo1            pod/productpage-v1-5f87f5b678-8dcwt 15000:15000 &

Now curl the localhost:15000 and
1. check "http.inbound_0.0.0.0_9080.downstream_cx_active" never crosses 1,
2. check "http.inbound_0.0.0.0_9080.downstream_cx_destroy" continously increases.
3. check "http.inbound_0.0.0.0_9080.downstream_cx_destroy_local" also continously increases.

    curl -s 127.0.0.1:15000/stats | grep cx  | grep 9080 | grep inbound | grep 'active\|destroy'

Have fun.
