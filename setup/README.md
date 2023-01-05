# Steps to set up Mastodon on DigitalOcean Kubernetes

##  Infrastructure setup using Terraform



##  Traefik install

```bash
kubectl create ns traefik
helm install --namespace=traefik traefik traefik/traefik
# Once the installation is complete copy the EXTERNAL_IP of the LoadBalancer as we need this to configure our DNS records
kubectl get services --namespace traefik traefik --output jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

## Configure DNS records

- Head to [DigitalOcean's Cloud Control Panel](https://cloud.digitalocean.com/networking/domains).

- Add an A record <domain.com> that points to the IP address of the loadbalancer. Once configured, this can be verified using `dig`, a networking tool to query DNS recrods.
```bash
dig domain.com

Output:
;; ANSWER SECTION:
domain.com.	25	IN	A	<IP of the LoadBalancer>
```

- Follow the SMTP Server service domain authentication steps

<insert blog or screenrecording>


## Install cert-manager
```bash
kubectl create ns cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.9.1/cert-manager.yaml
```

## Add the secrets as required by Mastodon
```bash
# Generate VAPID key and secret and add it to env
export $(docker run --rm tootsuite/mastodon bundle exec rake mastodon:webpush:generate_vapid_key)
# Set the credentials
kubectl create ns mastodon && 
kubectl create -n mastodon secret generic mastodon-creds \
  --from-literal=redis-password=<redacted> \
  --from-literal=password=<redacted>  \
  --from-literal=postgres-password=<redacted>  \
  --from-literal=AWS_ACCESS_KEY_ID=<redacted>  \
  --from-literal=AWS_SECRET_ACCESS_KEY=<redacted>  \
  --from-literal=SECRET_KEY_BASE=$(docker run --rm tootsuite/mastodon bundle exec rake secret) \
  --from-literal=OTP_SECRET=$(docker run --rm tootsuite/mastodon bundle exec rake secret) \
  --from-literal=VAPID_PRIVATE_KEY=$VAPID_PRIVATE_KEY \
  --from-literal=VAPID_PUBLIC_KEY=$VAPID_PUBLIC_KEY
# mastodon-redis secret
kubectl create -n mastodon secret generic mastodon-redis \
--from-literal=redis-password=<redacted>
```

## Install Mastodon official chart
```bash
helm install mastodon-on-doks ./ \
  --namespace mastodon \
  --timeout 10m0s \
  -f "values.yaml"
 ```
{ or }
## Install Mastodon bitnami chart
```bash
MASTODON_HELM_CHART_VERSION="0.1.2"

helm install mastodon bitnami/mastodon \
  --version "$MASTODON_HELM_CHART_VERSION" \
  --namespace mastodon \
  --timeout 10m0s \
  -f "bitnami-values.yaml"
```

## Post chart installation steps
```bash
# Set up clusterissuer, ingress
kubectl apply -f dns/
```