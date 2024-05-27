## Purpose

This envoyfilter can help you configure simple outlier detection in your apps accessed through Istio-gateway. Envoyfilter is aimed at Istio gateway.
## How to test it

**1. Install httpbin application, GW, VS**

Deploy httpbin app in the cluster. Expose the httpbin service using a istio gateway and a istio VS. App deployment yaml, Gateway yaml, VS yaml and secrets yaml is given in this directory. Make sure istio ingress gateway is of service type LoadBalancer. (Since I am using a kind cluster with MetalLB in my linux VM, I am not short of LB addresses).

**2. Install a sleep application to fake and co-exist with httpbin**

Sleep application using the same "app" labels is deployed along with httpbin application. Now "sleep" pod also become one of the valid endpoint for httpbin service. Sleep yaml is given in the directory.

**3. Connect to the app and check the logs of the sleep**

Make sure you are using one replica of httpbin and one replica of sleep.

From your host linux VM terminal where kind k8s cluster is running, connect to the httpbin  app using "curl" as given below.

    curl   -k -s https://httpbin.aegle.info/ip --connect-to httpbin.aegle.info:443:172.18.64.1:443

Response will mention the source IP of the request. In my case I see that output says it is ***"origin": "10.244.0.1"*** Check the logs of httpbin's and sleep's istio-proxy logs.

    gitmarut@gitmarut:~/go/src/envoyistio108/testcases/9-outlier$ k logs -n httpbin               sleep-9564bdbcd-vrn7q    -c istio-proxy --tail 3
    [2024-05-12T01:56:17.634Z] "GET /ip HTTP/1.1" 503 UF upstream_reset_before_response_started{remote_connection_failure,delayed_connect_error:_111} - "delayed_connect_error:_111" 0 152 0 - "10.244.0.1" "curl/7.81.0" "ca692bb6-c469-44ea-9577-62f8f76d53c5" "httpbin.aegle.info" "10.244.0.11:80" inbound|80|| - 10.244.0.11:80 10.244.0.1:0 outbound_.8000_._.httpbin.httpbin.svc.cluster.local default

    gitmarut@gitmarut:~/go/src/envoyistio108/testcases/9-outlier$ k logs -n httpbin              httpbin-5f7676ff7c-vm4nk    -c istio-proxy --tail 3
    [2024-05-14T00:56:48.417Z] "GET /ip HTTP/1.1" 200 - via_upstream - "-" 0 29 1 0 "10.244.0.1" "curl/7.81.0" "4a0dd62f-2da7-43e3-91b1-c8a99134ee62" "httpbin.aegle.info" "10.244.0.16:80" inbound|80|| 127.0.0.6:42589 10.244.0.16:80 10.244.0.1:0 outbound_.8000_._.httpbin.httpbin.svc.cluster.local default

All requests will succeed. But there will be 503 in sleep pod's logs. Because of the retries the request succeeds with sending it to httpbin again. Run the curl in a while loop like this.

    while true; do curl   -k -s https://httpbin.aegle.info/ip --connect-to httpbin.aegle.info:443:172.18.64.1:443; sleep 0.5; done;

If you keep checking "istioctl ep" output for ingressgateway pod, it will always show both endpoints for httpbin as HEALTHY OK always.

    gitmarut@gitmarut:~/go/src/envoyistio108/testcases/9-retries$ istioctl pc ep istio-ingressgateway-d4db74f5b-hqnsn.istio-system | grep httpbin
    10.244.0.20:80                                          HEALTHY     OK                outbound|8000||httpbin.httpbin.svc.cluster.local
    10.244.0.21:80                                          HEALTHY     OK                outbound|8000||httpbin.httpbin.svc.cluster.local


**4. Apply envoyfilter to the GW**

The envoy filter named "envoyfilter9.yaml" is configured to detect the outlier endpoint in an application and eject it from the service endpoints for 30s default value.  You can go check the API [documentation](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/cluster/v3/outlier_detection.proto) for outlier detection for more details.
Apply the envoy filter and check the traffic through while loop like earlier. Also check the "istioctl ep" for httpbin service. You will see IP address of the sleep pod in httpbin namespace is always shown as FAILED. This is due to outlier detection logic.

    istioctl pc ep istio-ingressgateway-d4db74f5b-hqnsn.istio-system | grep httpbin
    10.244.0.20:80                                          HEALTHY     OK                outbound|8000||httpbin.httpbin.svc.cluster.local
    10.244.0.21:80                                          HEALTHY     FAILED            outbound|8000||httpbin.httpbin.svc.cluster.local

If you remove the envoyfilter, both endpoints will come HEALTHY OK again.

