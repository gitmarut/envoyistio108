## Purpose

This simple envoyfilter can help you in handling abnormally long headers. Couple of scenarios can happen.
1. Application needs a long valid header like an auth token.
2. Application misbehaves on an abnormally long header.

By default envoy accepts header information upto 60kB. In the first case above we might have to increase this. And in the second case we might have to decrease this. 

## How to test it

  **1, Install app, GW, Secret and VS.**
   
  Install [bookinfo](https://github.com/istio/istio/blob/master/samples/bookinfo/platform/kube/bookinfo.yaml) in a namespace called bookinfo0 with istio-injection enabled.
 
Expose the bookinfo service using a istio gateway and a istio VS. Gateway yaml and VS yaml is given in this directory. Make sure istio ingress gateway is of service type LoadBalancer. (I am using a kind cluster with MetalLB in my linux VM).

Also I have exposed the productpage service as Load Balancer to test it out at directly for second step.

```
NOTE:
I have installed istio with istioctl and default profile
```

**2. Send traffic directly to productpage**

Productpage app supports headers upto the size of 65KB. You can test this out directly as given in "*curl5-1.sh*" This will send a directly to productpage with a large header but just less than the 65KB limit of productpage.

But this will fail, because the sidecar attached to productpage has 60kB header limits and will reject this with 431 from sidecar.  You can check the sidecar logs and also in response text.

    From sidecar -  [2024-02-19T05:57:23.502Z] "- - HTTP/1.1" 431 DPE http1.headers_too_large - "-" 0 31 0 - "-" "-" "-" "-" "-" - - 10.244.0.89:9080 10.244.0.1:21247 - -
    From curl response -   HTTP/1.1 431 Request Header Fields Too Large

Apply the envoyfilter named "*envoyfilter5-1yaml*"
Now execute the "*curl5-1.sh*". Now traffic will pass and you will see productpage loading.

**3. Traffic through the gateway.**

Execute the script in "*./curl5-2.sh*" Traffic will fail again. Because still in the ingressgateway pod header limits is 60kB and rejects the traffic. But it does not respond with 431. In the curl response there is a warning.

    From ingressgateway - [2024-02-19T06:05:46.079Z] "- - HTTP/2" 0 - http2.too_many_headers - "-" 0 0 0 - "-" "-" "-" "-" "-" - - 10.244.0.87:8443 10.244.0.1:57697 bookinfo.aegle.info -
    From curl response - * http2_send: Warning: The cumulative length of all headers exceeds 60000 bytes and that could cause the stream to be rejected.

Apply the envoyfilter named "*envoyfilter5-2.yaml*"
Now execute the "*curl5-2.sh*". Now traffic will pass and you will see productpage loading.

**4. Notes**

You can edit the field "*max_request_headers_kb*" to lower the header size for your application's security as well.
