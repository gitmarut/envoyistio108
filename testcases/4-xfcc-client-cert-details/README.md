## Purpose

This envoyfilter will demonstrate how client certificate can be forwarded or not forwarded to actual application.

XFCC is a proxy header which indicates certificate information of part or all of the clients or proxies that a request has flowed through, on its way from the client to the server.
## How to test it

  **1, Install httpbin app, GW and VS**

Deploy httpbin application (yaml [here](https://github.com/istio/istio/blob/master/samples/httpbin/httpbin.yaml)) to httpbin namespace. Istio injection is enabled in httpbin namespace.

Expose the httpbin service using a istio gateway and a istio VS. Gateway yaml and VS yaml is given in this directory. Make sure istio ingress gateway is of service type LoadBalancer. (I am using a kind cluster with MetalLB in my linux VM).

A point to note is GW is setup using MUTUAL TLS mode. Means client side has to send it's cert with the curl request. All the cert generation process is given in "cert-creation.md" file.
```
NOTE:
I have installed istio with istioctl and default profile
```
Without applying any envoyfilter in the ingress-gateway pod let's check how does it handle the XFCC. For that check the ingress gateway pod name and execute the following command with the ingress gateway pod name.

    istioctl pc all  [istio-ingressgateway-podnamehash].istio-system -o json > ig-config-dump
Open the file "ig-config-dump" and look for values - "*forward_client_cert_details*" & "*set_current_client_cert_details*". First one will be "SANITIZE_SET" and then four boolean true.

        "use_remote_address": true,
        "forward_client_cert_details": "SANITIZE_SET",
        "set_current_client_cert_details": {
         "subject": true,
         "cert": true,
         "dns": true,
         "uri": true
        },

"*set_current_client_cert_details*" intended to select what fields should be forwarded to next proxy.

Send a curl like given below and see the XFCC is populated in the output with big string, basically the whole client cert you are sending in the curl.

    curl -v -s https://httpbin.example.com/get?show_env=true --connect-to httpbin.example.com:443:172.18.64.1:443 --cacert ./example_certs4/example.com.crt --cert ./example_certs4/client.example.com.crt --key ./example_certs4/client.example.com.key

**2. Apply the envoyfilter4**

Apply the envoyfilter given in the testcase.

Check the ingress gateway pod's istio config as given in the previous step and check "*forward_client_cert_details*" & "*set_current_client_cert_details*" . First one is changed to  "APPEND_FORWARD" and then four boolean true. So what we set for "*set_current_client_cert_details*" is not taking any effect.

        "use_remote_address": true,
        "forward_client_cert_details": "APPEND_FORWARD"
        "set_current_client_cert_details": {
         "subject": true,
         "cert": true,
         "dns": true,
         "uri": true
        },

As per the documentation, "*set_current_client_cert_details*" should be effective when values for "*forward_client_cert_details*"  is either "SANITIZE_SET" or "APPEND_FORWARD". But there is some bug in the code where you cannot unset the boolean values in istio-proxy. Mostly this is the [bug](https://github.com/istio/istio/issues/18169#issuecomment-1673581700) and it is still present. Nevertheless "*chain*"  can be set and unset as it is does not have a default value in the code.

You can further edit the envoyfilter directly with:

    kubectl edit envoyfilter -n istio-system xffc-client-cert-details-4

Change the "*forward_client_cert_details*" to "FORWARD_ONLY". This will strip of the client cert from the XFCC header.

But you can play with "*forward_client_cert_details*" and see how it behaves.
Docmentation is [here](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto).

**3. Some observations**
After setting some of the values given in documentation, on restart of the ingress gateway pod it wont come up. Only after removal of envoyfilter it came up. Need to handle this in automation.

