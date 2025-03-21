# istio/istio's Issue #55616: ReferenceGrants stop working when mTLS is enabled for a Gateway Listener

This is the sample repository that displays the behavior associated with istio/istio's issue #55616

## Environment

- MicroK8s: `v1.32.2 revision 7731` (with `metallb` enabled)
- Istio: `client version: 1.25.0, control plane version: 1.25.0`

## Setup

1. Start `microk8s` and enable `metallb`.

```bash
microk8s start
```

```bash
microk8s enable metallb:192.128.0.105-192.128.0.106
```

1. Install a `1-25-0` revision of Istio, using the `minimal` deployment profile.

```bash
istioctl install --revision 1-25-0 --set profile=minimal
```

3. Generate root, httpbin, and client certificates.

```bash
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout certs/root/example.com.key -out certs/root/example.com.crt
```

```bash
openssl req -out certs/httpbin/httpbin.example.com.csr -newkey rsa:2048 -nodes -keyout certs/httpbin/httpbin.example.com.key -subj "/CN=httpbin.example.com/O=httpbin organization"
```

```bash
openssl x509 -req -sha256 -days 365 -CA certs/root/example.com.crt -CAkey certs/root/example.com.key -set_serial 0 -in certs/httpbin/httpbin.example.com.csr -out certs/httpbin/httpbin.example.com.crt
```

```bash
openssl req -out certs/client/client.example.com.csr -newkey rsa:2048 -nodes -keyout certs/client/client.example.com.key -subj "/CN=client.example.com/O=client organization"
```

```bash
openssl x509 -req -sha256 -days 365 -CA certs/root/example.com.crt -CAkey certs/root/example.com.key -set_serial 1 -in certs/client/client.example.com.csr -out certs/client/client.example.com.crt
```

## Enable Standard TLS

1. Deploy the `istio-gateways` and `httpbin` namespaces.

```bash
kubectl apply -f k8s/01-namespaces.yaml
```

2. Deploy the httpbin app.

```bash
kubectl apply -f apps/httpbin.yaml
```

3. Deploy a TLS Secret called `httpbin-credential` in the `httpbin` namespace with cert/key coming from `certs/httpbin`.

```bash
kubectl create secret tls -n httpbin httpbin-credential --cert certs/httpbin/httpbin.example.com.crt --key certs/httpbin/httpbin.example.com.key
```

4. Deploy the `ReferenceGrant` that will allow `Gateway` resources living in `istio-gateways` refer to `secret/httpbin-credential` living in `httpbin`.

```bash
kubectl apply -f k8s/02-httpbin-reference-grant.yaml
```

5. Deploy the gateway.

```bash
kubectl apply -f k8s/03-my-gateway-gateway.yaml
```

6. For convenience, set `INGRESS_HOST` and `SECURE_INGRESS_PORT` shell environment variables.

```bash
export INGRESS_HOST=$(kubectl get gateway my-gateway -n istio-gateways -o jsonpath='{.status.addresses[0].value}')
```

```bash
export SECURE_INGRESS_PORT=$(kubectl get gateway my-gateway -n istio-gateways -o jsonpath='{.spec.listeners[?(@.name=="https-httpbin")].port}')
```

7. Deploy the `HTTPRoute` that will forward traffic from `httpbin.example.com:443` to our `svc/httpbin` backend in service port `8000`.

```bash
kubectl apply -f k8s/04-httpbin-httproute.yaml
```

8. Verify standard TLS is working.

```stdout
$ curl -v -H Host:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" --cacert certs/root/example.com.crt "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"
* Added httpbin.example.com:443:192.128.0.105 to DNS cache
* Hostname httpbin.example.com was found in DNS cache
*   Trying 192.128.0.105:443...
* Connected to httpbin.example.com (192.128.0.105) port 443
* ALPN: curl offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
*  CAfile: certs/root/example.com.crt
*  CApath: /etc/ssl/certs
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384 / X25519 / RSASSA-PSS
* ALPN: server accepted h2
* Server certificate:
*  subject: CN=httpbin.example.com; O=httpbin organization
*  start date: Mar 21 16:46:26 2025 GMT
*  expire date: Mar 21 16:46:26 2026 GMT
*  common name: httpbin.example.com (matched)
*  issuer: O=example Inc.; CN=example.com
*  SSL certificate verify ok.
*   Certificate level 0: Public key type RSA (2048/112 Bits/secBits), signed using sha256WithRSAEncryption
*   Certificate level 1: Public key type RSA (2048/112 Bits/secBits), signed using sha256WithRSAEncryption
* using HTTP/2
* [HTTP/2] [1] OPENED stream for https://httpbin.example.com:443/status/418
* [HTTP/2] [1] [:method: GET]
* [HTTP/2] [1] [:scheme: https]
* [HTTP/2] [1] [:authority: httpbin.example.com]
* [HTTP/2] [1] [:path: /status/418]
* [HTTP/2] [1] [user-agent: curl/8.5.0]
* [HTTP/2] [1] [accept: */*]
> GET /status/418 HTTP/2
> Host:httpbin.example.com
> User-Agent: curl/8.5.0
> Accept: */*
>
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* old SSL session ID is stale, removing
< HTTP/2 418
< access-control-allow-credentials: true
< access-control-allow-origin: *
< content-type: text/plain; charset=utf-8
< x-more-info: http://tools.ietf.org/html/rfc2324
< date: Fri, 21 Mar 2025 17:18:36 GMT
< content-length: 13
< x-envoy-upstream-service-time: 13
< server: istio-envoy
<
* Connection #0 to host httpbin.example.com left intact
I'm a teapot!
```

