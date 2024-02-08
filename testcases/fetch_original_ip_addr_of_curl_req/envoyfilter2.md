## Purpose

This envoyfilter identifies the source IP of the request based on the configured values of 'use_remote_address' & 'xff_num_trusted_hops' mainly. All standard proxies support XFF header appending. Useful in knowing source IP of the request and may act in terms of security in the app itself, or logging, or whatever.

I have taken this envoyfilter from = https://github.com/istio/istio/issues/29893

## Explanation
Example envoyfilter has mainly three fields.
1. `skip_xff_append` - keep it false for this particular testcase (explanation [here](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto))

2. `use_remote_address` - we will play with this (explanation [here](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto))

	By default this is false. We will make it true, but the  `skip_xff_append` has to be false.

3. `xff_num_trusted_hops` - we will play with this (explanation [here](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto))

	We will test this mainly. Definition says this decides the position of the trusted client IP in a list of `X-forwarded-for` header from right most value.



## How to test it

  **1, Without envoy filter applied**

Deploy httpbin application (yaml [here](https://github.com/istio/istio/blob/master/samples/httpbin/httpbin.yaml)) to httpbin namespace. Istio injection is enabled in httpbin namespace.

Expose the httpbin service using a istio gateway and a istio VS. Gateway yaml and VS yaml is given in this directory. Make sure istio ingress gateway is of service type LoadBalancer. (Since I am using a kind cluster with MetalLB in my linux VM, I am not short of LB addresses).

Deploy a sleep application in any namespace(it does not have to be istio-injection enabled). Yaml [here.](https://github.com/istio/istio/blob/master/samples/sleep/sleep.yaml)

 - From the shell of "sleep" pod.

Use the following.

`kubectl exec -it -n sleep sleep-9454cc476-8rk8 -n  -- sh`

Do a curl from "sleep" pod's shell.

    curl -v -k -s http://httpbin.httpbin.svc.cluster.local:8000/ip
    curl -v -k -s http://httpbin.httpbin.svc.cluster.local:8000/headers

First curl will show an output as given below. This is the localhost IP address of sidecar attached to httpbin.	 

    {
      "origin": "127.0.0.6"
    }

Second curl should not show the header - `"X-Envoy-External-Address"`

 - From host Linux VM shell
 
 do curl

`curl -v -k -s https://httpbin.aegle.info/ip --connect-to httpbin.aegle.info:443:172.18.64.1:443`

`curl -v -k -s https://httpbin.aegle.info/headers --connectto httpbin.aegle.info:443:172.18.64.1:443`

First curl will show an output as given below. This IP address is first IP address from the pod IP range in kubernetes cluster. This curl request is coming through ingress gateway pod and by default for any gateway pod "use_remote_address" is enabled.

    {
      "origin": "10.244.0.1"
    }

Second curl should not show the header - `"X-Envoy-External-Address"`

 **2. With envoy filter applied**

Apply the envoy filter given. 

    kubectl apply -f envoyfilter2.yaml

 - From sleep pod's shell
 
 send the same curl as above

    `curl -v -k -s http://httpbin.httpbin.svc.cluster.local:8000/ip`
    `curl -v -k -s http://httpbin.httpbin.svc.cluster.local:8000/headers`

First curl will show the pod ip of sleep in output

`{
  "origin": "10.244.0.40"
}`

Second one will not show  "X-Envoy-External-Address", since there is no external client involved. But it will show " "X-Envoy-Internal": "true", which was not there when  envoyfilter was not present. More details on headers [here](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/headers#x-envoy-internal "here").

- From host linux VM shell

do curl

`curl -v -k -s https://httpbin.aegle.info/ip --connect-to httpbin.aegle.info:443:172.18.64.1:443`

`curl -v -k -s https://httpbin.aegle.info/headers --connect-to httpbin.aegle.info:443:172.18.64.1:443`

First curl will show the client IP (in this case first IP from podCIDR) and the pod IP of the istio GW pod.

`{
  "origin": "10.244.0.1,10.244.0.30"
}`

Second one will show `"X-Envoy-External-Address": "10.244.0.1"` . This is coming from the configuration in envoyfilter `xff_num_trusted_hops`. Since this is kept as one, it will take the first one on the left from the rightmost IP.

**3. Edit Envoyfilter for xff_num_trusted_hops: 2**

Edit the envoy filter and make "xff_num_trusted_hops: 2"

- From the sleep pod

Do curl

`curl -v -k -s  --header "X-Forwarded-For: 3.1.3.100,2.1.6.4" http://httpbin.httpbin.svc.cluster.local:8000/ip
`

`curl -v -k -s  --header "X-Forwarded-For: 3.1.3.100,2.1.6.4" http://httpbin.httpbin.svc.cluster.local:8000/headers
`

First curl will give fake `X-forwarded-for` IPs, then podIP of the sleep pod.

`{
  "origin": "3.1.3.100,2.1.6.4,10.244.0.40"
}`

Second curl will show `"X-Envoy-External-Address": "3.1.3.100"` . This is because httpbin is thinking that client is `3.1.3.100` for `xff_num_trusted_hops` config set to 2 and fake `X-Forwarded-For` header present in the curl request.

If you can edit `xff_num_trusted_hops`  to 1 and try the above two curls from sleep pod, you can see that first output has not changed. But second one has now `"X-Envoy-External-Address": "2.1.6.4"` which is because of httpbin is thinking that client is `2.1.6.4` for `xff_num_trusted_hops` config set to 1 and fake `X-Forwarded-For` header present in the curl request.

- From host linux VM shell

Send curls

`curl -v -k -s --header "X-Forwarded-For: 3.1.3.100,2.1.6.4" https://httpbin.aegle.info/ip --connect-to httpbin.aegle.info:443:172.18.64.1:443`

`curl -v -k -s --header "X-Forwarded-For: 3.1.3.100,2.1.6.4" https://httpbin.aegle.info/headers --connect-to httpbin.aegle.info:443:172.18.64.1:443`

First curl will giveFirst curl will give fake `X-forwarded-for` IPs, then first IP from PodCIDR, then istio GW pod IP.

`{
  "origin": "3.1.3.100,2.1.6.4,10.244.0.1,10.244.0.43"
}`



Second curl will show `"X-Envoy-External-Address": "2.1.6.4"` . This is because httpbin is thinking that client is `2.1.6.4` for `xff_num_trusted_hops` config set to 2 and fake `X-Forwarded-For` header present in the curl request.

If you can edit `xff_num_trusted_hops`  to 3 and try the above two curls from sleep pod, you can see that first output has not changed. But second one has now `"X-Envoy-External-Address": "3.1.3.100"` which is because of httpbin is thinking that client is `3.1.3.100` for `xff_num_trusted_hops` config set to 3 and fake `X-Forwarded-For` header present in the curl request.


