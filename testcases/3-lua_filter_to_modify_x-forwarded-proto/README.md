## Purpose

This envoyfilter is a lua filter to replace the headers on the requests. 
In this example lua code is replacing the header "x-forwarded-proto" to http even if the original request is https. (I know it sound silly)

## How to test it

  **1, Install httpbin app, GW and VS as in testcase 2 (fetch_original_ip_addr_of_curl_req)**

Deploy httpbin application (yaml [here](https://github.com/istio/istio/blob/master/samples/httpbin/httpbin.yaml)) to httpbin namespace. Istio injection is enabled in httpbin namespace.

Expose the httpbin service using a istio gateway and a istio VS. Gateway yaml and VS yaml is given in this directory. Make sure istio ingress gateway is of service type LoadBalancer. (Since I am using a kind cluster with MetalLB in my linux VM, I am not short of LB addresses).

```
NOTE
I have installed istio with istioctl and default profile
```
Send a curl request with --connecto-to flag set from host VM's shell.

    curl -v -k -s   https://httpbin.aegle.info/get?show_env=true --connect-to httpbin.aegle.info:443:172.18.64.1:443

Output will show that -  *"X-Forwarded-Proto": "https"*

**2. Apply the envoyfilter3**

Apply the envoyfilter given in the testcase.
Then send the above curl once again.
Check that output now has - *"X-Forwarded-Proto": "http"*

**3. Apply the authorization policy**

Apply the authorization policy given in the testcase. This will DENY the request if the *"X-Forwarded-Proto"* is not *"https"*

Then send the above curl once again.
Check that output now http response code 403 and a message at the end saying *"RBAC: access denied"*

