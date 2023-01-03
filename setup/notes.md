# Steps to set up Mastodon on DigitalOcean Kubernetes

##  Infrastructure setup using Terraform

TBD

##  Traefik install

```bash
kubectl create ns traefik
helm install --namespace=traefik traefik traefik/traefik
# Once the installation is complete copy the EXTERNAL_IP of the LoadBalancer as we need this to configure our DNS records
kubectl get services --namespace traefik traefik --output jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

Head to [DigitalOcean's Cloud Control Panel](https://cloud.digitalocean.com/networking/domains).

Add an A record <domain.com> that points to the IP address of the loadbalancer. Once configured, this can be verified using `dig`, a networking tool to query DNS recrods.

```bash
dig domain.com

Output:
;; ANSWER SECTION:
domain.com.	25	IN	A	<IP of the LoadBalancer>
```

Follow the SMTP Server service domain authentication steps

<insert blog or screenrecording>

```bash
# Generate VAPID key and secret and add it to env
export $(docker run --rm tootsuite/mastodon bundle exec rake mastodon:webpush:generate_vapid_key)
```
```bash
# Set the credentials
kubectl create secret generic mastodon-creds \
  --from-literal=redis-password=AVNS_4Gj-aBSM_8PqYrfWwct \
  --from-literal=password=AVNS_4Gj-aBSM_8PqYrfWwct \
  --from-literal=postgres-password=AVNS_9AJKJteXk-a-p50cb96 \
  --from-literal=AWS_ACCESS_KEY_ID=DO00K8N6XNT7PYFA4W2M \
  --from-literal=AWS_SECRET_ACCESS_KEY=gR1YrdnU5vtPlBYrg7sXUzWoeUURb7nLwHqv4XMVrXk \
  --from-literal=SECRET_KEY_BASE=$(docker run --rm tootsuite/mastodon bundle exec rake secret) \
  --from-literal=OTP_SECRET=$(docker run --rm tootsuite/mastodon bundle exec rake secret) \
  --from-literal=VAPID_PRIVATE_KEY=$VAPID_PRIVATE_KEY \
  --from-literal=VAPID_PUBLIC_KEY=$VAPID_PUBLIC_KEY
```


## Install cert-manager
kubectl create ns cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.9.1/cert-manager.yaml


```bash
# Install Mastodon
helm upgrade -i mastodon-on-doks ./ -f values.yaml


helm install mastodon-on-doks ./ \
  --namespace mastodon \
  --create-namespace \
  --timeout 10m0s \
  -f "values.yaml"

# Set up clusterissuer, let's encrypt, cert-manager
kubectl apply -f dns/

## Post installation
# Access the postgres database
# kubectl exec -it <postgres pod name> -- psql -U <user> <databse_name>
kubectl exec -it mastodon-on-doks-postgresql-0 -- psql -U mastodon mastodon_production
```