## Steps to set up Mastodon on Kubernetes

```bash
# Generate VAPID key and secret and add it to env
export $(docker run --rm tootsuite/mastodon bundle exec rake mastodon:webpush:generate_vapid_key)
```
```bash
# Set the credentials
kubectl create secret generic mastodon-creds \
  --from-literal=redis-password=redis \
  --from-literal=password=password \
  --from-literal=postgres-password=postgres \
  --from-literal=AWS_ACCESS_KEY_ID=mastodon-spaces \
  --from-literal=AWS_SECRET_ACCESS_KEY=<redacted> \
  --from-literal=SECRET_KEY_BASE=$(docker run --rm tootsuite/mastodon bundle exec rake secret) \
  --from-literal=OTP_SECRET=$(docker run --rm tootsuite/mastodon bundle exec rake secret) \
  --from-literal=VAPID_PRIVATE_KEY=$VAPID_PRIVATE_KEY \
  --from-literal=VAPID_PUBLIC_KEY=$VAPID_PUBLIC_KEY
```

```bash
# Install Mastodon
helm upgrade -i mastodon-on-doks ./ -f values.yaml

# Traefik install
kubectl create ns traefik
helm install --namespace=traefik traefik traefik/traefik

# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.9.1/cert-manager.yaml

# Set up clusterissuer, let's encrypt, cert-manager
kubectl create ns cert-manager 
kubectl apply -f dns/

## Post installation
# Access the postgres database
# kubectl exec -it <postgres pod name> -- psql -U <user> <databse_name>
kubectl exec -it mastodon-on-doks-postgresql-0 -- psql -U mastodon mastodon_production
```