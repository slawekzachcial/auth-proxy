# Authentication Proxy

Example of how to use `oauth2-proxy` along with `ingress-nginx` to provide
single OpenID Connect authentication proxy for multiple applications having the
same parent DNS domain.

# Deployment

0. Prerequisites:
   * Working Docker installation
   * [Kind](https://kind.sigs.k8s.io/) installed
   * `/etc/hosts` updated with the following entries:
     ```
     127.0.0.1 auth.foo.org
     127.0.0.1 test-app.foo.org
     ```
   * (Optional; otherwise update app deployment image tag) Your application
     image published in the local registry e.g.: `localhost:5001/hello-httpd:latest`
1. Create Kubernetes cluster: `./create-kind-cluster.sh
2. Deploy [ingress-nginx](https://github.com/kubernetes/ingress-nginx): `kubectl apply -f ingress-nginx.yaml`
3. Deploy [oauth2-proxy](https://oauth2-proxy.github.io/oauth2-proxy/): `kubectl apply -f auth.yaml`
4. Deploy application: `kubectl apply -f app.yaml`

# Testing

With your browser access https://test-app.foo.org/. You will need to accept
few certificate warnings as the solution uses self-signed certificates.

You should be redirected to the configured OpenID Connect provider's sign-in page.
Once successufully authenticated, you should see "Hello World!" page.

# Cleanup

```bash
kind delete cluster
```
