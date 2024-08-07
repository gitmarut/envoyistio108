## Purpose

This simple envoyfilter can help you in customizing response header "server". 

By default istio ingress gateway or sidecar will overwrite "server" header in response to "istio-envoy" without any envoyfilter applied.

## How to test it

**1, Install httpbin app, GW, VS and a client app**

Deploy httpbin application (yaml  [here](https://github.com/istio/istio/blob/master/samples/httpbin/httpbin.yaml)) to httpbin namespace. Istio injection is enabled in httpbin namespace.

Expose the httpbin service using a istio gateway and a istio VS. Gateway yaml and VS yaml is given in this directory. Make sure istio ingress gateway is of service type LoadBalancer. (I am using a kind cluster with MetalLB in my linux VM).
 
```
NOTE:
I have installed istio with istioctl and default profile
```
Without applying any envoyfilter in the system let's check how does it handle this response header.

Deploy sleep app in sleep namespace from this yaml [here](https://github.com/istio/istio/blob/master/samples/sleep/sleep.yaml).

Get into the shell of sleep pod and run the curl command from sleep pod's shell.

    kubectl exec -it -n sleep "$(kubectl get pod -l app=sleep -n sleep -o jsonpath={.items..metadata.name})" -- sh
   
    curl -v -s httpbin.httpbin.svc.cluster.local:8000/ip

This will show the response header - "server: istio-envoy" which is the default behavior for a sidecar.

Also do a curl from the host linux VM. This will pass through istio ingress-gateway pod.

    curl -v -k  https://httpbin.example.com/headers --connect-to httpbin.example.com:443:172.18.64.1:443

This will also show the response header "server: istio-envoy", which is the default behavior of the istio ingress gateway pod.

**2. Apply envoyfilters and check response header change**

2.1 Apply the envoyfilter named "envoyfilter6-3.yaml" And repeat the curl from sleep pod given above.
This will show the header as "server: gunicorn/19.9.0" a Python webserver framework which is the original header in httpbin application. This is because of the field "server_header_transformation: PASS_THROUGH" in "envoyfilter6-3.yaml".

2.2 Now apply the envoyfilter named "envoyfilter6-2.yaml" And repeat the curl from sleep pod given above.
This will show the header as "server: httpbin" which is the header mentioned in "envoyfilter6-2.yaml". This is because of the field "server_header_transformation: OVERWRITE" in "envoyfilter6-2.yaml".

2.3 Repeat the curl from the host linux VM given above. This will show the header as “server: istio-envoy” which is the default behavior of the istio ingress gateway pod.

2.4 Now apply the envoyfilter named "envoyfilter6-1.yaml". Repeat the curl from the host linux VM given above. This will show the header as "server: httpbin-gw" which is the header mentioned in "envoyfilter6-1.yaml". This is because of the field "server_header_transformation: OVERWRITE" in "envoyfilter6-1.yaml".



