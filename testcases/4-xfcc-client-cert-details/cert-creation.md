## Server certs

mkdir example_certs4

openssl req -x509 -sha256 -nodes -days 3650 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example_certs4/example.com.key -out example_certs4/example.com.crt


openssl req -out example_certs4/httpbin.example.com.csr -newkey rsa:2048 -nodes -keyout example_certs4/httpbin.example.com.key -subj "/CN=httpbin.example.com/O=httpbin organization"

openssl x509 -req -sha256 -days 3650 -CA example_certs4/example.com.crt -CAkey example_certs4/example.com.key -set_serial 0 -in example_certs4/httpbin.example.com.csr -out example_certs4/httpbin.example.com.crt


## Client cert

openssl req -out example_certs4/client.example.com.csr -newkey rsa:2048 -nodes -keyout example_certs4/client.example.com.key -subj "/CN=client.example.com/O=client organization"

openssl x509 -req -sha256 -days 3650 -CA example_certs4/example.com.crt -CAkey example_certs4/example.com.key -set_serial 1 -in example_certs4/client.example.com.csr -out example_certs4/client.example.com.crt


## Apply it in k8s namespace istio-system

 kubectl create -n istio-system secret generic httpbin-credential \
  --from-file=tls.key=example_certs4/httpbin.example.com.key \
  --from-file=tls.crt=example_certs4/httpbin.example.com.crt \
  --from-file=ca.crt=example_certs4/example.com.crt

