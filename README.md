# Ambassador Pro

Ambassador Pro is a set of paid add-on modules to the Ambassador open source API Gateway. The first functionality that will be available in Ambassador Pro is authentication. If you're interested in being an early adopter of Ambassador Pro, contact hello@datawire.io.

## Installation for Code Exchange Mode (no secret required)

Note: Ambassador Pro currently installs as an independent service. In the future, we expect to support a sidecar deployment model.

1. Install Ambassador.
2. Clone this repository; you'll need the YAML configuration.
   
   ```
   git clone https://github.com/datawire/ambassador-pro
   ```

3. In the `ambassador-pro.yaml`, configure the `AUTH_CALLBACK_URL`, `AUTH_DOMAIN`, `AUTH_AUDIENCE` and `AUTH_CLIENT_ID` environment variables based on your Auth0 configuration. (You'll need to create a custom API if you haven't already.)
   * The AUTH_DOMAIN is your Auth0 domain, e.g., foo.auth0.com.
   * AUTH_AUDIENCE is listed on the API page https://manage.auth0.com/#/apis
   * AUTH_CALLBACK_URL is the URL where you want to send users once they've authenticated.
   * AUTH_CLIENT_ID is the client ID of your application. 
   * Configure the `namespace` field appropriately for the `ClusterRoleBinding`, and is the namespace where your Ambassador and Ambassador Pro service is deployed.
4. Verify your Auth0 application:
   * Set Token Endpoint Authentication method to `None`
   * Add the value of AUTH_CALLBACK_URL above to the Allowed Callback URLs section
   * Add your domain to the Allowed Web Origins section
5. Deploy the authentication service: `kubectl apply -f ambassador-pro.yaml`.
6. Create the policy CRD: `kubectl apply -f policy-crd.yaml`.

## Installation for Client Credentials Mode (secret required)

Note: When `ambassador-pro.yaml` is deployed with the `AUTH_CLIENT_SECRET` environment variable, it will allow the app to talk directly to identity provider via the management api.

* Pros:
  * Facilitates the deployment by providing config validation on the start up.
  * No need for coding exchange even though it's still recommended to do.

* Cons:
  * The secret is in the manifest which makes it less secure.
  * In case of secret rotation, the app will need to be reconfigured. 
  * Extra setup is needed in the identity provider.

1. Install Ambassador.
2. Clone this repository; you'll need the YAML configuration.
   
   ```
   git clone https://github.com/datawire/ambassador-pro
   ```

3. In the `ambassador-pro.yaml`, configure the `AUTH_CALLBACK_URL`, `AUTH_DOMAIN`, `AUTH_AUDIENCE`, `AUTH_CLIENT_ID` and `AUTH_CLIENT_SECRET` environment variables based on your Auth0 configuration. (You'll need to create a custom API if you haven't already.)
   * The AUTH_DOMAIN is your Auth0 domain, e.g., foo.auth0.com.
   * AUTH_AUDIENCE is listed on the API page https://manage.auth0.com/#/apis
   * AUTH_CALLBACK_URL is the URL where you want to send users once they've authenticated.
   * AUTH_CLIENT_ID is the client ID of your application.
   * AUTH_CLIENT_SECRET is the client secret of your application. 
   * Configure the `namespace` field appropriately for the `ClusterRoleBinding`, and is the namespace where your Ambassador and Ambassador Pro service is deployed.
4. Verify your Auth0 application:
   * Set Token Endpoint Authentication method to `POST`
   * App is authorized to access Auth0 management api (APIs/Machine to Machine Applications) and following scopes have been granted:
      * read:clients
      * read:grants
   * App is set with the following grant types (Applications/Advanced Settings/Grant Types):
      * Authorization Code
      * Client Credentials   
   * Add the value of AUTH_CALLBACK_URL above to the Allowed Callback URLs section
   * Add your domain to the Allowed Web Origins section
5. Deploy the authentication service: `kubectl apply -f ambassador-pro.yaml`.
6. Create the policy CRD: `kubectl apply -f policy-crd.yaml`.

## Quick Start / Example

Authentication policies are managed using the `policy` CRD. Note that this CRD is in alpha state, and the schema is *subject to change*.

In this quick start, we'll create a route to the public httpbin.org service and manage access to the service.

1. Deploy the `httpbin` service: `kubectl apply -f httpbin.yaml`.
2. Verify that the httpbin service works correctly, e.g,

   ```
   $ curl http://$AMBASSADOR_IP/httpbin/ip
   {
    "origin": "35.205.31.151"
   }
   $ curl http://$AMBASSADOR_IP/httpbin/user-agent
   {
    "user-agent": "curl/7.54.0"
   }
   ```

3. Deploy the sample security policy: `kubectl apply -f httpbin-policy.yaml`. This policies gives public access to `/httpbin/ip` but restricts access to `/httpbin/user-agent`.
4. Test this out:

   ```
   $ curl http://$AMBASSADOR_IP/httpbin/ip
   {
    "origin": "35.205.31.151"
   }
   $ curl http://$AMBASSADOR_IP/httpbin/user-agent
   <a href="https://xxx.auth0.com/authorize?audience=https://xxx.auth0.com/api/v2/&amp;response_type=code&amp;redirect_uri=http://35.226.13.0/callback&amp;client_id=Z6m3lwCot6GaThT4L142nkOKNPeDe87n&amp;state=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1MzY2OTQ2MjglhdCI6MUzNjY5NDMyOCwianRpIjoiN2FjOThjZTQtYjdjZi00NTU3LTlkYTEtZGJjNzZjYzNjZjg4IiwibmJmIjowLCJwYXRoIjoiL2h0dHBiaW4vdXNi1hZ2VudCJ9.NtBA5deqPn5XI7vonca4tpgYNrM-212TiQhTZ_KzWos&amp;scope=offline_access openid profile">See Other</a>
   ```

5. In your browser, visit http://$AMBASSADOR_IP/httpbin/ip, and see the IP address.

6. In your browser, visit http://$AMBASSADOR_IP/httpbin/user-agent. You'll be redirected to a log in page. Authenticate, and you should be able to get the user-agent.

7. If you want to test with a JWT, you can get a JWT from Auth0. To do this, click on APIs, then the custom API you're using for the Ambassador Authentication service, and then the Test tab. Pass the JWT in the `authorization: Bearer` HTTP header:

   ```
   $ curl --header 'authorization: Bearer eyeJdfasdf...' http://$AMBASSADOR_IP/httpbin/user-agent
   {
     "user-agent": "curl/7.54.0"
   }
   ```

## Configuration

Ambassador Authentication supports creating arbitrary rules for authentication. By default, Ambassador Authentication returns unauthenticated requests. Each rule can have the following values:

| Value     | Example    | Description |
| -----     | -------    | -----------                  |
| `host`    | "*", "foo.com" | the Host that a given rule should match |
| `path`    | "/foo/url/"    | the URL path that a given rule should match to |
| `public`  | true           | a boolean that indicates whether or not authentication is required; default false |
| `scopes`  | "read:test" | the rights that need to be granted in a given API |

The following is a complete example of a policy:

```
apiVersion: stable.datawire.io/v1beta1
kind: Policy
metadata:
  name: httpbin-policy
spec:
  # everything defaults to private; you can create rules to make stuff
  # public, and you can create rules to require additional scopes
  # which will be automatically checked
  rules:
   - host: "*"
     path: /httpbin/ip
     public: true
   - host: "*"
     path: /httpbin/user-agent/*
     public: false
   - host: "*"
     path: /httpbin/headers/*
     scopes: "read:test"
```