## Failing to switch to Mutual TLS

1. Delete and recreate `secret/httpbin-credential` so it now includes the root CA's certificate.

```bash
kubectl delete -n httpbin secret httpbin-credential
```

```bash
kubectl create -n httpbin secret generic httpbin-credential --from-file=tls.key=certs/httpbin/httpbin.example.com.key --from-file=tls.crt=certs/httpbin/httpbin.example.com.crt --from-file=ca.crt=certs/root/example.com.crt
```

1. Edit `gateway/my-gateway` so the `https-httpbin` listener enforces Mutual TLS.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
  namespace: istio-gateways
spec:
  gatewayClassName: istio
  listeners:
    - name: https-httpbin
      # ...
      tls:
        # ...
        options:
          gateway.istio.io/tls-terminate-mode: MUTUAL
      # ...
```

```bash
kubectl apply -f k8s/03-my-gateway-gateway.yaml
```

3. Try to make an HTTPS request to `httpbin.example.com:443` with `curl`. You will get back a `curl: (35) Recv failure: Connection reset by peer` error message, indicating that our listener is no longer able to access certificate data on `secret/httpbin-credential`.

```stdout
$ curl -v -H Host:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" --cacert certs/root/example.com.crt "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"
* Added httpbin.example.com:443:192.128.0.105 to DNS cache
* Hostname httpbin.example.com was found in DNS cache
*   Trying 192.128.0.105:443...
* Connected to httpbin.example.com (192.128.0.105) port 443
* ALPN: curl offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
*  CAfile: certs/root/example.com.crt
*  CApath: /etc/ssl/certs
* Recv failure: Connection reset by peer
* OpenSSL SSL_connect: Connection reset by peer in connection to httpbin.example.com:443
* Closing connection
curl: (35) Recv failure: Connection reset by peer
```

> This should fail even if you try to send the client credentials.

```stdout
$ curl -v -H Host:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" --cacert certs/root/example.com.crt --cert certs/client/client.example.com.crt --key certs/client/client.example.com.key "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"
* Added httpbin.example.com:443:192.128.0.105 to DNS cache
* Hostname httpbin.example.com was found in DNS cache
*   Trying 192.128.0.105:443...
* Connected to httpbin.example.com (192.128.0.105) port 443
* ALPN: curl offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
*  CAfile: certs/root/example.com.crt
*  CApath: /etc/ssl/certs
* Recv failure: Connection reset by peer
* OpenSSL SSL_connect: Connection reset by peer in connection to httpbin.example.com:443
* Closing connection
curl: (35) Recv failure: Connection reset by peer
```

## Fixing Mutual TLS by moving credential Secrets to the gateway's Namespace

1. Delete and recreate `secret/httpbin-credential` so it lives in namespace `istio-gateways` instead of namespace `httpbin`.

```bash
kubectl delete secret/httpbin-credential -n httpbin
```

```bash
kubectl create -n istio-gateways secret generic httpbin-credential --from-file=tls.key=certs/httpbin/httpbin.example.com.key --from-file=tls.crt=certs/httpbin/httpbin.example.com.crt --from-file=ca.crt=certs/root/example.com.crt
```

2. Edit `gateway/my-gateway` so the `https-httpbin` listener finds the new location of `secret/httpbin-credential`.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
  namespace: istio-gateways
spec:
  gatewayClassName: istio
  listeners:
    - name: https-httpbin
      # ...
      tls:
        mode: Terminate
        certificateRefs:
          - group: ''
            kind: Secret
            name: httpbin-credential
            namespace: istio-gateways # or delete this field altogether
        options:
          gateway.istio.io/tls-terminate-mode: MUTUAL
      # ...
```

