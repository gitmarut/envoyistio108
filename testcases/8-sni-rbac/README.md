## Purpose

This simple envoyfilter can help you configure some access policies(RBAC) for your gateway hostnames. Example given here controls the http access based on the source IP address of the request.

## How to test it

**1. Install httpbin application, GW, VS**

Deploy httpbin app in the cluster. Expose the httpbin service using a istio gateway and a istio VS. App deployment yaml, Gateway yaml, VS yaml and secrets yaml is given in this directory. Make sure istio ingress gateway is of service type LoadBalancer. (I am using a kind cluster with MetalLB in my linux VM).

**2. Connect to the app**
From your host linux VM terminal where kind k8s cluster is running, connect to the httpbin  app using "curl" as given below.

    curl   -k -s https://httpbin.aegle.info/ip --connect-to httpbin.aegle.info:443:172.18.64.1:443

Response will mention the source IP of the request. In my case I see that output says it is ***"origin": "10.244.0.1"***

**3. Apply envoyfilter to the GW**

The envoy filter named "envoyfilter8-gw.yaml" is configured to allow requests from "10.244.0.0". You can edit the envoy filter to allow your network's IP range /host bits as "address_prefix" and "prefix_len".

Apply the envoy filter. Again, do a curl as given above. You will see that curl is allowed.

Edit the envoyfilter to change the "address_prefix" to an non-existent range, say "10.245.0.0". And apply this envoy filter again.  Again, do a curl as given above. You will see that curl does not return anything from the app.

**4. How to debug this**

We can check either envoy stats or envoy logs to debug this.

*4.1 Envoy stats*

We can check envoy stats to see whether GW pod is disallowing the requests when there is non-existent IP ranged in "address_prefix"

See that envoy filter has a stat prefix "httpbin-rbac.". So we can port-forward port 15000 of istio ingress gateway pod to the host machine as given below.

    kubectl port-forward -n istio-system pod/istio-ingressgateway-d4db74f5b-6v6f7 15000:15000 &
And then sending curl request to stats endpoint of envoy, check stats for httpbin as given below.

    gitmarut@gitmarut:~/go/src/envoyistio108/testcases/8-sni-rbac$ curl -X POST -s http://127.0.0.1:15000/stats | grep httpbin
    Handling connection for 15000
    httpbin-rbac.rbac.allowed: 2 <<<< curl succeeded
    httpbin-rbac.rbac.denied: 5 <<<< curl failed
    httpbin-rbac.rbac.shadow_allowed: 0
    httpbin-rbac.rbac.shadow_denied: 0

*4.2 Envoy logs*

You can port-forward port 15000 of istio ingress gateway pod to the host machine as given below.

    kubectl port-forward -n istio-system pod/istio-ingressgateway-d4db74f5b-6v6f7 15000:15000 &

Enable the following logs using envoy logging endpoint.

    curl -X POST -s http://127.0.0.1:15000/logging?paths=connection:debug,http:debug,conn_handler:debug,main:debug,pool:debug,rbac:debug

Now do the curl to httpbin with a date prefixed.

    date;curl -v  -k -s https://httpbin.aegle.info/ip --connect-to httpbin.aegle.info:443:172.18.64.1:443

Take the logs of ingress gateway pod and on a happy path passing case you will see a rbac enforced allowed with the policy name mentioned in log.

    2024-04-24T01:32:42.815579Z     debug   envoy rbac external/envoy/source/extensions/filters/network/rbac/rbac_filter.cc:156     enforced allowed, matched policy httpbin-traffic-policy thread=30

On failing case you will see 

    2024-04-24T01:29:21.481404Z     debug   envoy rbac external/envoy/source/extensions/filters/network/rbac/rbac_filter.cc:168     enforced denied, matched policy none    thread=30

