# kubernetes-howtos
This repo is a collection of HOWTOs for various useful Kubernetes services/apps.  These were all tested on Cisco Container Platform "Version 2" clusters with Kubernetes 1.14.

### External DNS
Use helm to enable external-dns in your cluster.  This example uses the infoblox provider.
```
helm install --name my-external-dns \
--set provider=infoblox \
--set policy=sync \
--set txtOwnerId=<YOUR ID HERE> \
--set rbac.create=true \
--set infoblox.gridHost=<YOUR INFOBLOX SERVER> \
--set infoblox.wapiUsername=<YOUR INFOBLOX USER> \
--set infoblox.wapiPassword=<YOUR INFOBLOX PASSWD> \
--set infoblox.domainFilter=<YOUR DOMAIN HERE> \
--set infoblox.noSslVerify=true \
--set infoblox.wapiPort=443 \
--set infoblox.wapiVersion=2.3.1 \
stable/external-dns
```
Notes:
- `policy=sync` will keep the DNS entries in sync with the enabled apps in your cluster.  That is, when you delete a service from your cluster, the external-dns service will remove it from Infoblox.
- `txtOwnerId` sets a string identifier for your cluster.  The external-dns service will create a TXT record in DNS for every A record it creates and put this ID in the TXT entry.  That way it knows which entries it created and won't delete any entries that do not match this ID.
- `rbac.create=true` is required for clusters with RBAC enabled (i.e. Cisco Container Platform)
- `infoblox.*` are the required settings for the Infoblox provider.  Replace these as appropriate.

### ngnix Ingress Controller
Create an ingress controller.
```
helm install --name my-ingress \
--set controller.publishService.enabled=true \
--set controller.ingressClass=my-ingress \
stable/nginx-ingress
```

### LetsEncrypt
Install cert-manager.
```
kubectl apply \
--validate=false \
-f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.12/deploy/manifests/00-crds.yaml

helm install --name my-cert-manager \
--set ingressShim.defaultIssuerName=letsencrypt-prod \
--set ingressShim.defaultIssuerKind=Issuer \
jetstack/cert-manager
```

Modify the `issuer.yml` file in this repo and change the following:
- Replace `<your-email-here>` with your email address.  LetsEncrypt uses this to identify your certs as well as communicate with you if there is an issue fulfilling your certificate requests.
- Replace `<your-class-here1>` with the class you created for your ingress controller above, e.g. `my-ingress`.

Create the `letsencrypt-prod` and `letsencrypt-staging` issuers.
```
kubectl apply -f issuer.yml
```

### Jenkins
This example brings up a Jenkins server with automated external DNS record creation and LetsEncrypt certificate.
```
cat << EOF > values.yml
master:
  numExecutors: 2
  ingress:
    enabled: true
    hostName: jenkins.example.com
    annotations: {
      kubernetes.io/ingress.class: my-ingress,
      cert-manager.io/issuer: letsencrypt-prod
    }
    path: /
    tls: [
    {
      hosts: ['jenkins.example.com'],
      secretName: jenkins-tls
    }
    ]
EOF
helm install --name my-jenkins -f values.yml stable/jenkins
```