```bash
kubectl apply -f k8s/03-my-gateway-gateway.yaml
```

3. Make an HTTPS request again to `httpbin.example.com:443` with an _unauthenticated_ `curl` client. You should now get back the typical server certificate logs, but still failing due to a new `alert certificate required` error, which correctly demands client certificate data.

```stdout
$ curl -v -H Host:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" --cacert certs/root/example.com.crt "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"
* Added httpbin.example.com:443:192.128.0.105 to DNS cache
* Hostname httpbin.example.com was found in DNS cache
*   Trying 192.128.0.105:443...
* Connected to httpbin.example.com (192.128.0.105) port 443
* ALPN: curl offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
*  CAfile: certs/root/example.com.crt
*  CApath: /etc/ssl/certs
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Request CERT (13):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (OUT), TLS handshake, Certificate (11):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384 / X25519 / RSASSA-PSS
* ALPN: server accepted h2
* Server certificate:
*  subject: CN=httpbin.example.com; O=httpbin organization
*  start date: Mar 21 16:46:26 2025 GMT
*  expire date: Mar 21 16:46:26 2026 GMT
*  common name: httpbin.example.com (matched)
*  issuer: O=example Inc.; CN=example.com
*  SSL certificate verify ok.
*   Certificate level 0: Public key type RSA (2048/112 Bits/secBits), signed using sha256WithRSAEncryption
*   Certificate level 1: Public key type RSA (2048/112 Bits/secBits), signed using sha256WithRSAEncryption
* TLSv1.3 (IN), TLS alert, unknown (628):
* OpenSSL SSL_read: OpenSSL/3.0.13: error:0A00045C:SSL routines::tlsv13 alert certificate required, errno 0
* Failed receiving HTTP2 data: 56(Failure when receiving data from the peer)
* Connection #0 to host httpbin.example.com left intact
curl: (56) OpenSSL SSL_read: OpenSSL/3.0.13: error:0A00045C:SSL routines::tlsv13 alert certificate required, errno 0
```

4. Make the same request, but now with client certificates. It should now succeed.

```stdout
$ curl -v -H Host:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" --cacert certs/root/example.com.crt --cert certs/client/client.example.com.crt --key certs/client/client.example.com.key "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"
* Added httpbin.example.com:443:192.128.0.105 to DNS cache
* Hostname httpbin.example.com was found in DNS cache
*   Trying 192.128.0.105:443...
* Connected to httpbin.example.com (192.128.0.105) port 443
* ALPN: curl offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
*  CAfile: certs/root/example.com.crt
*  CApath: /etc/ssl/certs
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Request CERT (13):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (OUT), TLS handshake, Certificate (11):
* TLSv1.3 (OUT), TLS handshake, CERT verify (15):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384 / X25519 / RSASSA-PSS
* ALPN: server accepted h2
* Server certificate:
*  subject: CN=httpbin.example.com; O=httpbin organization
*  start date: Mar 21 16:46:26 2025 GMT
*  expire date: Mar 21 16:46:26 2026 GMT
*  common name: httpbin.example.com (matched)
*  issuer: O=example Inc.; CN=example.com
*  SSL certificate verify ok.
*   Certificate level 0: Public key type RSA (2048/112 Bits/secBits), signed using sha256WithRSAEncryption
*   Certificate level 1: Public key type RSA (2048/112 Bits/secBits), signed using sha256WithRSAEncryption
* using HTTP/2
* [HTTP/2] [1] OPENED stream for https://httpbin.example.com:443/status/418
* [HTTP/2] [1] [:method: GET]
* [HTTP/2] [1] [:scheme: https]
* [HTTP/2] [1] [:authority: httpbin.example.com]
* [HTTP/2] [1] [:path: /status/418]
* [HTTP/2] [1] [user-agent: curl/8.5.0]
* [HTTP/2] [1] [accept: */*]
> GET /status/418 HTTP/2
> Host:httpbin.example.com
> User-Agent: curl/8.5.0
> Accept: */*
>
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* old SSL session ID is stale, removing
< HTTP/2 418
< access-control-allow-credentials: true
< access-control-allow-origin: *
< content-type: text/plain; charset=utf-8
< x-more-info: http://tools.ietf.org/html/rfc2324
< date: Fri, 21 Mar 2025 17:43:16 GMT
< content-length: 13
< x-envoy-upstream-service-time: 1
< server: istio-envoy
<
* Connection #0 to host httpbin.example.com left intact
I'm a teapot!
```

---