*4.3 Full logs for reference**
Passing 

    gitmarut@gitmarut:~/go/src/envoyistio108/testcases/8-sni-rbac$ grep 01:32:42 logs123
    
    2024-04-24T01:32:42.812279Z     debug   envoy conn_handler external/envoy/source/extensions/listener_managers/listener_manager/active_tcp_listener.cc:159       [Tags: "ConnectionId":"840"] new connection from 10.244.0.1:3032       thread=30
    2024-04-24T01:32:42.815526Z     debug   envoy rbac external/envoy/source/extensions/filters/network/rbac/rbac_filter.cc:90      checking connection: requestedServerName: httpbin.aegle.info, sourceIP: 10.244.0.1:3032, directRemoteIP: 10.244.0.1:3032,remoteIP: 10.244.0.1:3032, localAddress: 10.244.0.10:8443, ssl: uriSanPeerCertificate: , dnsSanPeerCertificate: , subjectPeerCertificate: , dynamicMetadata:   thread=30
    2024-04-24T01:32:42.815579Z     debug   envoy rbac external/envoy/source/extensions/filters/network/rbac/rbac_filter.cc:156     enforced allowed, matched policy httpbin-traffic-policy thread=30
    2024-04-24T01:32:42.815622Z     debug   envoy http external/envoy/source/common/http/conn_manager_impl.cc:391   [Tags: "ConnectionId":"840"] new stream thread=30
    2024-04-24T01:32:42.815662Z     debug   envoy http external/envoy/source/common/http/conn_manager_impl.cc:1194  [Tags: "ConnectionId":"840","StreamId":"7835256933037303912"] request headers complete (end_stream=true):
    2024-04-24T01:32:42.815671Z     debug   envoy http external/envoy/source/common/http/conn_manager_impl.cc:1177  [Tags: "ConnectionId":"840","StreamId":"7835256933037303912"] request end stream       thread=30
    2024-04-24T01:32:42.815691Z     debug   envoy connection external/envoy/source/common/network/connection_impl.h:98      [Tags: "ConnectionId":"840"] current connecting state: false  thread=30
    2024-04-24T01:32:42.818319Z     debug   envoy http external/envoy/source/common/http/conn_manager_impl.cc:1863  [Tags: "ConnectionId":"840","StreamId":"7835256933037303912"] encoding headers via codec (end_stream=false):
    2024-04-24T01:32:42.818419Z     debug   envoy http external/envoy/source/common/http/conn_manager_impl.cc:1968  [Tags: "ConnectionId":"840","StreamId":"7835256933037303912"] Codec completed encoding stream. thread=30
    2024-04-24T01:32:42.818963Z     debug   envoy connection external/envoy/source/common/network/connection_impl.cc:714    [Tags: "ConnectionId":"840"] remote close       thread=30
    2024-04-24T01:32:42.819007Z     debug   envoy connection external/envoy/source/common/network/connection_impl.cc:278    [Tags: "ConnectionId":"840"] closing socket: 0  thread=30
    2024-04-24T01:32:42.819100Z     debug   envoy connection external/envoy/source/extensions/transport_sockets/tls/ssl_socket.cc:329       [Tags: "ConnectionId":"840"] SSL shutdown: rc=1thread=30
    2024-04-24T01:32:42.819138Z     debug   envoy conn_handler external/envoy/source/extensions/listener_managers/listener_manager/active_stream_listener_base.cc:135       [Tags: "ConnectionId":"840"] adding to cleanup list    thread=30
    gitmarut@gitmarut:~/go/src/envoyistio108/testcases/8-sni-rbac$
    gitmarut@gitmarut:~/go/src/envoyistio108/testcases/8-sni-rbac$

