# Authentication Proxy

Example of how to use `oauth2-proxy` along with `ingress-nginx` to provide
single OpenID Connect authentication proxy for multiple Kubernetes applications
having the same parent DNS domain.

# Deployment

0. Prerequisites:
   * Working Docker installation
   * [Kind](https://kind.sigs.k8s.io/) installed, or a working Kubernetes cluster
1. Copy `auth/env.sample` to `auth/.env` and update it accordingly
2. Create Kubernetes cluster: `./create-kind-cluster.sh` unless you already have a working cluster
3. Deploy [ingress-nginx](https://github.com/kubernetes/ingress-nginx): `kubectl apply -f ingress-nginx.yaml`
4. Deploy [oauth2-proxy](https://oauth2-proxy.github.io/oauth2-proxy/): `kubectl apply -k auth`
5. Deploy "app1" application: `kubectl apply -f app1.yaml`
6. Deploy "app2" and "app3": `seq 2 3 | xargs -I{} sed 's/app1/app{}/' app1.yaml | kubectl apply -f-`

# Testing

> Note:
>
> For the DNS names resolution to ingress configs available at 127.0.0.1 we use
> https://nip.io service and so DNS domain becomes `.7f000001.nip.io`.

With your browser access https://app1.7f000001.nip.io/. You will need to accept
few certificate warnings as the solution uses self-signed certificates.

You should be redirected to the configured OpenID Connect provider's sign-in page.
Once successufully authenticated, you should see a page similar to the one below:

![Page with headers](auth-proxy-request.png)

Notice `x-user`, `x-email` and `x-access-token` headers present in the request
sent to app1.7f000001.nip.io and populated by oauth2-proxy auth_request
integration.

For the application to be integrated with the authentication proxy, its ingress
needs the following annotations, as specified in [app1.yaml](app1.yaml):

```yaml
  annotations:
    nginx.ingress.kubernetes.io/auth-signin: "https://auth.7f000001.nip.io/oauth2/start?rd=$scheme%3A%2F%2F$host$escaped_request_uri"
    # For the auth_request URL use Kubernetes service URL to not leave the cluster and speed things up
    nginx.ingress.kubernetes.io/auth-url: "http://oauth2-proxy.default.svc.cluster.local:4180/oauth2/auth"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      auth_request_set $user   $upstream_http_x_auth_request_user;
      auth_request_set $email  $upstream_http_x_auth_request_email;
      auth_request_set $token  $upstream_http_x_auth_request_access_token;
      proxy_set_header X-User         $user;
      proxy_set_header X-Email        $email;
      proxy_set_header X-Access-Token $token;
```

Using your browser you should also be able to access https://app2.7f000001.nip.io
(accept the same certificate warnings).

`oauth2-proxy` is setup so that access to its auth request endpoint `/oauth2/auth`
can only be done from within the cluster. Assuming we trust what runs in our cluster
this will prevent the external applications from using our proxy with their Nginx
auth request.

To show how this works, let's simulate access by external application. We first
need to execute `kubectl edit ing app3` and replace

```yaml
    nginx.ingress.kubernetes.io/auth-url: "http://oauth2-proxy.default.svc.cluster.local:4180/oauth2/auth"
```

with

```yaml
    nginx.ingress.kubernetes.io/auth-url: https://auth.7f000001.nip.io/oauth2/auth
```

Now access with your browser https://app3.7f000001.nip.io. You'll see HTTP 500
error. That's because the auth request could not be completed as it is initiated
from outside of the cluster - comes through an ingress that rejects it.

# Cleanup

```bash
kind delete cluster
```
