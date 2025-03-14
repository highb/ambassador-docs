import Alert from '@material-ui/lab/Alert';

# The Module Resource

<div class="docs-article-toc">
<h3>Contents</h3>

* [Envoy](#envoy)
* [General](#general)
* [gRPC](#grpc)
* [Header behavior](#header-behavior)
* [Misc](#misc)
* [Observability](#observability)
* [Protocols](#protocols)
* [Security](#security)
* [Service health / timeouts](#service-health--timeouts)
* [Traffic management](#traffic-management)


</div>

If present, the Module defines system-wide configuration. This module can be applied to any Kubernetes service (the `ambassador` service itself is a common choice). **You may very well not need this Module.** To apply the Module to an Ambassador Service, it MUST be named `ambassador`, otherwise it will be ignored.  To create multiple `ambassador` Modules in the same namespace, they should be put in the annotations of each separate Ambassador Service.

The defaults in the Module are:

```yaml
apiVersion: getambassador.io/v2
kind:  Module
metadata:
  name:  ambassador
spec:
# Use ambassador_id only if you are using multiple instances of $productName$ in the same cluster.
# See below for more information.
  ambassador_id: "<ambassador_id>"
  config:
  # Use the items below for config fields
```

There are many config field items that can be configured on the Module, they are listed below with examples and grouped by category.

## Envoy

##### Envoy access logs

* `envoy_log_path` defines the path of log Envoy will use. By default this is standard output.
* `envoy_log_type` defines the type of log envoy will use, currently only support json or text.
* `envoy_log_format` defines the envoy log line format.

See the Envoy docs for a [complete list of operators](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/access_log) and [the standard log format](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#default-format-string).

These logs can be formatted using [Envoy operators](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#command-operators) to display specific information about an incoming request. The example below will show only the protocol and duration of a request:

```yaml
envoy_log_path: /dev/fd/1
envoy_log_type: json
envoy_log_format:
  {
    "protocol": "%PROTOCOL%",
    "duration": "%DURATION%"
  }
```

##### Error response overrides

Defines error response overrides for 4XX and 5XX response codes with `error_response_overrides`. By default, $productName$ will pass through error responses without modification, and errors generated locally will use Envoy's default response body, if any.

See [using error response overrides](../custom-error-responses) for usage details.

```yaml
error_response_overrides:
 - on_status_code: 404
   body:
     text_format: "File not found"
```

##### Forward client cert details
Add the `X-Forwarded-Client-Cert` header on upstream requests, which contains information about the TLS client certificate verified by $productName$.

See the Envoy documentation on [X-Forwarded-Client-Cert](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/headers.html?highlight=xfcc#x-forwarded-client-cert) and [SetCurrentClientCertDetails](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto.html#extensions-filters-network-http-connection-manager-v3-httpconnectionmanager-setcurrentclientcertdetails) for more information.

```yaml
forward_client_cert_details: true
```

##### Server name

By default Envoy sets server_name response header to `envoy`. Override it with this variable.

```yaml
server_name: envoy
```

##### Set current client cert details
Specify how to handle the `X-Forwarded-Client-Cert` header.

See the Envoy documentation on [X-Forwarded-Client-Cert](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/headers.html?highlight=xfcc#x-forwarded-client-cert) and [SetCurrentClientCertDetails](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto.html#enum-extensions-filters-network-http-connection-manager-v3-httpconnectionmanager-forwardclientcertdetails) for more information.

```yaml
set_current_client_cert_details: SANITIZE
```

##### Suppress Envoy headers

If true, $productName$ will not emit certain additional headers to HTTP requests and responses.

For the exact set of headers covered by this config, see the [Envoy documentation](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/router_filter#config-http-filters-router-headers-set)

```yaml
suppress_envoy_headers: true
```

##### Time to validate envoy configuration
Defines the timeout, in seconds, for validating a new Envoy configuration. The default is 10; a value of 0 disables Envoy configuration validation. Most installations will not need to use this setting.

```yaml
envoy_validation_timeout: 30
```

##### Content-Length headers
Allows Envoy to process requests/responses with both Content-Length and Transfer-Encoding headers set. By default such messages are rejected, but if option is enabled - Envoy will remove Content-Length header and process message. See the [Envoy documentation for more details](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/protocol.proto.html?highlight=allow_chunked_length#config-core-v3-http1protocoloptions)

```yaml
allow_chunked_length: true
```

---
## General

##### Ambassador ID

Use only if you are using multiple instances of $productName$ in the same cluster. See [this page](../running/#ambassador_id) for more information.

```yaml
ambassador_id: "<ambassador_id>"
```

##### Defaults

The `defaults` element is a dictionary of default values that will be applied to various $productName$ resources.

See [using defaults](../../using/defaults) for more information.


---

## gRPC

##### gRPC HTTP/1.1 bridge

Enable the gRPC-http11 bridge

```yaml
enable_grpc_http11_bridge: true
```

$productName$ supports bridging HTTP/1.1 clients to backend gRPC servers. When an HTTP/1.1 connection is opened and the request content type is `application/grpc`, $productName$ will buffer the response and translate into gRPC requests.

For more details on the translation process, see the [Envoy gRPC HTTP/1.1 bridge documentation](https://www.envoyproxy.io/docs/envoy/v1.11.2/configuration/http_filters/grpc_http1_bridge_filter.html). This setting can be enabled by setting `enable_grpc_http11_bridge: true`.

##### gRPC-Web

Enable the gRPC-Web protocol?

```yaml
enable_grpc_web: true
```

gRPC is a binary HTTP/2-based protocol. While this allows high performance, it is problematic for any programs that cannot speak raw HTTP/2 (such as JavaScript in a browser). gRPC-Web is a JSON and HTTP-based protocol that wraps around the plain gRPC to alleviate this problem and extend benefits of gRPC to the browser, at the cost of performance.

The gRPC-Web specification requires a server-side proxy to translate between gRPC-Web requests and gRPC backend services. $productName$ can serve as the service-side proxy for gRPC-Web when `enable_grpc_web: true` is set.

Find more on the [gRPC Web client GitHub repo](https://github.com/grpc/grpc-web).

##### Statistics

Enables telemetry of gRPC calls using Envoy's [gRPC Statistics Filter](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/grpc_stats_filter).

```yaml
---
apiVersion: getambassador.io/v2
kind:  Module
metadata:
  name: ambassador
spec:
  config:
    grpc_stats:
      upstream_stats: true
      services:
        - name: <package>.<service>
          method_names: [<method>]
```

Use the Envoy filter to enable telemetry of gRPC calls.

Supported parameters:
* `all_methods`
* `services`
* `upstream_stats`

Available metrics:
* `envoy_cluster_grpc_<service>_<status_code>`
* `envoy_cluster_grpc_<service>_request_message_count`
* `envoy_cluster_grpc_<service>_response_message_count`
* `envoy_cluster_grpc_<service>_success`
* `envoy_cluster_grpc_<service>_total`
* `envoy_cluster_grpc_upstream_<stats>` - **only when `upstream_stats: true`**

Please note that `<service>` will only be present if `all_methods` is set or the service and the method are present under `services`. If `all_methods` is false or the method is not on the list, the available metrics will be in the format `envoy_cluster_grpc_<stats>`.

* `all_methods`: If set to true, emit stats for all service/method names.
If set to false, emit stats for all service/message types to the same stats without including the service/method in the name.
**This option is only safe if all clients are trusted. If this option is enabled with untrusted clients, the clients could cause unbounded growth in the number
of stats in Envoy, using unbounded memory and potentially slowing down stats pipelines.**

* `services`: If set, specifies an allow list of service/methods that will have individual stats emitted for them. Any call that does not match the allow list will be counted in a stat with no method specifier (generic metric).

<Alert severity="warning">
  If both <code>all_methods</code> and <code>services</code> are present, <code>all_methods</code> will be ignored.
</Alert>

* `upstream_stats`: If true, the filter will gather a histogram for the request time of the upstream.

---

## Header behavior

##### Linkerd interoperability

When using Linkerd, requests going to an upstream service need to include the `l5d-dst-override` header to ensure that Linkerd will route them correctly. Setting `add_linkerd_headers` does this automatically.  See the [Mapping](../../using/mappings#linkerd-interoperability-add_linkerd_headers) documentation for more details.

```yaml
add_linkerd_headers: false
```

##### Header case

Enables upper casing of response headers by proper casing words: the first character and any character following a special character will be capitalized if it’s an alpha character. For example, “content-type” becomes “Content-Type”.

Please see the [Envoy documentation](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/protocol.proto.html#config-core-v3-http1protocoloptions-headerkeyformat)

```yaml
proper_case: false
```

##### Max request headers size

Sets the maximum allowed request header size in kilobytes. If not set, the default value from Envoy of 60 KB will be used.

See [Envoy documentation](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto.html) for more information.

```yaml
max_request_headers_kb: None
```

##### Overriding header case

Array of header names whose casing should be forced, both when proxied to upstream services and when returned downstream to clients. For every header that matches (case insensitively) to an element in this array, the resulting header name is forced to the provided casing in the array. Cannot be used together with 'proper_case'. This feature provides overrides for Envoy's normal [header casing rules](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/header_casing).

Enables overriding the case of response headers returned by $productName$. The `header_case_overrides` field is an array of header names. When $productName$ handles response headers that match any of these headers, matched case-insensitively, they will be rewritten to use their respective case-sensitive names. For example, the following configuration will force response headers that match `X-MY-Header` and `X-EXPERIMENTAL` to use that exact casing, regardless of the original case in the upstream response.

```yaml
header_case_overrides:
- X-MY-Header
- X-EXPERIMENTAL
```

If the upstream service responds with `x-my-header: 1`, Ambasasdor will return `X-MY-Header: 1` to the client. Similarly, if the upstream service responds with `X-Experimental: 1`, Ambasasdor will return `X-EXPERIMENTAL: 1` to the client. Finally, if the upstream service responds with a header for which there is no header case override, $productName$ will return the default, lowercase header.

This configuration is helpful when dealing with clients that are sensitive to specific HTTP header casing. In general, this configuration should be avoided, if possible, in favor of updating clients to work correctly with HTTP headers in a case-insensitive way.

##### Preserve external request ID
Controls whether to override the `X-REQUEST-ID` header or keep it as it is coming from incoming request. The default value is false.

```yaml
preserve_external_request_id: true
```

##### Prune unreachable routes
If true, routes with `:authority` matches will be removed from consideration for Hosts that don't match the `:authority` header. The default is false.

```yaml
prune_unreachable_routes: false
```

##### Strip matching host port
If true, $productName$ will strip the port from host/authority headers before processing and routing the request. This only applies if the port matches the underlying Envoy listener port.

```yaml
strip_matching_host_port: true
```



---

## Misc


##### Envoy's admin port

The port where $productName$'s Envoy will listen for low-level admin requests. You should almost never need to change this.

```yaml
admin_port: 8001
```

##### Lua scripts

Run a custom Lua script on every request. This is useful for simple use cases that mutate requests or responses, for example to add a custom header.

```yaml
lua_scripts: |
  function envoy_on_response(response_handle)
    response_handle:headers():add("Lua-Scripts-Enabled", "Processed")
  end
```

For more details on the Lua API, see the [Envoy Lua filter documentation](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/lua_filter.html).

Some caveats around the embedded scripts:

* They run in-process, so any bugs in your Lua script can break every request
* They're inlined in the $productName$ YAML, so it is recommended to not write complex logic in here
* They're run on every request/response to every URL

If you need more flexible and configurable options, $productName$ supports a [pluggable Filter system](/docs/edge-stack/latest/topics/using/filters/).

##### Merge slashes

If true, $productName$ will merge adjacent slashes for the purpose of route matching and request filtering. For example, a request for `//foo///bar` will be matched to a Mapping with prefix `/foo/bar`.

```yaml
merge_slashes: true
```
##### Modify Default Buffer Size

By default, the Envoy that ships with $productName$ uses a defailt of 1MiB soft limit for an upstream service's read and write buffer limits. This setting allows you to configure that buffer limit. See the [Envoy docs](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/cluster/v3/cluster.proto.html?highlight=per_connection_buffer_limit_bytes) for more information. 

```yaml
buffer_limit_bytes: 5242880 # Sets the default buffer limit to 5 MiB
```

##### Override default ports

If present, this sets the port $productName$ listens on for microservice access. If not present, $productName$ will use 8443 if TLS is enabled and 8080 if it is not.

```yaml
service_port: 1138
```

##### Regular expressions

**These features are deprecated.**

If `regex_type` is unset (the default), or is set to any value other than `unsafe`, $productName$ will use the [RE2](https://github.com/google/re2/wiki/Syntax) regular expression engine. This engine is designed to support most regular expressions, but keep bounds on execution time. **RE2 is the recommended regular expression engine.**

If `regex_type` is set to `unsafe`, $productName$ will use the [modified ECMAScript](https://en.cppreference.com/w/cpp/regex/ecmascript) regular expression engine. Please migrate your regular expressions to be compatible with RE2.

##### Use $productName$ namespace for service resolution

Controls whether $productName$ will resolve upstream services assuming they are in the same namespace as the element referring to them  For example, a Mapping in namespace `foo` will look for its service in namespace `foo`. If true, $productName$ will resolve the upstream services assuming they are in the same namespace as $productName$, unless the service explicitly mentions a different namespace.

```yaml
use_ambassador_namespace_for_service_resolution: false
```

---

## Observability

##### Diagnostics

Enable or disable the [Edge Policy Console](/docs/edge-stack/latest/topics/using/edge-policy-console/) and `/ambassador/v0/diag/` endpoints.

- Both $OSSproductName$ and $AESproductName$ provide low-level diagnostics at `/ambassador/v0/diag/`.
- $AESproductName$ also provides the higher-level Edge Policy Console at `/edge_stack/admin/`.

By default, both services are enabled.

Setting `diagnostics.enabled` to false will disable the routes for both services:

```yaml
diagnostics:
  enabled: false
```

With the routes disabled, `/ambassador/v0/diag` and `/edge_stack/admin/` will respond with 404 -- however, the services themselves are still running, and `/ambassador/v0/diag/` is reachable from inside the $productName$ Pod at `https://localhost:8877`. You can use Kubernetes port forwarding to set up remote access to the diagnostics page temporarily:

```
kubectl port-forward -n ambassador deploy/ambassador 8877
```

Alternately, you can expose the diagnostics page and Edge Policy Console but control them via `Host` based routing. Set `diagnostics.enabled` to false and create Mappings as specified in the [FAQ](../../../about/faq#how-do-i-disable-the-default-admin-mappings), and if exposing the diagnostics page, use `localhost:8877` as the `service` on the Mapping.

##### Diagnostics - allow non local

Whether or not to allow connections to the [Edge Policy Console](/docs/edge-stack/latest/topics/using/edge-policy-console/) and `/ambassador/v0/diag/` endpoints from any Pod in the entire cluster.

```yaml
diagnostics:
  allow_non_local: true
```

##### StatsD

Configures $productName$ statistics. These values can be set in the $productName$ module or in an environment variable.

For more information, see the [Statistics reference](../statistics#exposing-statistics-via-statsd).


---
## Protocols

##### Allow proxy protocol

Controls whether Envoy will honor the PROXY protocol on incoming requests.  Many load balancers can use the [PROXY protocol](https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt) to convey information about the connection they are proxying.

```yaml
use_proxy_proto: false
```

The default is false since the PROXY protocol is not compatible with HTTP.

##### Enable IPv4 and IPv6

Sets whether $productName$ should do IPv4 and/or IPv6 DNS lookups when contacting services. IPv4 defaults to true and IPv6 defaults to false. Either can be overridden in a [`Mapping`](../../using/mappings).

```yaml
enable_ipv4: true
enable_ipv6: false
```

If both IPv4 and IPv6 are enabled, $productName$ will prefer IPv6. This can have strange effects if $productName$ receives `AAAA` records from a DNS lookup, but the underlying network of the pod doesn't actually support IPv6 traffic. For this reason, the default is IPv4 only.

A Mapping can override both `enable_ipv4` and `enable_ipv6`, but if either is not stated explicitly in a Mapping, the values here are used. Most $productName$ installations will probably be able to avoid overriding these settings in Mappings.

##### HTTP/1.0 support

Enable or disable the handling of incoming HTTP/1.0 and HTTP 0.9 requests.

```yaml
enable_http10: true
```

---
## Security

##### Cross origin resource sharing (CORS)

Sets the default CORS configuration for all mappings in the cluster. See the [CORS syntax](../../using/cors).

```yaml
cors:
  origins: http://foo.example,http://bar.example
  methods: POST, GET, OPTIONS
  ...
```

##### IP allow and deny

Defines HTTP source IP address ranges to allow or deny.  Traffic not matching a range set to allow will be denied and vice versa. A list of ranges to allow and a separate list to deny may not both be specified.

Both take a list of IP address ranges with a keyword specifying how to interpret the address, for example:

```yaml
ip_allow:
- peer: 127.0.0.1
- remote: 99.99.0.0/16
```

The keyword `peer` specifies that the match should happen using the IP address of the other end of the network connection carrying the request: `X-Forwarded-For` and the `PROXY` protocol are both ignored. Here, our example specifies that connections originating from the $productName$ pod itself should always be allowed.

The keyword `remote` specifies that the match should happen using the IP address of the HTTP client, taking into account `X-Forwarded-For` and the `PROXY` protocol if they are allowed (if they are not allowed, or not present, the peer address will be used instead). This permits matches to behave correctly when, for example, $productName$ is behind a layer 7 load balancer. Here, our example specifies that HTTP clients from the IP address range `99.99.0.0` - `99.99.255.255` will be allowed.

You may specify as many ranges for each kind of keyword as desired.

##### Trust downstream client IP

Sets whether Envoy will trust the remote address of incoming connections or rely exclusively on the `X-Forwarded-For` header.

```yaml
use_remote_address: true
```

In $productName$ 0.50 and later, the default value for `use_remote_address` is set to true. When set to true, $productName$ will append to the `X-Forwarded-For` header its IP address so upstream clients of $productName$ can get the full set of IP addresses that have propagated a request.  You may also need to set `externalTrafficPolicy: Local` on your `LoadBalancer` as well to propagate the original source IP address.

See the [Envoy documentation on the `X-Forwarded-For header` ](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/headers) and the [Kubernetes documentation on preserving the client source IP](https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/#preserving-the-client-source-ip) for more details.

<Alert severity="warning">
  If you need to use <code>x_forwarded_proto_redirect</code>, you <strong>must</strong> set <code>use_remote_address</code> to false. Otherwise, unexpected behavior can occur.
</Alert>

##### `X_Forwarded_Proto` redirect

$productName$ lets through only the HTTP requests with `X-FORWARDED-PROTO: https` header set, and redirects all the other requests to HTTPS if this field is set to true. Note that `use_remote_address` must be set to false for this feature to work as expected.

```yaml
x_forwarded_proto_redirect: false
```

##### `X-Forwarded-For` trusted hops

Controls the how Envoy sets the trusted client IP address of a request. If you have a proxy in front of $productName$, Envoy will set the trusted client IP to the address of that proxy. To preserve the original client IP address, setting `x_num_trusted_hops: 1` will tell Envoy to use the client IP address in `X-Forwarded-For`.

Please see the [Envoy documentation](https://www.envoyproxy.io/docs/envoy/v1.11.2/configuration/http_conn_man/headers#x-forwarded-for) for more information.

```yaml
xff_num_trusted_hops: 1
```

The value of `xff_num_trusted_hops` indicates the number of trusted proxies in front of $productName$. The default setting is 0 which tells Envoy to use the immediate downstream connection's IP address as the trusted client address. The trusted client address is used to populate the `remote_address` field used for rate limiting and can affect which IP address Envoy will set as `X-Envoy-External-Address`.

`xff_num_trusted_hops` behavior is determined by the value of `use_remote_address` (which is true true by default).

* If `use_remote_address` is false and `xff_num_trusted_hops` is set to a value N that is greater than zero, the trusted client address is the (N+1)th address from the right end of XFF. (If the XFF contains fewer than N+1 addresses, Envoy falls back to using the immediate downstream connection’s source address as a trusted client address.)

* If `use_remote_address` is true and `xff_num_trusted_hops` is set to a value N that is greater than zero, the trusted client address is the Nth address from the right end of XFF. (If the XFF contains fewer than N addresses, Envoy falls back to using the immediate downstream connection’s source address as a trusted client address.)

Refer to [Envoy's documentation](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/headers.html#x-forwarded-for) for some detailed examples of this interaction.

<Alert severity="info">
  This value is not dynamically configurable in Envoy. A restart is required changing the value of <code>xff_num_trusted_hops</code> for Envoy to respect the change.
</Alert>

##### Rejecting Client Requests With Escaped Slashes

$productName$ can be configured to reject client requests that contain escaped slashes using the following configuration on the Ambassador Module:

```yaml
reject_requests_with_escaped_slashes: true
```

When set to true, $productName$ will reject client requests that contain escaped slashes by returning HTTP 400. These requests are
defined by containing `%2F`, `%2f`, `%5C` or `%5c` sequences in their URI path. By default, $productName$ will forward these requests unmodified.

###### Envoy and $productName$ behavior

Internally, Envoy treats escaped and unescaped slashes distinctly for matching purposes. This means that an $productName$ mapping
for path `/httpbin/status` will not be matched by a request for `/httpbin%2fstatus`.

On the other hand, when using $productName$, escaped slashes will be treated like unescaped slashes when applying FilterPolicies. For example, a request to `/httpbin%2fstatus/200` will be matched against a FilterPolicy for `/httpbin/status/*`.

###### Security Concern Example

With $productName$, this can become a security concern when combined with `bypass_auth` in the following scenario:
* Use a Mapping for path `/prefix` with `bypass_auth` set to true. The intention here is to apply no FilterPolicies under this prefix, by default.
* Use a Mapping for path `/prefix/secure/` without setting bypass_auth to true. The intention here is to selectively apply a FilterPolicy to this longer prefix.
* Have an upstream service that receives both `/prefix` and `/prefix/secure/` traffic (from the Mappings above), but the upstream service treats escaped and unescaped slashes equivalently.

In this scenario, when a client makes a request to `/prefix%2fsecure/secret.txt`, the underlying Envoy configuration will _not_ match the routing rule for `/prefix/secure/`, but will instead
match the routing rule for `/prefix` which has `bypass_auth` set to true. $productName$ FilterPolicies will _not_ be enforced in this case, and the upstream service will receive
a request that it treats equivalently to `/prefix/secure/secret.txt`, potentially leaking information that was assumed protected by an $productName$ FilterPolicy.

One way to avoid this particular scenario is to avoid using bypass_auth and instead use a FilterPolicy that applies no filters when no authorization behavior is desired.

The other way to avoid this scenario is to reject client requests with escaped slashes altogether to eliminate this class of path confusion security concerns. This is recommended when there is no known, legitimate reason to accept client requests that contain escaped slashes. This is especially true if it is not known whether upstream services will treat escaped and unescaped slashes equivalently.

This document is not intended to provide an exhaustive set of scenarios where path confusion can lead to security concerns. As part of good security practice it is recommended to audit end-to-end request flow and the behavior of each component’s escaped path handling to determine the best configuration for your use case.

###### Summary

Envoy treats escaped and unescaped slashes _distinctly_ for matching purposes. Matching is the underlying behavior used by $productName$ Mappings.

$productName$ treats escaped and unescaped slashes _equivalently_ when selecting FilterPolicies. FilterPolicies are applied by $productName$ after Envoy has performed route matching.

Finally, whether upstream services treat escaped and unescaped slashes equivalently is entirely dependent on the upstream service, and therefore dependent on your use case. Configuration intended to implement security policies will require audit with respect to escaped slashes. By setting reject_requests_with_escaped_slashes, this class of security concern can largely be eliminated.

---

## Service health / timeouts

##### Keepalive

Sets the global keepalive settings. $productName$ will use for all Mappings unless overridden on a Mapping's configuration. No default value is provided by $productName$.

More information at the [Envoy keepalive documentation](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/address.proto.html#config-core-v3-tcpkeepalive).

```yaml
keepalive:
  time: 2
  interval: 2
  probes: 100
```

##### Upstream idle timeout

Set the default upstream-connection idle timeout. Default is 1 hour.

```yaml
cluster_idle_timeout_ms: 3600000
```

If set, this specifies the timeout (in milliseconds) after which an idle connection upstream is closed. If disabled by setting it to 0, you risk upstream connections never getting closed due to idling if you do not set [`idle_timeout_ms` on each Mapping](../../using/timeouts/).

##### Upstream max lifetime

Set the default maximum upstream-connection lifetime. Default is 0 which means unlimited.

```yaml
cluster_max_connection_lifetime_ms: 10000
```

If set, this specifies the maximum amount of time (in milliseconds) after which an upstream connection is drained and closed, regardless of whether it is idle or not. Connection recreation incurs additional overhead when processing requests. The overhead tends to be nominal for plaintext (HTTP) connections within the same cluster, but may be more significant for secure HTTPS connections or upstreams with high latency. For this reason, it is generally recommended to set this value to at least 10000 ms to minimize the amortized cost of connection recreation while providing a reasonable bound for connection lifetime.

If not set (or set to zero), then upstream connections may remain open for arbitrarily long. This can be set on a per-Mapping basis by setting [`cluster_max_connection_lifetime_ms` on the Mapping](../../using/timeouts/).

##### Request timeout

Set the default end-to-end timeout for requests. Default is 3000 ms.

```yaml
cluster_request_timeout_ms: 3000
```

If set, this specifies the default end-to-end timeout for the requests. This can be set on a per-Mapping basis by setting [`timeout_ms` on the Mapping](../../using/timeouts/).

##### Retry policy

This lets you add resilience to your services in case of request failures by performing automatic retries.

```yaml
retry_policy:
  retry_on: "5xx"
```
##### Listener idle timeout

Controls how Envoy configures the tcp idle timeout on the http listener. Default is 1 hour.

```yaml
listener_idle_timeout_ms: 3600000
```

Controls how Envoy configures the TCP idle timeout on the HTTP listener. Default is no timeout (TCP connection may remain idle indefinitely). This is useful if you have proxies and/or firewalls in front of $productName$ and need to control how $productName$ initiates closing an idle TCP connection.

Please see the [Envoy documentation on HTTP protocol options](https://www.envoyproxy.io/docs/envoy/v1.12.2/api-v2/api/v2/core/protocol.proto#envoy-api-msg-core-httpprotocoloptions) for more information.

##### Readiness and liveness probes

The default liveness and readiness probes map `/ambassador/v0/check_alive` and `ambassador/v0/check_ready` internally to check Envoy itself.

```yaml
readiness_probe:
  enabled: true
liveness:
  enabled: true
```

You can change these to route requests to some other service. For example, to have the readiness probe map to the `quote` application's health check:

```yaml
readiness_probe:
  enabled: true
  service: quote
  rewrite: /backend/health
```

The liveness and readiness probes both support `prefix`, `rewrite`, and Module, with the same meanings as for [Mappings](../../using/mappings). Additionally, the `enabled` boolean may be set to false to disable API support for the probe.  It will, however, remain accessible on port 8877.

---

## Traffic management

##### Circuit breaking

Sets the global circuit breaking configuration that $productName$ will use for all Mappings, unless overridden in a Mapping.

More information at the [circuit breaking reference](../../using/circuit-breakers).

```yaml
circuit_breakers
  max_connections: 2048
  ...
```

##### Default label domain and labels

Set a default domain and request labels to every request for use by rate limiting.

For more on how to use these, see the [Rate Limit reference](../../using/rate-limits).

##### Load balancer

Sets the global load balancing type and policy that $productName$ will use for all mappings unless overridden in a mapping. Defaults to round robin with Kubernetes.

More information at the [load balancer reference](../load-balancer).

```yaml
load_balancer:
  policy: round_robin
```

