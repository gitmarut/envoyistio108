## Purpose

This simple envoyfilter can help you setting a idle timeout in TCP connections from the server side. Default idle timeout is 1 hour for tcp connections through envoy.

## How to test it

**1. Install a tcp application**

Deploy tcp-random-app application using the yaml file given in with this testcase. Source code for this app can be seen [here](https://github.com/gitmarut/tcp-random-app). This simple App generates random numbers on a TCP connection. Istio injection is enabled in test namespace where this app is installed. Service is exposed as a loadbalancer (I am using a kind cluster with MetalLB in my linux VM).

**2. Connect to the app**
From your host linux VM terminal where kind k8s cluster is running, connect to the tcp random app using "nc" as given below.

    nc <Load Balancer IP> 17777
Press "Enter" or "Return" key 3-4 times to see random numbers are generated and printed to the screen. 

You can check logs of the tcp random app pod as given. It will print logs when a connection is started or closed.

    kubectl logs -n tcp-random-app       <tcp-random-app-pod-name>  -f

See that connection is not timed out even if you not entering "Enter" or "Return" key for 3 minutes.

**3. Apply envoy filter and test again**

Apply the envoyfilter given.

From your host linux VM terminal where kind k8s cluster is running, connect to the tcp random app using "nc" as given below.

    nc <Load Balancer IP> 17777
Press "Enter" or "Return" key 3-4 times to see random numbers are generated and printed to the screen. 

You can check logs of the tcp random app pod as given. It will print logs when a connection is started or closed.

    kubectl logs -n tcp-random-app       <tcp-random-app-pod-name>  -f

Make sure you don''t press enter(return) key many times. You will see that connection times out after 60 seconds from last enter(return) key press.

