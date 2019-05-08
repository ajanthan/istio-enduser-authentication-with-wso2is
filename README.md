# istio-enduser-authentication-with-wso2is
A guide on how to authenticate endusers in Istio using WSO2 Identity Server

### Prerequists
1. [Kubernetes](https://kubernetes.io/docs/setup/)
2. [WSO2 Identity Server on Kubernetes](https://medium.com/@balaajanthan/deploying-wso2-identity-server-in-kubernetes-d9320342806d)
3. [Istio](https://istio.io/docs/setup/kubernetes/install/kubernetes/)

### Deploying Sample(httpbin) Service
In this guide the official httpbin sample from Istio distribution is going to be secured with JWT. Deploy the sample by issuing following command from Istio installation directory.

```text
kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml)
```
Here assumption is the automatic sidecar injection is not used.

### Applying Istio Traffic Rule
An Istio gateway and a virtualservice is needed to be able to access the service from outside. Clone this repostitory and apply following policies.

```text
git clone https://github.com/ajanthan/istio-enduser-authentication-with-wso2is.git

cd istio-enduser-authentication-with-wso2is
kubectl apply -f httpbin-gateway.yaml
kubectl apply -f httpbin-virtualservice.yaml
```

### Applying End User Authentication Policy
Following JWT policy will configure Istio to secure the httbin service with JWT authentication from WSO2 Identity Server.

```yaml
apiVersion: "authentication.istio.io/v1alpha1"
kind: "Policy"
metadata:
  name: "jwt-example"
spec:
  targets:
  - name: httpbin
  origins:
  - jwt:
      issuer: "https://wso2is:9443/oauth2/token"
      jwksUri: "http://wso2is-service.default.svc.cluster.local:9763/oauth2/jwks"
```
To apply the policy issue following command.

```text
kubectl apply -f jwt-auth-policy.yaml
```

### Generating JWT Token From WSO2 Identity Server

Register a service provider with `OAuth/OpenID Connect Configuration` inbound authentication type and obtain `OAuth Client Key` and `OAuth Client Secret`.

In the next step the ID token is going to be generated using OAuth2 endpoint using `Client Credential` grant type.

```text
curl -vk -d "grant_type=password&username=admin&password=admin&scope=openid" -H "Authorization: Basic base64encode(OAuth Client Key:OAuth Client Secret)" -H "Content-Type: application/x-www-form-urlencoded" https://wso2is/oauth2/token
```
Get the `id_token` from the response to be used as the access token to access the httpbin service.

### Invoking the Service

[Determine the IP address and port of the Istio Gateway](https://istio.io/docs/tasks/traffic-management/ingress/#determining-the-ingress-ip-and-ports) and invoke the service as follows.

```text
curl -kv  http://$INGRESS_HOST:$INGRESS_PORT/headers -H "Authorization: Bearer <id_token>"
```
Without a valid `id_token` you will not be able to invoke the httbin service succcessfully.