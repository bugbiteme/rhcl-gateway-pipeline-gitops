**Secret for Kuadrant DNS integration (`ingress-gateway` namespace)**
(AWS Route 53 IAM Credentials)

   ```bash
   oc -n ingress-gateway create secret generic aws-credentials \
     --type=kuadrant.io/aws \
     --from-literal=AWS_ACCESS_KEY_ID=$KUADRANT_AWS_ACCESS_KEY_ID \
     --from-literal=AWS_SECRET_ACCESS_KEY=$KUADRANT_AWS_SECRET_ACCESS_KEY
   ```