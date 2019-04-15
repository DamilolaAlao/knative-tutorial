# Configure Domain

We've been using Knative's default domain `example.com` so far which results in qualified domain names in the form of `{route}.{namespace}.{default-domain}` for our services. All our services are served behind the same external IP and we need to provide the host header in our `curl` commands for Knative to differentiate between different services:

```bash
curl -H "Host: helloworld-csharp.default.example.com" http://$ISTIO_INGRESS
```
It's possible to setup custom domains with Knative, so that you can have urls like `http://helloworld-go.default.mydomain.com` for your services. [Setting a custom domain](https://www.knative.dev/docs/serving/using-a-custom-domain/) page explains how. 

Even if you don't have a registered domain, it's still useful to setup a custom domain via a [NIP.IO](http://nip.io/). This would not only simplify our curl commands but it will also help for the next lab where we use a Knative service as a webhook to a third party service.

## NIP.IO

NIP.IO is a wildcard DNS matching service. It allows you have mappings like this:
* 10.0.0.1.nip.io maps to 10.0.0.1
* app.10.0.0.1.nip.io maps to 10.0.0.1
* customer1.app.10.0.0.1.nip.io maps to 10.0.0.1*

In our case, we can use the Istio ingress IP and let NIP.IO to map our services. For example, if your ingress IP is 1.2.3.4, then the following curl command would map to our `helloworld-csharp` service in `default` namespace:

```bash
curl http://helloworld-csharp.default.1.2.3.4.nip.io
```
## Edit domain configuration  

Edit the domain configuration:

```bash
kubectl edit cm config-domain -n knative-serving
```
Add your NIP domain (replace `1.2.3.4.` with the ingress IP) right before the `_example` line like this:

```yaml
apiVersion: v1
data:
  1.2.3.4.nip.io: ""
  _example: |
```
You can then check that the custom domain is applied:

```
kubectl get route helloworld-csharp --output jsonpath="{.status.domain}"
```

Set `ISTIO_INGRESS` if you haven't done so in the previous step:

```bash
export ISTIO_INGRESS=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

Finally, you can test that the domain works with curl:

```bash
curl http://helloworld-csharp.default.$ISTIO_INGRESS.nip.io
Hello C# Sample v1
```
## What's Next?
[Change configuration](03-changeconfig.md)