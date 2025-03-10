import Alert from '@material-ui/lab/Alert';

# The OAuth2 Filter

The OAuth2 filter type performs OAuth2 authorization against an identity provider implementing [OIDC Discovery](https://openid.net/specs/openid-connect-discovery-1_0.html). The filter is both:

* An OAuth Client, which fetches resources from the Resource Server on the user's behalf.
* Half of a Resource Server, validating the Access Token before allowing the request through to the upstream service, which implements the other half of the Resource Server.

This is different from most OAuth implementations where the Authorization Server and the Resource Server are in the same security domain. With the $productName$, the Client and the Resource Server are in the same security domain, and there is an independent Authorization Server.

## The $productName$ authentication flow

This is what the authentication process looks like at a high level when using $productName$ with an external identity provider. The use case is an end-user accessing a secured app service.

![$productName$ Authentication OAuth/OIDC](../../../../images/ambassador_oidc_flow.jpg)

### Some basic authentication terms

For those unfamiliar with authentication, here is a basic set of definitions.

* OpenID: is an [open standard](https://openid.net/) and [decentralized authentication protocol](https://en.wikipedia.org/wiki/OpenID). OpenID allows users to be authenticated by co-operating sites, referred to as "relying parties" (RP) using a third-party authentication service. End-users can create accounts by selecting an OpenID identity provider (such as Auth0, Okta, etc), and then use those accounts to sign onto any website that accepts OpenID authentication.
* Open Authorization (OAuth): an open standard for [token-based authentication and authorization](https://oauth.net/) on the Internet. OAuth provides to clients a "secure delegated access" to server or application resources on behalf of an owner, which means that although you won't manage a user's authentication credentials, you can specify what they can access within your application once they have been successfully authenticated. The current latest version of this standard is OAuth 2.0.
* Identity Provider (IdP): an entity that [creates, maintains, and manages identity information](https://en.wikipedia.org/wiki/Identity_provider) for user accounts (also referred to "principals") while providing authentication services to external applications (referred to as "relying parties") within a distributed network, such as the web.
* OpenID Connect (OIDC): is an [authentication layer that is built on top of OAuth 2.0](https://openid.net/connect/), which allows applications to verify the identity of an end-user based on the authentication performed by an IdP, using a well-specified RESTful HTTP API with JSON as a data format. Typically an OIDC implementation will allow you to obtain basic profile information for a user that successfully authenticates, which in turn can be used for implementing additional security measures like Role-based Access Control (RBAC).
* JSON Web Token (JWT): is a [JSON-based open standard for creating access tokens](https://jwt.io/), such as those generated from an OAuth authentication. JWTs are compact, web-safe (or URL-safe), and are often used in the context of implementing single sign-on (SSO) within federated applications and organizations. Additional profile information, claims, or role-based information can be added to a JWT, and the token can be passed from the edge of an application right through the application's service call stack.

If you look back at the authentication process diagram, the function of the entities involved should now be much clearer.

### Using an identity hub

Using an identity hub or broker allows you to support many IdPs without having to code individual integrations with them. For example, [Auth0](https://auth0.com/docs/identityproviders) and [Keycloak](https://www.keycloak.org/docs/latest/server_admin/index.html#social-identity-providers) both offer support for using Google and GitHub as an IdP.

An identity hub sits between your application and the IdP that authenticates your users, which not only adds a level of abstraction so that your application (and $productName$) is isolated from any changes to each provider's implementation, but it also allows your users to chose which provider they use to authenticate (and you can set a default, or restrict these options).

The Auth0 docs provide a guide for adding social IdP "[connections](https://auth0.com/docs/identityproviders)" to your Auth0 account, and the Keycloak docs provide a guide for adding social identity "[brokers](https://www.keycloak.org/docs/latest/server_admin/index.html#social-identity-providers)".


## OAuth2 global arguments

```yaml
---
apiVersion: getambassador.io/v2
kind: Filter
metadata:
  name: "example-oauth2-filter"
  namespace: "example-namespace"
spec:
  OAuth2:
    authorizationURL:       "url"      # required

    ############################################################################
    # OAuth Client settings                                                    #
    ############################################################################

    expirationSafetyMargin: "duration" # optional; default is "0"

    # Which settings exist depends on the grantType; supported grantTypes
    # are "AuthorizationCode", "Password", and "ClientCredentials".
    grantType:              "enum"     # optional; default is "AuthorizationCode"

    # How should $productName$ authenticate itself to the identity provider?
    clientAuthentication:              # optional
      method: "enum"                     # optional; default is "HeaderPassword"
      jwtAssertion:                      # optional if method method=="JWTAssertion"; forbidden otherwise
        setClientID:     bool              # optional; default is false
        # the following members of jwtAssertion only apply when the
        # grantType is NOT "ClientCredentials".
        audience:        "string"          # optional; default is to use the token endpoint from the authorization URL
        signingMethod:   "enum"            # optional; default is "RS256"
        lifetime:        "duration"        # optional; default is "1m"
        setNBF:          bool              # optional; default is false
        nbfSafetyMargin: "duration"        # optional; default is 0s
        setIAT:          bool              # optional; default is false
        otherClaims:                       # optional; default is {}
          "string": anything
        otherHeaderParameters:             # optional; default is {}
          "string": anything

    ## OAuth Client settings: grantType=="AuthorizationCode" ###################
    clientURL:              "string"   # deprecated; use 'protectedOrigins' instead
    protectedOrigins:                  # required; must have at least 1 item
    - origin: "url"                      # required
      internalOrigin: "url"              # optional; default is to just use the 'origin' field
      includeSubdomains: bool            # optional; default is false
    useSessionCookies:                 # optional; default is { value: false }
      value: bool                        # optional; default is true
      ifRequestHeader:                   # optional; default to apply "useSessionCookies.value" to all requests
        name: "string"                     # required
        negate: bool                       # optional; default is false
        # It is invalid to specify both "value" and "valueRegex".
        value: "string"                    # optional; default is any non-empty string
        valueRegex: "regex"                # optional; default is any non-empty string
    extraAuthorizationParameters:      # optional; default is {}
      "string": "string"

    ## OAuth Client settings: grantType=="AuthorizationCode" or "Password" #####
    clientID:               "string"   # required
    # The client secret can be specified by including the raw secret as a
    # string in "secret", or by referencing Kubernetes secret with
    # "secretName" (and "secretNamespace").  It is invalid to specify both
    # "secret" and "secretName".
    secret:                 "string"   # required (unless secretName is set)
    secretName:             "string"   # required (unless secret is set)
    secretNamespace:        "string"   # optional; default is the same namespace as the Filter

    ## OAuth Client settings (grantType=="ClientCredentials") ##################
    #
    # (there are no additional client settings for
    # grantType=="ClientCredentials")

    ############################################################################
    # OAuth Resource Server settings                                           #
    ############################################################################

    allowMalformedAccessToken: bool     # optional; default is false
    accessTokenValidation:     "enum"   # optional; default is "auto"
    accessTokenJWTFilter:               # optional; default is null
      name:                "string"       # required
      namespace:           "string"       # optional; default is the same namespace as the Filter
      inheritScopeArgument: bool          # optional; default is false
      stripInheritedScope:  bool          # optional; default is false
      arguments: JWT-Filter-Arguments     # optional
    injectRequestHeaders:               # optional; default is []
    - name:   "header-name-string"        # required
      value:  "go-template-string"        # required

    ############################################################################
    # HTTP client settings for talking with the identity provider              #
    ############################################################################

    insecureTLS:           bool       # optional; default is false
    renegotiateTLS:        "enum"     # optional; default is "never"
    maxStale:              "duration" # optional; default is "0"
```

### General settings

 - `authorizationURL`: Identifies where to look for the `/.well-known/openid-configuration` descriptor to figure out how to talk to the OAuth2 provider

### OAuth client settings

These settings configure the OAuth Client part of the filter.

 - `grantType`: Which type of OAuth 2.0 authorization grant to request from the identity provider.  Currently supported are:
   * `"AuthorizationCode"`: Authenticate by redirecting to a login page served by the identity provider.

   * `"ClientCredentials"`: Authenticate by requiring that the
     incoming HTTP request include as headers the credentials for
     $productName$ to use to authenticate to the identity provider.

     The type of credentials needing to be submitted depends on the
     `clientAuthentication.method` (below):
     + For `"HeaderPassword"` and `"BodyPassword"`, the headers
       `X-Ambassador-Client-ID` and `X-Ambassador-Client-Secret` must
       be set.
     + For `"JWTAssertion"`, the `X-Ambassador-Client-Assertion`
       header must be set to a JWT that is signed by your client
       secret, and conforms with the requirements in RFC 7521 section
       5.2 and RFC 7523 section 3, as well as any additional specified
       by your identity provider.

   * `"Password"`: Authenticate by requiring `X-Ambassador-Username` and `X-Ambassador-Password` on all
     incoming requests, and use them to authenticate with the identity provider using the OAuth2
     `Resource Owner Password Credentials` grant type.

 - `expirationSafetyMargin`: Check that access tokens not expire for
   at least this much longer; otherwise consider them to be already
   expired.  This provides a safety margin of time for your
   application to send it to an upstream Resource Server that grants
   insufficient leeway to account for clock skew and
   network/application latency.

 - `clientAuthentication`: Configures how $productName$ uses the
   `clientID` and `secret` to authenticate itself to the identity
   provider:
   * `method`: Which method $productName$ should use to authenticate
     itself to the identity provider.  Currently supported are:
     + `"HeaderPassword"`: Treat the client secret (below) as a
       password, and pack that in to an HTTP header for HTTP Basic
       authentication.
     + `"BodyPassword"`: Treat the client secret (below) as a
       password, and put that in the HTTP request bodies submitted to
       the identity provider.  This is NOT RECOMMENDED by RFC 6749,
       and should only be used when using HeaderPassword isn't
       possible.
     + `"JWTAssertion"`: Treat the client secret (below) as a
       password, and put that in the HTTP request bodies submitted to
       the identity provider.  This is NOT RECOMMENDED by RFC 6749,
       and should only be used when using HeaderPassword isn't
       possible.
   * `jwtAssertion`: Settings to use when `method: "JWTAssertion"`.
     + `setClientID`: Whether to set the Client ID as an HTTP
       parameter; setting it as an HTTP parameter is optional (per RFC
       7521 §4.2) because the Client ID is also contained in the JWT
       itself, but some identity providers document that they require
       it to also be set as an HTTP parameter anyway.
     + `audience` (only when `grantType` is not
       `"ClientCredentials"`): The audience value that your identity
       provider requires.
     + `signingMethod` (only when `grantType` is not
       `"ClientCredentials"`): The method to use to sign the JWT; how
       to interpret the `secret` (below).  Supported values are:
       - RSA: `"RS256"`, `"RS384"`, `"RS512"`: The secret must be a
         PEM-encoded RSA private key.
       - RSA-PSS: `"PS256"`, `"PS384"`, `"PS512"`: The secret must be
         a PEM-encoded RSA private key.
       - ECDSA: `"ES256"`, `"ES384"`, `"ES512"`: The secret must be a
         PEM-encoded Eliptic Curve private key.
       - HMAC-SHA: `"HS256"`, `"HS384"`, `"HS512"`: The secret is a
         raw string of bytes; it can contain anything.
     + `lifetime` (only when `grantType` is not
       `"ClientCredentials"`): The lifetime of the generated JWT; just
       enough time for the request to the identity provider to
       complete (plus possibly an extra allowance for clock skew).
     + `setNBF` (only when `grantType` is not `"ClientCredentials"`):
       Whether to set the optional "nbf" ("Not Before") claim in the
       generated JWT.
     + `nbfSafetyMargin` (only `setNBF` is true): The safety margin to
       build-in to the "nbf" claim, to allow for clock skew between
       ambassador and the identity provider.
     + `setIAT` (only when `grantType` is not `"ClientCredentials"`):
       Whether to set the optional "iat" ("Issued At") claim in the
       generated JWT.
     + `otherClaims` (only when `grantType` is not
       `"ClientCredentials"`): Any extra non-standard claims to
       include in the generated JWT.
     + `otherHeaderParameters` (only when `grantType` is not
       `"ClientCredentials"`): Any extra JWT header parameters to
       include in the generated JWT non-standard claims to include in
       the generated JWT; only the "typ" and "alg" header parameters
       are set by default.

Depending on which `grantType` is used, different settings exist.

Settings that are only valid when `grantType: "AuthorizationCode"` or `grantType: "Password"`:

 - `clientID`: The Client ID you get from your identity provider.
 - The client secret you get from your identity provider can be specified 2 different ways:
   * As a string, in the `secret` field.
   * As a Kubernetes `generic` Secret, named by `secretName`/`secretNamespace`.  The Kubernetes secret must of
     the `generic` type, with the value stored under the key`oauth2-client-secret`.  If `secretNamespace` is not given, it defaults to the namespace of the Filter resource.
   * **Note**: It is invalid to set both `secret` and `secretName`.

Settings that are only valid when `grantType: "AuthorizationCode"`:

 - `protectedOrigins`: (You determine these, and must register them
   with your identity provider) Identifies hostnames that can
   appropriately set cookies for the application.  Only the scheme
   (`https://`) and authority (`example.com:1234`) parts are used; the
   path part of the URL is ignored.

   You will need to register each origin in `protectedOrigins` as an
   authorized callback endpoint with your identity provider. The URL
   will look like
   `{{ORIGIN}}/.ambassador/oauth2/redirection-endpoint`.
   <!-- If you're looking at the above sentence and thinking "that's
   not correct!" (as I was): Yes, it's a lie that you need to register
   each one; you only need to register the first one, but support has
   the strong opinion that it's much simpler to just tell people
   register all of them.  Plus that gives us more flexibility for
   future changes.  So leave the lie.  -->

   If you provide more than one `protectedOrigin`, all share the same
   authentication system, so that logging into one origin logs you
   into all origins; to have multiple domains that have separate
   logins, use separate `Filter`s.

   + `internalOrigin`: This sub-field of `protectedOrigins[i]` allows
     you to tell $productName$ that there is another gateway in front of
     $productName that rewrites the `Host` header, so that on the
     internal network between that gateway and $productName$, the origin
     appears to be `internalOrigin` instead of `origin`.  As a
     special-case the scheme and/or authority of the `internalOrigin`
     may be `*`, which matches any scheme or any domain respectively.
     The `*` is most useful in configurations with exactly one
     protected origin; in such a configuration, $productName$ doesn't
     need to know what the origin looks like on the internal network,
     just that a gateway in front of $productName$ is rewriting it.  It
     is invalid to use `*` with `includeSubdomains: true`.

     For example, if you have a gateway in front of $productName$
     handling traffic for `myservice.example.com`, terminating TLS
     and routing that traffic to $productName$ with the name
     `ambassador.internal`, you might write:

         ```yaml
         protectedOrigins:
         - origin: https://myservice.example.com
           internalOrigin: http://ambassador.internal
         ```

     or, to avoid being fragile to renaming `ambassador.internal` to
     something else, since there are not multiple origins that the
     Filter must distinguish between, you could instead write:

         ```yaml
         protectedOrigins:
         - origin: https://myservice.example.com
           internalOrigin: "*://*"
         ```

 - `clientURL` is deprecated, and is equivalent to setting

   ```yaml
   protectedOrigins:
   - origin: clientURL-value
     internalOrigin: "*://*"
   ```

 - `extraAuthorizationParameters`: Extra (non-standard or extension) OAuth authorization parameters to use.  It is not valid to specify a parameter used by OAuth itself ("response_type", "client_id", "redirect_uri", "scope", or "state").

 - By default, any cookies set by the $productName$ will be
   set to expire when the session expires naturally.  The
   `useSessionCookies` setting may be used to cause session cookies to
   be used instead.

    * Normally cookies are set to be deleted at a specific time;
      session cookies are deleted whenever the user closes their web
      browser.  This may mean that the cookies are deleted sooner than
      normal if the user closes their web browser; conversely, it may
      mean that cookies persist for longer than normal if the use does
      not close their browser.
    * The cookies being deleted sooner may or may not affect
      user-perceived behavior, depending on the behavior of the
      identity provider.
    * Any cookies persisting longer will not affect behavior of the
      system; the $productName$ validates whether the session
      is expired when considering the cookie.

   If `useSessionCookies` is non-`null`, then:

    * By default it will have the cookies for all requests be
      session cookies or not according to the
      `useSessionCookies.value` sub-argument.

    * Setting the `useSessionCookies.ifRequestHeader` sub-argument
      tells it to use `useSessionCookies.value` for requests that
      match the condition, and `!useSessionCookies.value` for
      requests don't match.

      When determining if a request matches, it looks at the HTTP
      header field named by `useSessionCookies.ifRequestHeader.name`
      (case-insensitive), and checks if it is either set to (if
      `useSessionCookies.ifRequestHeader.negate: false`) or not set
      to (if `useSessionCookies.ifRequestHeader.negate: true`)...
       + a non-empty string (if neither
         `useSessionCookies.ifRequestHeader.value` nor
         `useSessionCookies.ifRequestHeader.valueRegex` are set)
       + the exact string `value` (case-sensitive) (if
         `useSessionCookies.ifRequestHeader.value` is set)
       + a string that matches the regular expression
         `useSessionCookies.ifRequestHeader.valueRegex` (if
         `valueRegex` is set).  This uses [RE2][] syntax (always, not
         obeying [`regex_type` in the `ambassador Module`][]) but does
         not support the `\C` escape sequence.
       + (it is invalid to have both `value` and `valueRegex` set)

### OAuth resource server settings

 - `allowMalformedAccessToken`: Allow any access token, even if they are not RFC 6750-compliant.
 - `injectRequestHeaders` injects HTTP header fields in to the request before sending it to the upstream service; where the header value can be set based on the JWT value.
 If an OAuth2 filter is chained with a JWT filter with `injectRequestHeaders` configured, both sets of headers will be injected.
 If the same header is injected in both filters, the OAuth2 filter will populate the value.
 The value is specified as a [Go `text/template`][] string, with the following data made available to it:

    * `.token.Raw` → `string` the access token raw JWT
    * `.token.Header` → `map[string]interface{}` the access token JWT header, as parsed JSON
    * `.token.Claims` → `map[string]interface{}` the access token JWT claims, as parsed JSON
    * `.token.Signature` → `string` the access token signature
    * `.idToken.Raw` → `string` the raw id token JWT
    * `.idToken.Header` → `map[string]interface{}` the id token JWT header, as parsed JSON
    * `.idToken.Claims` → `map[string]interface{}` the id token JWT claims, as parsed JSON
    * `.idToken.Signature` → `string` the id token signature
    * `.httpRequestHeader` → [`http.Header`][] a copy of the header of the incoming HTTP request.  Any changes to `.httpRequestHeader` (such as by using using `.httpRequestHeader.Set`) have no effect.  It is recommended to use `.httpRequestHeader.Get` instead of treating it as a map, in order to handle capitalization correctly.

 - `accessTokenValidation`: How to verify the liveness and scope of Access Tokens issued by the identity provider.  Valid values are either `"auto"`, `"jwt"`, or `"userinfo"`.  Empty or unset is equivalent to `"auto"`.
   * `"jwt"`: Validates the Access Token as a JWT.
     + By default: It accepts the RS256, RS384, or RS512 signature
       algorithms, and validates the signature against the JWKS from
       OIDC Discovery.  It then validates the `exp`, `iat`, `nbf`,
       `iss` (with the Issuer from OIDC Discovery), and `scope`
       claims: if present, none of the scope values are required to be
       present.  This relies on the identity provider using
       non-encrypted signed JWTs as Access Tokens, and configuring the
       signing appropriately
     + This behavior can be modified by delegating to [`JWT`
       Filter](../jwt/) with `accessTokenJWTFilter`:
       - `name` and `namespace` are used to identify which JWT Filter
         to use.  It is an error to point at a Filter that is not a
         JWT filter.
       - `arguments` is is the same as the `arguments` field when
         referring to a JWT Filter from a FilterPolicy.
       - `inheritScopeArgument` sets whether to inherit the `scope`
         argument from the FilterPolicy rule that triggered the OAuth2
         Filter (similarly special-casing the `offline_access` scope
         value); if the `arguments` field also specifies a `scope`
         argument, then the union of the two is used.
       - `stripInheritedScope` modifies the behavior of
         `inheritScopeArgument`.  Some identity providers use scope
         values that are URIs when speaking OAuth, but when encoding
         those scope values in to a JWT the provider strips the
         leading path of the value; removing everything up to and
         including the last "/" in the value.  Setting
         `stripInheritedScope` mimics this when passing the required
         scope to the JWT Filter.  It is meaningless to set
         `stripInheritedScope` if `inheritScopeArgument` is not set.
   * `"userinfo"`: Validates the access token by polling the OIDC UserInfo Endpoint. This means that the $productName$ must initiate an HTTP request to the identity provider for each authorized request to a protected resource.  This performs poorly, but functions properly with a wider range of identity providers.  It is not valid to set `accessTokenJWTFilter` if `accessTokenValidation: userinfo`.
   * `"auto"` attempts to do `"jwt"` validation if any of these
     conditions are true:

     + `accessTokenJWTFilter` is set, or
     + `grantType` is `"ClientCredentials"`, or
     + the Access Token parses as a JWT and the signature is valid,

     and otherwise falls back to `"userinfo"` validation.

[RE2]: https://github.com/google/re2/wiki/Syntax
[`regex_type` in the `ambassador Module`]: ../../../running/ambassador/#regular-expressions-regex_type

### HTTP client

These HTTP client settings are used for talking to the identity
provider:

 - `maxStale`: How long to keep stale cached OIDC replies for. This sets the `max-stale` Cache-Control directive on requests, and also ignores the `no-store` and `no-cache` Cache-Control directives on responses.  This is useful for maintaining good performance when working with identity providers with misconfigured Cache-Control.
 - `insecureTLS` disables TLS verification when speaking to an identity provider with an `https://` `authorizationURL`.  This is discouraged in favor of either using plain `http://` or [installing a self-signed certificate](../#installing-self-signed-certificates).
 - `renegotiateTLS` allows a remote server to request TLS renegotiation.  Accepted values are "never", "onceAsClient", and "freelyAsClient".

`"duration"` strings are parsed as a sequence of decimal numbers, each with optional fraction and a unit suffix, such as "300ms", "-1.5h" or "2h45m". Valid time units are "ns", "us" (or "µs"), "ms", "s", "m", "h".  See [Go `time.ParseDuration`](https://golang.org/pkg/time/#ParseDuration).

## OAuth2 path-specific arguments

```yaml
---
apiVersion: getambassador.io/v2
kind: FilterPolicy
metadata:
  name: "example-filter-policy"
  namespace: "example-namespace"
spec:
  rules:
  - host: "*"
    path: "*"
    filters:
    - name: "example-oauth2-filter"
      arguments:
        scope:                      # optional; default is ["openid"] for `grantType=="AuthorizationCode"`; [] for `grantType=="ClientCredentials"` and `grantType=="Password"`
        - "scopevalue1"
        - "scopevalue2"
        scopes:                     # deprecated; use 'scope' instead
        insteadOfRedirect:          # optional for "AuthorizationCode"; default is to do a redirect to the identity provider
          ifRequestHeader:            # optional; default is to return httpStatusCode for all requests that would redirect-to-identity-provider
            name: "string"              # required
            negate: bool                # optional; default is false
            # It is invalid to specify both "value" and "valueRegex".
            value: "string"             # optional; default is any non-empty string
            valueRegex: "regex"  # optional; default is any non-empty string
          # option 1:
          httpStatusCode: integer     # optional; default is 403 (unless `filters` is set)
          # option 2:
          filters:                    # optional; default is to use `httpStatusCode` instead
          - name: "string"              # required
            namespace: "string"         # optional; default is the same namespace as the FilterPolicy
            ifRequestHeader:            # optional; default to apply this filter to all requests matching the host & path
              name: "string"              # required
              negate: bool                # optional; default is false
              # It is invalid to specify both "value" and "valueRegex".
              value: "string"             # optional; default is any non-empty string
              valueRegex: "regex"         # optional; default is any non-empty string
            onDeny: "enum"              # optional; default is "break"
            onAllow: "enum"             # optional; default is "continue"
            arguments: DEPENDS          # optional
        sameSite: "enum"                # optional; the SameSite attribute to set on cookies created by this filter. valid values include: "lax", "strict", "none". by default, no SameSite attribute is set, which typically allows the browser to decide the value.
```

 - `scope`: A list of OAuth scope values to include in the scope of the authorization request.  If one of the scope values for a path is not granted, then access to that resource is forbidden; if the `scope` argument lists `foo`, but the authorization response from the provider does not include `foo` in the scope, then it will be taken to mean that the authorization server forbade access to this path, as the authenticated user does not have the `foo` resource scope.

   If `grantType: "AuthorizationCode"`, then the `openid` scope value is always included in the requested scope, even if it is not listed in the `FilterPolicy` argument.

   If `grantType: "ClientCredentials"` or `grantType: "Password"`, then the default scope is empty. If your identity provider does not have a default scope, then you will need to configure one here.

   As a special case, if the `offline_access` scope value is requested, but not included in the response then access is not forbidden. With many identity providers, requesting the `offline_access` scope is necessary to receive a Refresh Token.

   The ordering of scope values does not matter, and is ignored.

 - `scopes` is deprecated, and is equivalent to setting `scope`.

 - `insteadOfRedirect`: An action to perform instead of redirecting
   the User-Agent to the identity provider, when using `grantType: "AuthorizationCode"`.
   By default, if the User-Agent does not have a currently-authenticated session,
   then the $productName$ will redirect the User-Agent to the
   identity provider. Setting `insteadOfRedirect` allows you to modify
   this behavior. `insteadOfRedirect` does nothing when `grantType:
   "ClientCredentials"`, because the $productName$ will never
   redirect the User-Agent to the identity provider for the client
   credentials grant type.
    * If `insteadOfRedirect` is non-`null`, then by default it will
      apply to all requests that would cause the redirect; setting the
      `ifRequestHeader` sub-argument causes it to only apply to
      requests that have the HTTP header field
      `name` (case-insensitive) either set to (if `negate: false`) or
      not set to (if `negate: true`)
       + a non-empty string if neither `value` nor `valueRegex` are set
       + the exact string `value` (case-sensitive) (if `value` is set)
       + a string that matches the regular expression `valueRegex` (if
         `valueRegex` is set).  This uses [RE2][] syntax (always, not
         obeying [`regex_type` in the `ambassador Module`][]) but does
         not support the `\C` escape sequence.
    * By default, it serves an authorization-denied error page; by default HTTP 403 ("Forbidden"), but this can be configured by the `httpStatusCode` sub-argument.
    * Instead of serving that simple error page, it can instead be configured to call out to a list of other Filters, by setting the `filters` list. The syntax and semantics of this list are the same as `.spec.rules[].filters` in a [`FilterPolicy`](#filterpolicy-definition). Be aware that if one of these filters modify the request rather than returning a response, then the request will be allowed through to the backend service, even though the `OAuth2` Filter denied it.
    * It is invalid to specify both `httpStatusCode` and `filters`.

## XSRF protection

The `ambassador_xsrf.NAME.NAMESPACE` cookie is an opaque string that should be used as an XSRF token.  Applications wishing to leverage the $productName$ in their XSRF attack protection should take two extra steps:

 1. When generating an HTML form, the server should read the cookie, and include a `<input type="hidden" name="_xsrf" value="COOKIE_VALUE" />` element in the form.
 2. When handling submitted form data should verify that the form value and the cookie value match.  If they do not match, it should refuse to handle the request, and return an HTTP 4XX response.

Applications using request submission formats other than HTML forms should perform analogous steps of ensuring that the value is present in the request duplicated in the cookie and also in either the request body or secure header field.  A secure header field is one that is not `Cookie`, is not "[simple](https://www.w3.org/TR/cors/#simple-header)", and is not explicitly allowed by the CORS policy.


<Alert severity="info">
  Prior versions of the $productName$ did not have an <code>ambassador_xsrf.NAME.NAMESPACE</code> cookie, and instead required you to use the <code>ambassador_session.NAME.NAMESPACE</code> cookie.  The <code>ambassador_session.NAME.NAMESPACE</code> cookie should no longer be used for XSRF-protection purposes.
</Alert>

## RP-initiated logout

When a logout occurs, it is often not enough to delete the $productName$'s
session cookie or session data; after this happens, and the web
browser is redirected to the Identity Provider to re-log-in, the
Identity Provider may remember the previous login, and immediately
re-authorize the user; it would be like the logout never even
happened.

To solve this, the $productName$ can use [OpenID Connect Session
Management](https://openid.net/specs/openid-connect-session-1_0.html)
to perform an "RP-Initiated Logout", where $productName$
(the OpenID Connect "Relying Party" or "RP")
communicates directly with Identity Providers that support OpenID
Connect Session Management, to properly log out the user.
Unfortunately, many Identity Providers do not support OpenID Connect
Session Management.

This is done by having your application direct the web browser `POST`
*and navigate* to `/.ambassador/oauth2/logout`.  There are 2
form-encoded values that you need to include:

 1. `realm`: The `name.namespace` of the `Filter` that you want to log
    out of.  This may be submitted as part of the POST body, or may be set as a URL query parameter.
 2. `_xsrf`: The value of the `ambassador_xsrf.{{realm}}` cookie
    (where `{{realm}}` is as described above).  This must be set in the POST body, the URL query part will not be checked.

### Example configurations

```html
<form method="POST" action="/.ambassador/oauth2/logout" target="_blank">
  <input type="hidden" name="realm" value="myfilter.mynamespace" />
  <input type="hidden" name="_xsrf" value="{{ .Cookie.Value }}" />
  <input type="submit" value="Log out" />
</form>
```


```html
<form method="POST" action="/.ambassador/oauth2/logout?realm=myfilter.mynamespace" target="_blank">
  <input type="hidden" name="_xsrf" value="{{ .Cookie.Value }}" />
  <input type="submit" value="Log out" />
</form>
```

Using JavaScript:

```js
function getCookie(name) {
    var prefix = name + "=";
    var cookies = document.cookie.split(';');
    for (var i = 0; i < cookies.length; i++) {
        var cookie = cookies[i].trimStart();
        if (cookie.indexOf(prefix) == 0) {
            return cookie.slice(prefix.length);
        }
    }
    return "";
}

function logout(realm) {
    var form = document.createElement('form');
    form.method = 'post';
    form.action = '/.ambassador/oauth2/logout?realm='+realm;
    //form.target = '_blank'; // uncomment to open the identity provider's page in a new tab

    var xsrfInput = document.createElement('input');
    xsrfInput.type = 'hidden';
    xsrfInput.name = '_xsrf';
    xsrfInput.value = getCookie("ambassador_xsrf."+realm);
    form.appendChild(xsrfInput);

    document.body.appendChild(form);
    form.submit();
}
```

## Redis

The $productName$ relies on Redis to store short-lived authentication credentials and rate limiting information. If the Redis data store is lost, users will need to log back in and all existing rate-limits would be reset.

## Further reading

In this architecture, $productName$ is functioning as an Identity Aware Proxy in a Zero Trust Network. For more about this security architecture, read the [BeyondCorp security architecture whitepaper](https://ai.google/research/pubs/pub43231) by Google.

The ["How-to" section](../../../../howtos/) has detailed tutorials on integrating $productName$ with a number of Identity Providers.