Failing

    gitmarut@gitmarut:~/go/src/envoyistio108/testcases/8-sni-rbac$
    gitmarut@gitmarut:~/go/src/envoyistio108/testcases/8-sni-rbac$ grep 01:29:21 logs123
    2024-04-24T01:29:21.002493Z     debug   envoy conn_handler external/envoy/source/extensions/listener_managers/listener_manager/active_tcp_listener.cc:159       [Tags: "ConnectionId":"733"] new connection from 10.244.0.1:44160      thread=31
    2024-04-24T01:29:21.002816Z     debug   envoy http external/envoy/source/common/http/conn_manager_impl.cc:391   [Tags: "ConnectionId":"733"] new stream thread=31
    2024-04-24T01:29:21.003012Z     debug   envoy http external/envoy/source/common/http/conn_manager_impl.cc:1194  [Tags: "ConnectionId":"733","StreamId":"8870473208851658696"] request headers complete (end_stream=true):
    2024-04-24T01:29:21.003040Z     debug   envoy http external/envoy/source/common/http/conn_manager_impl.cc:1177  [Tags: "ConnectionId":"733","StreamId":"8870473208851658696"] request end stream       thread=31
    2024-04-24T01:29:21.003070Z     debug   envoy connection external/envoy/source/common/network/connection_impl.h:98      [Tags: "ConnectionId":"733"] current connecting state: false  thread=31
    2024-04-24T01:29:21.003152Z     debug   envoy pool external/envoy/source/common/conn_pool/conn_pool_base.cc:265 [Tags: "ConnectionId":"8"] using existing fully connected connection  thread=31
    2024-04-24T01:29:21.003226Z     debug   envoy pool external/envoy/source/common/conn_pool/conn_pool_base.cc:182 [Tags: "ConnectionId":"8"] creating stream      thread=31
    2024-04-24T01:29:21.004020Z     debug   envoy http external/envoy/source/common/http/conn_manager_impl.cc:1794  [Tags: "ConnectionId":"733","StreamId":"8870473208851658696"] closing connection due to connection close header        thread=31
    2024-04-24T01:29:21.004105Z     debug   envoy http external/envoy/source/common/http/conn_manager_impl.cc:1863  [Tags: "ConnectionId":"733","StreamId":"8870473208851658696"] encoding headers via codec (end_stream=true):
    'date', 'Wed, 24 Apr 2024 01:29:21 GMT'
    2024-04-24T01:29:21.004137Z     debug   envoy http external/envoy/source/common/http/conn_manager_impl.cc:1968  [Tags: "ConnectionId":"733","StreamId":"8870473208851658696"] Codec completed encoding stream. thread=31
    2024-04-24T01:29:21.004164Z     debug   envoy connection external/envoy/source/common/network/connection_impl.cc:146    [Tags: "ConnectionId":"733"] closing data_to_write=143 type=0 thread=31
    2024-04-24T01:29:21.004176Z     debug   envoy connection external/envoy/source/common/network/connection_impl_base.cc:47        [Tags: "ConnectionId":"733"] setting delayed close timer with timeout 1000 ms  thread=31
    2024-04-24T01:29:21.004201Z     debug   envoy pool external/envoy/source/common/http/http1/conn_pool.cc:53      [Tags: "ConnectionId":"8"] response complete    thread=31
    2024-04-24T01:29:21.004228Z     debug   envoy pool external/envoy/source/common/conn_pool/conn_pool_base.cc:215 [Tags: "ConnectionId":"8"] destroying stream: 0 remaining       thread=31
    2024-04-24T01:29:21.004435Z     debug   envoy connection external/envoy/source/common/network/connection_impl.cc:788    [Tags: "ConnectionId":"733"] write flush complete       thread=31
    2024-04-24T01:29:21.004454Z     debug   envoy connection external/envoy/source/common/network/connection_impl.cc:278    [Tags: "ConnectionId":"733"] closing socket: 1  thread=31
    2024-04-24T01:29:21.004507Z     debug   envoy conn_handler external/envoy/source/extensions/listener_managers/listener_manager/active_stream_listener_base.cc:135       [Tags: "ConnectionId":"733"] adding to cleanup list    thread=31
    2024-04-24T01:29:21.478774Z     debug   envoy conn_handler external/envoy/source/extensions/listener_managers/listener_manager/active_tcp_listener.cc:159       [Tags: "ConnectionId":"734"] new connection from 10.244.0.1:17569      thread=30
    2024-04-24T01:29:21.481350Z     debug   envoy rbac external/envoy/source/extensions/filters/network/rbac/rbac_filter.cc:90      checking connection: requestedServerName: httpbin.aegle.info, sourceIP: 10.244.0.1:17569, directRemoteIP: 10.244.0.1:17569,remoteIP: 10.244.0.1:17569, localAddress: 10.244.0.10:8443, ssl: uriSanPeerCertificate: , dnsSanPeerCertificate: , subjectPeerCertificate: , dynamicMetadata:      thread=30
    2024-04-24T01:29:21.481404Z     debug   envoy rbac external/envoy/source/extensions/filters/network/rbac/rbac_filter.cc:168     enforced denied, matched policy none    thread=30
    2024-04-24T01:29:21.481415Z     debug   envoy connection external/envoy/source/common/network/connection_impl.cc:146    [Tags: "ConnectionId":"734"] closing data_to_write=0 type=1   thread=30
    2024-04-24T01:29:21.481419Z     debug   envoy connection external/envoy/source/common/network/connection_impl.cc:278    [Tags: "ConnectionId":"734"] closing socket: 1  thread=30
    2024-04-24T01:29:21.481452Z     debug   envoy connection external/envoy/source/extensions/transport_sockets/tls/ssl_socket.cc:329       [Tags: "ConnectionId":"734"] SSL shutdown: rc=0thread=30
    2024-04-24T01:29:21.481488Z     debug   envoy conn_handler external/envoy/source/extensions/listener_managers/listener_manager/active_stream_listener_base.cc:135       [Tags: "ConnectionId":"734"] adding to cleanup list    thread=30
    2024-04-24T01:29:21.552072Z     debug   envoy main external/envoy/source/server/server.cc:263   flushing stats  thread=22
    gitmarut@gitmarut:~/go/src/envoyistio108/testcases/8-sni-rbac$

