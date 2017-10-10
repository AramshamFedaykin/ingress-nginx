The GCE ingress controller was moved to [github.com/kubernetes/ingress-gce](https://github.com/kubernetes/ingress-gce).

---

# NGINX Ingress Controller

[![Build Status](https://travis-ci.org/kubernetes/ingress-nginx.svg?branch=master)](https://travis-ci.org/kubernetes/ingress-nginx)
[![Coverage Status](https://coveralls.io/repos/github/kubernetes/ingress-nginx/badge.svg?branch=master)](https://coveralls.io/github/kubernetes/ingress-nginx?branch=master)
[![Go Report Card](https://goreportcard.com/badge/github.com/kubernetes/ingress-nginx)](https://goreportcard.com/report/github.com/kubernetes/ingress-nginx)
[![GoDoc](https://godoc.org/github.com/kubernetes/ingress-nginx?status.svg)](https://godoc.org/github.com/kubernetes/ingress-nginx)

## Description

This repository contains the NGINX controller built around the [Kubernetes Ingress resource](http://kubernetes.io/docs/user-guide/ingress/) that uses [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configmap/#understanding-configmaps) to store the NGINX configuration.

Learn more about using Ingress on [k8s.io](http://kubernetes.io/docs/user-guide/ingress/)

### What is an Ingress Controller?

Configuring a webserver or loadbalancer is harder than it should be. Most webserver configuration files are very similar. There are some applications that have weird little quirks that tend to throw a wrench in things, but for the most part you can apply the same logic to them and achieve a desired result.

The Ingress resource embodies this idea, and an Ingress controller is meant to handle all the quirks associated with a specific "class" of Ingress.

An Ingress Controller is a daemon, deployed as a Kubernetes Pod, that watches the apiserver's `/ingresses` endpoint for updates to the [Ingress resource](https://kubernetes.io/docs/concepts/services-networking/ingress/). Its job is to satisfy requests for Ingresses.

## Contents

* [Conventions](#conventions)
* [Requirements](#requirements)
* [Contribute](#contribute)
* [Command line arguments](#command-line-arguments)
* [Deployment](#deployment)
* [HTTP](#http)
* [HTTPS](#https)
  * [Default SSL Certificate](#default-ssl-certificate)
  * [HTTPS enforcement](#server-side-https-enforcement)
  * [HSTS](#http-strict-transport-security)
  * [Kube-Lego](#automated-certificate-management-with-kube-lego)
* [Source IP address](#source-ip-address)
* [TCP Services](#exposing-tcp-services)
* [UDP Services](#exposing-udp-services)
* [Proxy Protocol](#proxy-protocol)
* [ModSecurity Web Application Firewall](#modsecurity-web-application-firewall)
* [Opentracing](#opentracing)
* [NGINX customization](configuration.md)
* [Custom errors](#custom-errors)
* [NGINX status page](#nginx-status-page)
* [Running multiple ingress controllers](#running-multiple-ingress-controllers)
* [Running on Cloudproviders](#running-on-cloudproviders)
* [Disabling NGINX ingress controller](#disabling-nginx-ingress-controller)
* [Log format](#log-format)
* [Local cluster](#local-cluster)
* [Debug & Troubleshooting](#debug--troubleshooting)
* [Limitations](#limitations)
* [Why endpoints and not services?](#why-endpoints-and-not-services)
* [NGINX Notes](#nginx-notes)

## Conventions

Anytime we reference a tls secret, we mean (x509, pem encoded, RSA 2048, etc). You can generate such a certificate with:
`openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ${KEY_FILE} -out ${CERT_FILE} -subj "/CN=${HOST}/O=${HOST}"`
and create the secret via `kubectl create secret tls ${CERT_NAME} --key ${KEY_FILE} --cert ${CERT_FILE}`

## Requirements

Default backend [404-server](https://github.com/kubernetes/ingress/tree/master/images/404-server)

## Contribute

See the [contributor guidelines](CONTRIBUTING.md)

## Command line arguments

```console
Usage of :
      --alsologtostderr                  log to standard error as well as files
      --apiserver-host string            The address of the Kubernetes Apiserver to connect to in the format of protocol://address:port, e.g., http://localhost:8080. If not specified, the assumption is that the binary runs inside a Kubernetes cluster and local discovery is attempted.
      --configmap string                 Name of the ConfigMap that contains the custom configuration to use
      --default-backend-service string   Service used to serve a 404 page for the default backend. Takes the form
    	namespace/name. The controller uses the first node port of this Service for
    	the default backend.
      --default-server-port int          Default port to use for exposing the default server (catch all) (default 8181)
      --default-ssl-certificate string   Name of the secret
		that contains a SSL certificate to be used as default for a HTTPS catch-all server
      --disable-node-list                Disable querying nodes. If --force-namespace-isolation is true, this should also be set.
      --election-id string               Election id to use for status update. (default "ingress-controller-leader")
      --enable-ssl-passthrough           Enable SSL passthrough feature. Default is disabled
      --force-namespace-isolation        Force namespace isolation. This flag is required to avoid the reference of secrets or
		configmaps located in a different namespace than the specified in the flag --watch-namespace.
      --health-check-path string         Defines
		the URL to be used as health check inside in the default server in NGINX. (default "/healthz")
      --healthz-port int                 port for healthz endpoint. (default 10254)
      --http-port int                    Indicates the port to use for HTTP traffic (default 80)
      --https-port int                   Indicates the port to use for HTTPS traffic (default 443)
      --ingress-class string             Name of the ingress class to route through this controller.
      --kubeconfig string                Path to kubeconfig file with authorization and master location information.
      --log_backtrace_at traceLocation   when logging hits line file:N, emit a stack trace (default :0)
      --log_dir string                   If non-empty, write log files in this directory
      --logtostderr                      log to standard error instead of files
      --profiling                        Enable profiling via web interface host:port/debug/pprof/ (default true)
      --publish-service string           Service fronting the ingress controllers. Takes the form
 		namespace/name. The controller will set the endpoint records on the
 		ingress objects to reflect those on the service.
      --sort-backends                    Defines if backends and it's endpoints should be sorted
      --ssl-passtrough-proxy-port int    Default port to use internally for SSL when SSL Passthgough is enabled (default 442)
      --status-port int                  Indicates the TCP port to use for exposing the nginx status page (default 18080)
      --stderrthreshold severity         logs at or above this threshold go to stderr (default 2)
      --sync-period duration             Relist and confirm cloud resources this often. Default is 10 minutes (default 10m0s)
      --tcp-services-configmap string    Name of the ConfigMap that contains the definition of the TCP services to expose.
		The key in the map indicates the external port to be used. The value is the name of the
		service with the format namespace/serviceName and the port of the service could be a
		number of the name of the port.
		The ports 80 and 443 are not allowed as external ports. This ports are reserved for the backend
      --udp-services-configmap string    Name of the ConfigMap that contains the definition of the UDP services to expose.
		The key in the map indicates the external port to be used. The value is the name of the
		service with the format namespace/serviceName and the port of the service could be a
		number of the name of the port.
      --update-status                    Indicates if the
		ingress controller should update the Ingress status IP/hostname. Default is true (default true)
      --update-status-on-shutdown        Indicates if the
		ingress controller should update the Ingress status IP/hostname when the controller
		is being stopped. Default is true (default true)
  -v, --v Level                          log level for V logs
      --vmodule moduleSpec               comma-separated list of pattern=N settings for file-filtered logging
      --watch-namespace string           Namespace to watch for Ingress. Default is to watch all namespaces
```

## Deployment

First create a default backend and it's corresponding service:

```console
kubectl create -f examples/default-backend.yaml
```

Follow the [example-deployment](examples/deployment/README.md) steps to deploy nginx-ingress-controller in Kubernetes cluster (you may prefer other type of workloads, like Daemonset, in production environment).
Loadbalancers are created via a ReplicationController or Daemonset:

## HTTP

First we need to deploy some application to publish. To keep this simple we will use the [echoheaders app](https://github.com/kubernetes/contrib/blob/master/ingress/echoheaders/echo-app.yaml) that just returns information about the http request as output

```console
kubectl run echoheaders --image=gcr.io/google_containers/echoserver:1.8 --replicas=1 --port=8080
```

Now we expose the same application in two different services (so we can create different Ingress rules)

```console
kubectl expose deployment echoheaders --port=80 --target-port=8080 --name=echoheaders-x
kubectl expose deployment echoheaders --port=80 --target-port=8080 --name=echoheaders-y
```

Next we create a couple of Ingress rules

```console
kubectl create -f examples/ingress.yaml
```

we check that ingress rules are defined:

```console
$ kubectl get ing
NAME      RULE          BACKEND   ADDRESS
echomap   -
          foo.bar.com
          /foo          echoheaders-x:80
          bar.baz.com
          /bar          echoheaders-y:80
          /foo          echoheaders-x:80
```

Before the deploy of the Ingress controller we need a default backend [404-server](https://github.com/kubernetes/contrib/tree/master/404-server)

```console
kubectl create -f examples/default-backend.yaml
kubectl expose rc default-http-backend --port=80 --target-port=8080 --name=default-http-backend
```

Check NGINX it is running with the defined Ingress rules:

```console
$ LBIP=$(kubectl get node `kubectl get po -l name=nginx-ingress-lb --template '{{range .items}}{{.spec.nodeName}}{{end}}'` --template '{{range $i, $n := .status.addresses}}{{if eq $n.type "ExternalIP"}}{{$n.address}}{{end}}{{end}}')
$ curl $LBIP/foo -H 'Host: foo.bar.com'
```

## HTTPS

You can secure an Ingress by specifying a secret that contains a TLS private key and certificate. Currently the Ingress only supports a single TLS port, 443, and assumes TLS termination. This controller supports SNI. The TLS secret must contain keys named tls.crt and tls.key that contain the certificate and private key to use for TLS, eg:

```yaml
apiVersion: v1
data:
  tls.crt: base64 encoded cert
  tls.key: base64 encoded key
kind: Secret
metadata:
  name: foo-secret
  namespace: default
type: kubernetes.io/tls
```

Referencing this secret in an Ingress will tell the Ingress controller to secure the channel from the client to the loadbalancer using TLS:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: no-rules-map
spec:
  tls:
    secretName: foo-secret
  backend:
    serviceName: s1
    servicePort: 80
```

Please follow [PREREQUISITES](examples/PREREQUISITES.md) as a guide on how to generate secrets containing SSL certificates. The name of the secret can be different than the name of the certificate.

Check the [example](examples/tls-termination/nginx)

### Default SSL Certificate

NGINX provides the option [server name _](http://nginx.org/en/docs/http/server_names.html) as a catch-all in case of requests that do not match one of the configured server names. This configuration works without issues for HTTP traffic. In case of HTTPS, NGINX requires a certificate. For this reason the Ingress controller provides the flag `--default-ssl-certificate`. The secret behind this flag contains the default certificate to be used in the mentioned case. If this flag is not provided NGINX will use a self signed certificate.

Running without the flag `--default-ssl-certificate`:

```console
$ curl -v https://10.2.78.7:443 -k
* Rebuilt URL to: https://10.2.78.7:443/
*   Trying 10.2.78.4...
* Connected to 10.2.78.7 (10.2.78.7) port 443 (#0)
* ALPN, offering http/1.1
* Cipher selection: ALL:!EXPORT:!EXPORT40:!EXPORT56:!aNULL:!LOW:!RC4:@STRENGTH
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/certs/ca-certificates.crt
  CApath: /etc/ssl/certs
* TLSv1.2 (OUT), TLS header, Certificate Status (22):
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Client hello (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS change cipher, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES128-GCM-SHA256
* ALPN, server accepted to use http/1.1
* Server certificate:
*    subject: CN=foo.bar.com
*    start date: Apr 13 00:50:56 2016 GMT
*    expire date: Apr 13 00:50:56 2017 GMT
*    issuer: CN=foo.bar.com
*    SSL certificate verify result: self signed certificate (18), continuing anyway.
> GET / HTTP/1.1
> Host: 10.2.78.7
> User-Agent: curl/7.47.1
> Accept: */*
>
< HTTP/1.1 404 Not Found
< Server: nginx/1.11.1
< Date: Thu, 21 Jul 2016 15:38:46 GMT
< Content-Type: text/html
< Transfer-Encoding: chunked
< Connection: keep-alive
< Strict-Transport-Security: max-age=15724800; includeSubDomains; preload
<
<span>The page you're looking for could not be found.</span>

* Connection #0 to host 10.2.78.7 left intact
```

Specifying `--default-ssl-certificate=default/foo-tls`:

```console
core@localhost ~ $ curl -v https://10.2.78.7:443 -k
* Rebuilt URL to: https://10.2.78.7:443/
*   Trying 10.2.78.7...
* Connected to 10.2.78.7 (10.2.78.7) port 443 (#0)
* ALPN, offering http/1.1
* Cipher selection: ALL:!EXPORT:!EXPORT40:!EXPORT56:!aNULL:!LOW:!RC4:@STRENGTH
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/certs/ca-certificates.crt
  CApath: /etc/ssl/certs
* TLSv1.2 (OUT), TLS header, Certificate Status (22):
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Client hello (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS change cipher, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES128-GCM-SHA256
* ALPN, server accepted to use http/1.1
* Server certificate:
*    subject: CN=foo.bar.com
*    start date: Apr 13 00:50:56 2016 GMT
*    expire date: Apr 13 00:50:56 2017 GMT
*    issuer: CN=foo.bar.com
*    SSL certificate verify result: self signed certificate (18), continuing anyway.
> GET / HTTP/1.1
> Host: 10.2.78.7
> User-Agent: curl/7.47.1
> Accept: */*
>
< HTTP/1.1 404 Not Found
< Server: nginx/1.11.1
< Date: Mon, 18 Jul 2016 21:02:59 GMT
< Content-Type: text/html
< Transfer-Encoding: chunked
< Connection: keep-alive
< Strict-Transport-Security: max-age=15724800; includeSubDomains; preload
<
<span>The page you're looking for could not be found.</span>

* Connection #0 to host 10.2.78.7 left intact
```

### Server-side HTTPS enforcement

By default the controller redirects (301) to HTTPS if TLS is enabled for that ingress . If you want to disable that behaviour globally, you can use `ssl-redirect: "false"` in the NGINX config map.

To configure this feature for specific ingress resources, you can use the `ingress.kubernetes.io/ssl-redirect: "false"` annotation in the particular resource.

### HTTP Strict Transport Security

HTTP Strict Transport Security (HSTS) is an opt-in security enhancement specified through the use of a special response header. Once a supported browser receives this header that browser will prevent any communications from being sent over HTTP to the specified domain and will instead send all communications over HTTPS.

By default the controller redirects (301) to HTTPS if there is a TLS Ingress rule.

To disable this behavior use `hsts=false` in the NGINX config map.

### Automated Certificate Management with Kube-Lego

[Kube-Lego] automatically requests missing or expired certificates from [Let's Encrypt] by monitoring ingress resources and their referenced secrets. To enable this for an ingress resource you have to add an annotation:

```console
kubectl annotate ing ingress-demo kubernetes.io/tls-acme="true"
```

To setup Kube-Lego you can take a look at this [full example]. The first
version to fully support Kube-Lego is nginx Ingress controller 0.8.

[full example]:https://github.com/jetstack/kube-lego/tree/master/examples
[Kube-Lego]:https://github.com/jetstack/kube-lego
[Let's Encrypt]:https://letsencrypt.org

## Source IP address

By default NGINX uses the content of the header `X-Forwarded-For` as the source of truth to get information about the client IP address. This works without issues in L7 **if we configure the setting `proxy-real-ip-cidr`** with the correct information of the IP/network address of the external load balancer.
If the ingress controller is running in AWS we need to use the VPC IPv4 CIDR. This allows NGINX to avoid the spoofing of the header.
Another option is to enable proxy protocol using `use-proxy-protocol: "true"`.
In this mode NGINX do not uses the content of the header to get the source IP address of the connection.

## Exposing TCP services

Ingress does not support TCP services (yet). For this reason this Ingress controller uses the flag `--tcp-services-configmap` to point to an existing config map where the key is the external port to use and the value is `<namespace/service name>:<service port>:[PROXY]:[PROXY]`
It is possible to use a number or the name of the port. The two last fields are optional. Adding `PROXY` in either or both of the two last fields we can use Proxy Protocol decoding (listen) and/or encoding (proxy_pass) in a TCP service (https://www.nginx.com/resources/admin-guide/proxy-protocol/).

The next example shows how to expose the service `example-go` running in the namespace `default` in the port `8080` using the port `9000`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tcp-configmap-example
data:
  9000: "default/example-go:8080"
```

Please check the [tcp services](examples/tcp/README.md) example

## Exposing UDP services

Since 1.9.13 NGINX provides [UDP Load Balancing](https://www.nginx.com/blog/announcing-udp-load-balancing/).

Ingress does not support UDP services (yet). For this reason this Ingress controller uses the flag `--udp-services-configmap` to point to an existing config map where the key is the external port to use and the value is `<namespace/service name>:<service port>`
It is possible to use a number or the name of the port.

The next example shows how to expose the service `kube-dns` running in the namespace `kube-system` in the port `53` using the port `53`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: udp-configmap-example
data:
  53: "kube-system/kube-dns:53"
```


Please check the [udp services](examples/udp/README.md) example

## Proxy Protocol

If you are using a L4 proxy to forward the traffic to the NGINX pods and terminate HTTP/HTTPS there, you will lose the remote endpoint's IP addresses. To prevent this you could use the [Proxy Protocol](http://www.haproxy.org/download/1.5/doc/proxy-protocol.txt) for forwarding traffic, this will send the connection details before forwarding the actual TCP connection itself.

Amongst others [ELBs in AWS](http://docs.aws.amazon.com/ElasticLoadBalancing/latest/DeveloperGuide/enable-proxy-protocol.html) and [HAProxy](http://www.haproxy.org/) support Proxy Protocol.

Please check the [proxy-protocol](examples/proxy-protocol/) example

## ModSecurity Web Application Firewall

ModSecurity is an open source, cross platform web application firewall (WAF) engine for Apache, IIS and Nginx that is developed by Trustwave's SpiderLabs. It has a robust event-based programming language which provides protection from a range of attacks against web applications and allows for HTTP traffic monitoring, logging and real-time analys… https://www.modsecurity.org

The [ModSecurity-nginx](https://github.com/SpiderLabs/ModSecurity-nginx) connector is the connection point between NGINX and libmodsecurity (ModSecurity v3).

The default modsecurity configuration file is located in `/etc/nginx/modsecurity/modsecurity.conf`. This is the only file located in this directory and it contains the default recommended configuration. Using a volume we can replace this file with the desired configuration.
To enable the modsecurity feature we need to specify `enable-modsecurity: "true"` in the configuration configmap.

The OWASP ModSecurity Core Rule Set (CRS) is a set of generic attack detection rules for use with ModSecurity or compatible web application firewalls. The CRS aims to protect web applications from a wide range of attacks, including the OWASP Top Ten, with a minimum of false alerts.
The directory `/etc/nginx/owasp-modsecurity-crs` contains the https://github.com/SpiderLabs/owasp-modsecurity-crs repository.
Using `enable-owasp-modsecurity-crs: "true"` we enable the use of the this rules.

## Opentracing

Using the third party module [rnburn/nginx-opentracing](https://github.com/rnburn/nginx-opentracing) the NGINX ingress controller can configure NGINX to enable [OpenTracing](http://opentracing.io) instrumentation.
By default this feature is disabled.

To enable the instrumentation we just need to enable the instrumentation in the configuration configmap and set the host where we should send the traces.

In the [aledbf/zipkin-js-example](https://github.com/aledbf/zipkin-js-example) github repository is possible to see a dockerized version of zipkin-js-example with the required Kubernetes descriptors.
To install the example and the zipkin collector we just need to run:

```
kubectl create -f https://raw.githubusercontent.com/aledbf/zipkin-js-example/kubernetes/kubernetes/zipkin.yaml
kubectl create -f https://raw.githubusercontent.com/aledbf/zipkin-js-example/kubernetes/kubernetes/deployment.yaml
```

Also we need to configure the NGINX controller configmap with the required values:

```yaml
apiVersion: v1
data:
  enable-opentracing: "true"
  zipkin-collector-host: zipkin.default.svc.cluster.local
kind: ConfigMap
metadata:
  labels:
    k8s-app: nginx-ingress-controller
  name: nginx-custom-configuration
```

Using curl we can generate some traces:

```console
$ curl -v http://$(minikube ip)/api -H 'Host: zipkin-js-example'
$ curl -v http://$(minikube ip)/api -H 'Host: zipkin-js-example'
```

In the zipkin inteface we can see the details:

![zipkin screenshot](docs/images/zipkin-demo.png "zipkin collector screenshot")

### Custom errors

In case of an error in a request the body of the response is obtained from the `default backend`.
Each request to the default backend includes two headers:

- `X-Code` indicates the HTTP code to be returned to the client.
- `X-Format` the value of the `Accept` header.

**Important:** the custom backend must return the correct HTTP status code to be returned. NGINX do not changes the reponse from the custom default backend.

Using this two headers is possible to use a custom backend service like [this one](https://github.com/kubernetes/ingress/tree/master/examples/customization/custom-errors/nginx) that inspect each request and returns a custom error page with the format expected by the client. Please check the example [custom-errors](examples/customization/custom-errors/README.md)

NGINX sends aditional headers that can be used to build custom response:

- X-Original-URI
- X-Namespace
- X-Ingress-Name
- X-Service-Name

### NGINX status page

The ngx_http_stub_status_module module provides access to basic status information. This is the default module active in the url `/nginx_status`.
This controller provides an alternative to this module using [nginx-module-vts](https://github.com/vozlt/nginx-module-vts) third party module.
To use this module just provide a config map with the key `enable-vts-status=true`. The URL is exposed in the port 18080.
Please check the example `example/rc-default.yaml`

![nginx-module-vts screenshot](https://cloud.githubusercontent.com/assets/3648408/10876811/77a67b70-8183-11e5-9924-6a6d0c5dc73a.png "screenshot with filter")

To extract the information in JSON format the module provides a custom URL: `/nginx_status/format/json`

### Running multiple ingress controllers

If you're running multiple ingress controllers, or running on a cloudprovider that natively handles ingress, you need to specify the annotation `kubernetes.io/ingress.class: "nginx"` in all ingresses that you would like this controller to claim.
Not specifying the annotation will lead to multiple ingress controllers claiming the same ingress. Specifying the wrong value will result in all ingress controllers ignoring the ingress.
Multiple ingress controllers running in the same cluster was not supported in Kubernetes versions < 1.3.

### Running on Cloudproviders

If you're running this ingress controller on a cloudprovider, you should assume the provider also has a native Ingress controller and specify the ingress.class annotation as indicated in this section.
In addition to this, you will need to add a firewall rule for each port this controller is listening on, i.e :80 and :443.

### Disabling NGINX ingress controller

Setting the annotation `kubernetes.io/ingress.class` to any value other than "nginx" or the empty string, will force the NGINX Ingress controller to ignore your Ingress. Do this if you wish to use one of the other Ingress controllers at the same time as the NGINX controller.

### Log format

The default configuration uses a custom logging format to add additional information about upstreams

```
    log_format upstreaminfo '{{ if $cfg.useProxyProtocol }}$proxy_protocol_addr{{ else }}$remote_addr{{ end }} - '
        '[$proxy_add_x_forwarded_for] - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" '
        '$request_length $request_time [$proxy_upstream_name] $upstream_addr $upstream_response_length $upstream_response_time $upstream_status';
```

Sources:
  - [upstream variables](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#variables)
  - [embedded variables](http://nginx.org/en/docs/http/ngx_http_core_module.html#variables)

Description:
- `$proxy_protocol_addr`: if PROXY protocol is enabled
- `$remote_addr`: if PROXY protocol is disabled (default)
- `$proxy_add_x_forwarded_for`: the `X-Forwarded-For` client request header field with the $remote_addr variable appended to it, separated by a comma
- `$remote_user`: user name supplied with the Basic authentication
- `$time_local`: local time in the Common Log Format
- `$request`: full original request line
- `$status`: response status
- `$body_bytes_sent`: number of bytes sent to a client, not counting the response header
- `$http_referer`: value of the Referer header
- `$http_user_agent`: value of User-Agent header
- `$request_length`: request length (including request line, header, and request body)
- `$request_time`: time elapsed since the first bytes were read from the client
- `$proxy_upstream_name`: name of the upstream. The format is `upstream-<namespace>-<service name>-<service port>`
- `$upstream_addr`: keeps the IP address and port, or the path to the UNIX-domain socket of the upstream server. If several servers were contacted during request processing, their addresses are separated by commas
- `$upstream_response_length`: keeps the length of the response obtained from the upstream server
- `$upstream_response_time`: keeps time spent on receiving the response from the upstream server; the time is kept in seconds with millisecond resolution
- `$upstream_status`: keeps status code of the response obtained from the upstream server

### Local cluster

Using [`hack/local-up-cluster.sh`](https://github.com/kubernetes/kubernetes/blob/master/hack/local-up-cluster.sh) is possible to start a local kubernetes cluster consisting of a master and a single node. Please read [running-locally.md](https://github.com/kubernetes/community/blob/master/contributors/devel/running-locally.md) for more details.

Use of `hostNetwork: true` in the ingress controller is required to falls back at localhost:8080 for the apiserver if every other client creation check fails (eg: service account not present, kubeconfig doesn't exist, no master env vars...)

### Debug & Troubleshooting

Using the flag `--v=XX` it is possible to increase the level of logging.
In particular:
- `--v=2` shows details using `diff` about the changes in the configuration in nginx

```console
I0316 12:24:37.581267       1 utils.go:148] NGINX configuration diff a//etc/nginx/nginx.conf b//etc/nginx/nginx.conf
I0316 12:24:37.581356       1 utils.go:149] --- /tmp/922554809  2016-03-16 12:24:37.000000000 +0000
+++ /tmp/079811012  2016-03-16 12:24:37.000000000 +0000
@@ -235,7 +235,6 @@

     upstream default-echoheadersx {
         least_conn;
-        server 10.2.112.124:5000;
         server 10.2.208.50:5000;

     }
I0316 12:24:37.610073       1 command.go:69] change in configuration detected. Reloading...
```

- `--v=3` shows details about the service, Ingress rule, endpoint changes and it dumps the nginx configuration in JSON format
- `--v=5` configures NGINX in [debug mode](http://nginx.org/en/docs/debugging_log.html)

Peruse the [FAQ section](docs/faq/README.md)
Ask on one of the [user-support channels](CONTRIBUTING.md#support-channels)

### Limitations

- Ingress rules for TLS require the definition of the field `host`

### Why endpoints and not services

The NGINX ingress controller does not uses [Services](http://kubernetes.io/docs/user-guide/services) to route traffic to the pods. Instead it uses the Endpoints API in order to bypass [kube-proxy](http://kubernetes.io/docs/admin/kube-proxy/) to allow NGINX features like session affinity and custom load balancing algorithms. It also removes some overhead, such as conntrack entries for iptables DNAT.

### NGINX notes

Since `gcr.io/google_containers/nginx-slim:0.8` NGINX contains the next patches:
- Dynamic TLS record size [nginx__dynamic_tls_records.patch](https://blog.cloudflare.com/optimizing-tls-over-tcp-to-reduce-latency/)
NGINX provides the parameter `ssl_buffer_size` to adjust the size of the buffer. Default value in NGINX is 16KB. The ingress controller changes the default to 4KB. This improves the [TLS Time To First Byte (TTTFB)](https://www.igvita.com/2013/12/16/optimizing-nginx-tls-time-to-first-byte/) but the size is fixed. This patches adapts the size of the buffer to the content is being served helping to improve the perceived latency.
- [HTTP/2 header compression](https://raw.githubusercontent.com/cloudflare/sslconfig/master/patches/nginx_http2_hpack.patch)
