kubesec decrypt kubesec-prod-mystok-gcp-sealedsecret-cert.yaml | k apply -f -
kubesec decrypt kubesec-dev-mystok-gcp-sealedsecret-cert.yaml | k apply -f -

kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.12.6/controller.yaml

kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl apply -f argocd_application.yaml -n argocd
(@/overlays/prod/argocd)

kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
kubectl port-forward svc/argocd-server -n argocd 8080:443
kubectl port-forward svc/argocd-server -n argocd 8090:443


kustomize build . | k apply -f -
#######################
openssl req -x509 -nodes -newkey rsa:4096 -keyout "$PRIVATEKEY" -out "$PUBLICKEY" -subj "/CN=sealed-secret/O=sealed-secret"

kubectl -n "$NAMESPACE" create secret tls "$SECRETNAME" --cert="$PUBLICKEY" --key="$PRIVATEKEY"
kubectl -n "$NAMESPACE" label secret "$SECRETNAME" sealedsecrets.bitnami.com/sealed-secrets-key=active

kubectl -n  "$NAMESPACE" delete pod -l name=sealed-secrets-controller

kubectl -n "$NAMESPACE" logs -l name=sealed-secrets-controller

kubeseal --cert "./${PUBLICKEY}" --scope cluster-wide < mysecret.yaml | kubectl apply -f-

kubectl -n "$NAMESPACE" logs -l name=sealed-secrets-controller


######################
You are required to create a Master-to-Node firewall rule to allow GKE to communicate to the kubeseal container endpoint port tcp/8080.

CLUSTER_NAME=foo-cluster
gcloud config set compute/zone your-zone-or-region
Get the MASTER_IPV4_CIDR.

MASTER_IPV4_CIDR=$(gcloud container clusters describe $CLUSTER_NAME \
  | grep "masterIpv4CidrBlock: " \
  | awk '{print $2}')
Get the NETWORK.

NETWORK=$(gcloud container clusters describe $CLUSTER_NAME \
  | grep "^network: " \
  | awk '{print $2}')
Get the NETWORK_TARGET_TAG.

NETWORK_TARGET_TAG=$(gcloud compute firewall-rules list \
  --filter network=$NETWORK --format json \
  | jq ".[] | select(.name | contains(\"$CLUSTER_NAME\"))" \
  | jq -r '.targetTags[0]' | head -1)
Check the values.

echo $MASTER_IPV4_CIDR $NETWORK $NETWORK_TARGET_TAG

# example output
10.0.0.0/28 foo-network gke-foo-cluster-c1ecba83-node
Create the firewall rule.

gcloud compute firewall-rules create gke-to-kubeseal-8080 \
  --network "$NETWORK" \
  --allow "tcp:8080" \
  --source-ranges "$MASTER_IPV4_CIDR" \
  --target-tags "$NETWORK_TARGET_TAG" \
  --priority 1000

kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.12.6/controller.yaml

#########################
kubesec encrypt --key=gcp:<resource-id of Google Cloud KMS key> secret.yml > sealed-secret.yaml

kubesec decrypt secret.yaml

kubeseal のときは -o yaml しないとデフォでjson出力
