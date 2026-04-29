Prereqs

Operators installed:
GitOps
RHCL

kuadrant-system is deployed

**Secret for Kuadrant DNS integration (`ingress-gateway` namespace)**
(AWS Route 53 IAM Credentials)

   ```bash
   oc -n ingress-gateway create secret generic aws-credentials \
     --type=kuadrant.io/aws \
     --from-literal=AWS_ACCESS_KEY_ID=$KUADRANT_AWS_ACCESS_KEY_ID \
     --from-literal=AWS_SECRET_ACCESS_KEY=$KUADRANT_AWS_SECRET_ACCESS_KEY
   ```

**Secret for cert-manager (same credentials, `cert-manager` namespace)**

   ```bash
   oc -n cert-manager create secret generic aws-credentials \
     --type=kuadrant.io/aws \
     --from-literal=AWS_ACCESS_KEY_ID=$KUADRANT_AWS_ACCESS_KEY_ID \
     --from-literal=AWS_SECRET_ACCESS_KEY=$KUADRANT_AWS_SECRET_ACCESS_KEY
   ```

   ClusterIssuer has been created for letsencrypt

to test success
```bash
export GATEWAY_IP=$(oc -n ingress-gateway get gateway prod-gateway -o jsonpath='{.status.addresses[*].value}')
curl -v http://$GATEWAY_IP  
``` 

or
```bash
curl -v -H "Host: demo.leonlevy.lol" http://$GATEWAY_IP
```

Expect a 404 , which means the gateway is reachable

```
Host GATEWAY_IP:80 was resolved.
* IPv6: (none)
* IPv4: 3.13.235.136, 3.132.179.205
*   Trying 3.13.235.136:80...
* Connected to  GATEWAY_IP (3.13.235.136) port 80
> GET / HTTP/1.1
> Host:  GATEWAY_IP
> User-Agent: curl/8.7.1
> Accept: */*
> 
* Request completely sent off
< HTTP/1.1 404 Not Found
< date: Wed, 29 Apr 2026 22:48:34 GMT
< server: istio-envoy
< content-length: 0
< 
* Connection #0 to host  GATEWAY_IP left intact

DNS Resolution

```bash
dig demo.leonlevy.lol
```