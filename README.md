# Install cert-manager with helm and automate the certificate issue and renewal process in a K8s Cluster

## Prerequisites

If you have already installed cert-manager following the previous article, then you cannot upgrade via helm, in that case, you need to delete the cert-manager deployment and all the TLS Secrets, Certificates, ClusterIssuers, and any other resources related to cert-manager.

## Steps

1. Add the Jetstack Helm repository
This repository is the only supported source of cert-manager charts. There are some other mirrors and copies across the internet, but those are entirely unofficial and could present a security risk.
Notably, the "Helm stable repository" version of cert-manager is deprecated and should not be used.

```
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

2. Install CustomResourceDefinitions
cert-manager requires a number of CRD resources, which can be installed manually using kubectl, or using the installCRDs option when installing the Helm chart.

```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.6.3/cert-manager.crds.yaml
```

3. Install cert-manager

```
helm install \
 cert-manager jetstack/cert-manager \
 --namespace cert-manager \
 --create-namespace \
 --version v1.6.3 \
 # --set installCRDs=true
```

4. Check pods
Once you’ve installed cert-manager, you can verify it is deployed correctly by checking the cert-manager namespace for running pods:

```
kubectl get po -n cert-manager
```

5. Configure the clusterissuer
Check if you already have a clusterissuer running on your cluster. if not, create the clusterissuer yaml and apply it on the cluster:

```
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: {email@address}
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
```

Apply it on the cluster:

```
kubectl apply -f cluster-issuer.yaml
```

Verify that the account has been registered successfully:

```
kubectl describe clusterissuer letsencrypt-prod
```

5. Modify your ingress resource files
Now, you need to modify the Ingress files, by adding the following in the annotations indent:

```
certmanager.k8s.io/cluster-issuer: letsencrypt-prod
kubernetes.io/tls-acme: "true"
```

Also, you need to have tls on the same ingress resource file:

```
tls:
  - hosts:
      - {domain-name}
    secretName: {secret-name}
```

Apply the ingress resource file and your certificates will be issued automatically. You can check that with:

```
kubectl get cert -n {namespace}
```

Note that now, we no longer need the certificate yaml file to apply it to get the certificate, it will be handled automatically by the ingress resource.

That’s it. Easy.


https://cert-manager.io/v1.6-docs/installation/helm/
