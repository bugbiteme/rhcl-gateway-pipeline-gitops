# RHCL Gateway Pipeline GitOps

GitOps configuration for an Istio ingress gateway managed by [Kuadrant](https://kuadrant.io), deployed via OpenShift GitOps (ArgoCD).

Steps:

Preqeqs - Certificate issuer is set up with AWS secrets and  hosted zone ID

create external-secret management resources
(make sure they are set)
```bash
env | grep AWS  
```

```bash
oc apply -f argocd/app-external-secrets.yaml 
```

wait for arcgocd to fully sync `external-secrets`

Add AWS authentication secret to `secret-store` ns

```bash
oc create secret generic aws-credentials \
  --type=kuadrant.io/aws \
  --from-literal=AWS_ACCESS_KEY_ID=$KUADRANT_AWS_ACCESS_KEY_ID \
  --from-literal=AWS_SECRET_ACCESS_KEY=$KUADRANT_AWS_SECRET_ACCESS_KEY \
  -n secret-store
```
Create cluster issues (update hosted zone ID in values.yaml)


Create ingress-gateway with RHCL policies attached (DNS,TLS,Auth,RateLimit)

```bash
oc apply -f argocd/app-gw.yaml    
```

wait for arcgocd to fully sync `rhcl-gateway`

Create bookinfo sample app with RHCL piolicies attached to the httproute

```bash
oc apply -f argocd/app-bookinfo-demo.yaml 
```

Test the bookinfo HTTPRoute with auth and rate limiting

Get the hostname from the HTTPRoute:

```bash
export GATEWAY_URL=$(oc -n bookinfo get httproutes.gateway.networking.k8s.io bookinfo \
  -o jsonpath='{.spec.hostnames[0]}')

echo "Testing: https://${GATEWAY_URL}/api/v1/products/0/ratings"
```

**Valid user (alice) — authenticated, rate-limited to 2 req/10s:**
```bash
curl -k -so - -w "\n%{http_code}\n" \
  "https://${GATEWAY_URL}/api/v1/products/0/ratings" \
  -H 'Authorization: APIKEY IAMALICE'
```

**Valid user (bob) — authenticated, higher rate limit of 20 req/10s:**
```bash
curl -k -so - -w "\n%{http_code}\n" \
  "https://${GATEWAY_URL}/api/v1/products/0/ratings" \
  -H 'Authorization: APIKEY IAMBOB'
```

**Invalid key — expect `401 Unauthorized`:**
```bash
curl -k -so - -w "\n%{http_code}\n" \
  "https://${GATEWAY_URL}/api/v1/products/0/ratings" \
  -H 'Authorization: APIKEY NOONE'
```

Test RatelimitPolicy

expect
```
429
Too Many Requests
```

```bash
for i in {1..5}
do
curl -k -so - -w "\n%{http_code}\n" \
  "https://${GATEWAY_URL}/api/v1/products/0/ratings" \
  -H 'Authorization: APIKEY IAMALICE'
done
```
See no error codes
```bash
for i in {1..5}
do
curl -k -so - -w "\n%{http_code}\n" \
  "https://${GATEWAY_URL}/api/v1/products/0/ratings" \
  -H 'Authorization: APIKEY IAMBOB'
done
```

If things get stuck
kubectl patch namespace ingress-gateway -p '{"metadata":{"finalizers":null}}' --type merge

oc patch app rhcl-gateway -n openshift-gitops -p '{"metadata":{"finalizers":null}}' --type merge

flush cache (mac
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder)