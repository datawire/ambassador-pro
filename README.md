## Installation

1. Install Ambassador.
2. In the `auth.yaml`, configure the `AUTH0_DOMAIN` and `AUTH0_AUDIENCE` environment variables.
   * AUTH0_DOMAIN example is foo.auth0.com.
   * AUTH0_AUDIENCE is listed on the API page https://manage.auth0.com/#/apis
   * Configure the `namespace` fields appropriately.
3. Deploy the authentication service: `kubectl apply -f auth.yaml`.
4. Create the policy CRD: `kubectl apply -f policy-crd.yaml`.

## Quick Start / Example

Authentication policies are managed using the policy CRD. ** Note: The policy CRD is in alpha state, and the schema and fields are subject to change.**

In this quick start, we'll create a dummy service and manage access to the service.

1. Deploy the `backend` service: `kubectl apply -f backend.yaml`.
2. Deploy the example policy: `kubectl apply -f example-policy.yaml`.
3. Test public access to the backend service:

   ```
   curl http://$AMBASSADOR_IP/backend/public/
   ```

   You should get a bunch of JSON.

4. Test a private URL:

   ```
   curl http://$AMBASSADOR_IP/backend/private/
   ```

   You should get `{"message":"unauthorized"}`.

5. Get a JWT from Auth0. To do this, click on APIs, then the custom API you're using for the Ambassador Authentication service, and then the Test tab.

   ```
   curl --header 'authorization: Bearer eyeJdfasdf...' http://$AMBASSADOR_IP/backend/private/
   ```
  
   This should return a bunch of JSON again.

## Configuration

Ambassador Authentication supports creating arbitrary rules for authentication. By default, Ambassador Authentication returns unauthenticated requests. Each rule can have the following values:

| Value     | Example    | Description |
| -----     | -------    | -----------                  |
| `host`    | "*", "foo.com" | the Host that a given rule should match |
| `path`    | "/foo/url/"    | the URL path that a given rule should match to |
| `public`  | true           | a boolean that indicates whether or not authentication is required; default false |
| `scopes`  | "read:contacts" | the rights that need to be granted in a given API |

The following is a complete example of a policy:

```
apiVersion: stable.datawire.io/v1beta1
kind: Policy
metadata:
  name: example-policy
spec:
  # everything defaults to private, you can create rules to make stuff
  # public, and you can create rules to require additional scopes
  # which will be automatically checked
  rules:
   - host: "*"
     path: /backend/public/*
     public: true
   - host: "*"
     path: /
     public: true
   - host: "*"
     path: /backend/private/*
     public: false
   - host: "*"
     path: /backend/private-scoped/*
     scopes: admin
```