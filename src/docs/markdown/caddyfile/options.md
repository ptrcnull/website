---
title: Global options (Caddyfile)
---

<script>
$(function() {
	// We'll add links on the options in the code block at the top
	// to their associated anchor tags.
	let headers = $('article h5').map((i, el) => el.id.replace(/-/g, "_")).toArray();
	$('pre.chroma .k')
		.filter((k, item) => headers.includes(item.innerText))
		.map(function(k, item) {
			let text = item.innerText.replace(/</g,'&lt;').replace(/>/g,'&gt;');
			let url = '#' + item.innerText.replace(/_/g, "-");
			$(item).html('<a href="' + url + '" style="color: inherit;" title="' + text + '">' + text + '</a>');
		});
	$('pre.chroma .k:contains("servers")')
		.map(function(k, item) {
			let text = item.innerText.replace(/</g,'&lt;').replace(/>/g,'&gt;');
			$(item).html('<a href="#server-options" style="color: inherit;" title="Server Options">' + text + '</a>');
		});
});
</script>

# Global options

The Caddyfile has a way for you to specify options that apply globally. Some options act as default values, while others customize the behavior of the Caddyfile [adapter](/docs/config-adapters).

The very top of your Caddyfile can be a **global options block**. This is a block that has no keys:

```caddy
{
	...
}
```

There can only be one at most, and it must be the first block of the Caddyfile.

Possible options are:

```caddy
{
	# General Options
	debug
	http_port    <port>
	https_port   <port>
	default_bind <host>
	order <dir1> first|last|[before|after <dir2>]
	storage <module_name> {
		<options...>
	}
	storage_clean_interval <duration>
	renew_interval <duration>
	admin   off|<addr> {
		origins <origins...>
		enforce_origin
	}
	log [name] {
		output  <writer_module> ...
		format  <encoder_module> ...
		level   <level>
		include <namespaces...>
		exclude <namespaces...>
	}
	grace_period <duration>

	# TLS Options
	auto_https off|disable_redirects|ignore_loaded_certs|disable_certs
	email <yours>
	default_sni <name>
	local_certs
	skip_install_trust
	acme_ca <directory_url>
	acme_ca_root <pem_file>
	acme_eab <key_id> <mac_key>
	acme_dns <provider> ...
	on_demand_tls {
		ask      <endpoint>
		interval <duration>
		burst    <n>
	}
	key_type ed25519|p256|p384|rsa2048|rsa4096
	cert_issuer <name> ...
	ocsp_stapling off
	preferred_chains [smallest] {
		root_common_name <common_names...>
		any_common_name  <common_names...>
	}

	# Server Options
	servers [<listener_address>] {
		listener_wrappers {
			<listener_wrappers...>
		}
		timeouts {
			read_body   <duration>
			read_header <duration>
			write       <duration>
			idle        <duration>
		}
		max_header_size <size>
		log_credentials
		protocol {
			allow_h2c
			experimental_http3
			strict_sni_host [on|insecure_off]
		}
	}

	# PKI Options
	pki {
		ca [<id>] {
			name            <name>
			root_cn         <name>
			intermediate_cn <name>
			root {
				format <format>
				cert   <path>
				key    <path>
			}
			intermediate {
				format <format>
				cert   <path>
				key    <path>
			}
		}
	}
}
```


## General Options

##### `debug`
Enables debug mode, which sets the log level to `DEBUG` for the [default logger](#log). This reveals more details that can be useful when troubleshooting (and is very verbose in production). We ask that you enable this before asking for help on the [community forums](https://caddy.community). For example, at the top of your Caddyfile, if you have no other global options:

```caddy
{
	debug
}
```


##### `http_port`
The port for the server to use for HTTP. For internal use only; does not change the HTTP port for clients. Default: `80`


##### `https_port`
The port for the server to use for HTTPS. For internal use only; does not change the HTTPS port for clients. Default: `443`


##### `default_bind`
The default bind address to be used for all sites, if the [`bind` directive](/docs/caddyfile/directives/bind) is not used in the site. Default: empty, which binds to all interfaces.


##### `order`
Assigns an order to HTTP handler directive(s). As HTTP handlers execute in a sequential chain, it is necessary for the handlers to be executed in the right order. Standard directives have [a pre-defined order](/docs/caddyfile/directives#directive-order), but if using third-party HTTP handler modules, you'll need to define the order explicitly by either using this option or placing the directive in a [`route` block](/docs/caddyfile/directives/route). Ordering can be described absolutely (`first` or `last`), or relatively (`before` or `after`) to another directive.

For example, to use the [`replace-response` plugin](https://github.com/caddyserver/replace-response), you'd want to make sure its directive is ordered after `encode` so that it can perform replacements before the response is encoded (because responses flow up the handler chain, not down):

```caddy-d
order replace after encode
```


##### `storage`
Configures Caddy's storage mechanism. The default is [`file_system`](/docs/json/storage/file_system/). There are many other available [storage modules](/docs/json/storage/) provided as plugins.

For example, to change the file system's storage location:

```caddy-d
storage file_system /path/to/custom/location
```

Customizing the storage module is typically needed when syncing Caddy's storage across multiple instances of Caddy to make sure they all use the same certificates and keys. See the [Automatic HTTPS section on storage](/docs/automatic-https#storage) for more details.


##### `storage_clean_interval`
How often to scan storage units for old or expired assets and remove them. These scans exert lots of reads (and list operations) on the storage module, so choose a longer interval for large deployments. The value is a [duration value](/docs/conventions#durations). Default: `24h`.

Storage will always be cleaned when the process first starts. Then, a new cleaning will be started this duration after the previous cleaning started if the previous cleaning finished in less than half the time of this interval (otherwise next start will be skipped).


##### `renew_interval`
How often to scan all loaded, managed certificates for expiration, and trigger renewal if expired. Default: `10m`.


##### `admin`
Customizes the [admin API endpoint](/docs/api). If `off`, then the admin endpoint will be disabled. If disabled, config changes will be impossible without stopping and starting the server.

- **origins** configures the list of remotes/origins that are allowed to connect to the endpoint.

- **enforce_origin** enables enforcement of the Origin header. (This is different from enforcing origins generally, which is always done.)


##### `log`
Configures named loggers. The name can be passed to indicate a specific logger for which to customize the behavior. If no name is specified, the behavior of the `default` logger is modified. Multiple loggers with different names can be configured by using the `log` multiple times. You can read more about the `default` logger and an explanation of [how logging works in Caddy](/docs/logging).

The differs from the [`log` directive](/docs/caddyfile/directives/log), which only configures HTTP request logging (also known as access logs). The `log` global option shares its configuration structure with the directive (except for `include` and `exclude`), and complete documentation can be found on the directive's page.

- **output** configures where to write the logs. See the [`log` directive](/docs/caddyfile/directives/log#output-modules) for complete documentation.
- **format** describes how to encode, or format, the logs. See the [`log` directive](/docs/caddyfile/directives/log#format-modules) for complete documentation.
- **level** is the minimum entry level to log. Default: `INFO`.
- **include** specifies the log names to be included in this logger. For example, to include only logs emitted by the admin API, you would include `admin.api`.
- **exclude** specifies the log names to be excluded from this logger. For example, to exclude only HTTP access logs, you would exclude `http.log.access`.


##### `grace_period`
Defines the grace period for shutting down HTTP servers during config reloads. If clients do not finish their requests within the grace period, the server will be forcefully terminated to allow the reload to complete and free up resources. By default, no grace period is set.



## TLS Options

##### `auto_https`
Configure automatic HTTPS. There are a few modes to choose from:

- `off`: Disabled entirely. No certificate management or redirects.
- `disable_redirects`: Disable only HTTP-to-HTTPS redirects.
- `disable_certs`: Disable only certificate automation.
- `ignore_loaded_certs`: Automate certificates even for names which appear on manually-loaded certificates

See the [Automatic HTTPS](/docs/automatic-https) page for more details.


##### `email`
Your email address. Mainly used when creating an ACME account with your CA, and is highly recommended in case there are problems with your certificates.


##### `default_sni`
Sets a default TLS ServerName for when clients do not use SNI in their ClientHello.


##### `local_certs`
Causes all certificates to be issued internally by default, rather than through a (public) ACME CA such as Let's Encrypt. This is useful in development environments.


##### `skip_install_trust`
Skips the attempts to install the local CA's root into the system trust store, as well as into Java and Mozilla Firefox trust stores.


##### `acme_ca`
Specifies the URL to the ACME CA's directory. It is strongly recommended to set this to Let's Encrypt's [staging endpoint](https://letsencrypt.org/docs/staging-environment/) for testing or development. Default: ZeroSSL and Let's Encrypt's production endpoints.


##### `acme_ca_root`
Specifies a PEM file that contains a trusted root certificate for ACME CA endpoints, if not in the system trust store.


##### `acme_eab`
Specifies an External Account Binding to use for all ACME transactions.


##### `acme_dns`
Configures the ACME DNS challenge provider to use for all ACME transactions. The tokens following the name of the provider set up the provider the same as if specified in the [`tls` directive's `acme` issuer](/docs/caddyfile/directives/tls#acme).


##### `on_demand_tls`
Configures [On-Demand TLS](/docs/automatic-https#on-demand-tls) where it is enabled, but does not enable it (to enable it, use the [on_demand `tls` subdirective](/docs/caddyfile/directives/tls#syntax)). Highly recommended if using in production environments, to prevent abuse.

- **ask** will cause Caddy to make an HTTP request to the given URL with a query string of `?domain=` containing the value of the domain name. If the endpoint returns 200 OK, Caddy will be authorized to obtain a certificate for that name.

- **interval** and **burst** allows `<n>` certificate operations within `<duration>` interval.


##### `key_type`
Specifies the type of key to generate for TLS certificates; only change this if you have a specific need to customize it. The possible values are: `ed25519`, `p256`, `p384`, `rsa2048`, `rsa4096`.


##### `cert_issuer`
Defines the issuer (or source) of TLS certificates. The tokens following the name of the issuer set up the issuer the same as if specified in the [`tls` directive](/docs/caddyfile/directives/tls#issuer). May be repeated if you wish to configure more than one issuer to try. They will be tried in the order they are defined.


##### `ocsp_stapling`
Can be set to `off` to disable OCSP stapling. Useful in environments where responders are not reachable due to firewalls.


##### `preferred_chains`
If your CA provides multiple certificate chains, you can use this option to specify which chain Caddy should prefer. Set one of the following options:

- **smallest** will tell Caddy to prefer chains with the fewest amount of bytes.
- **root_common_name** is a list of one or more common names; Caddy will choose the first chain that has a root that matches with at least one of the specified common names.
- **any_common_name** is a list of one or more common names; Caddy will choose the first chain that has an issuer that matches with at least one of the specified common names.

Note that specifying `preferred_chains` as a global option will affect all issuers if there isn't any [overriding issuer level config](/docs/caddyfile/directives/tls#acme).



## Server Options

Customizes [HTTP servers](/docs/json/apps/http/servers/) with settings that potentially span multiple sites and thus can't be rightly configured in site blocks. These options affect the listener/socket, or other behavior beneath the HTTP layer.

Can be specified more than once, with different `listener_address` values, to configure different options per server. For example, `servers :443` will only apply to the server that is bound to the listener address `:443`. Omitting the listener address will apply the options to any remaining server.

<aside class="tip">
	Use the <a href="/docs/command-line#caddy-adapt"><code>caddy adapt</code></a> command to find the listen address for the servers in your Caddyfile.
</aside>

For example, to configure different options for the servers on port `:80` and `:443`, you would specify two `servers` blocks:

```caddy
{
	servers :443 {
		protocol {
			experimental_http3
		}
	}

	servers :80 {
		protocol {
			allow_h2c
		}
	}
}
```

##### `listener_wrappers`

Allows configuring [listener wrappers](/docs/json/apps/http/servers/listener_wrappers/), which can modify the behaviour of the base listener. They are applied in the given order.

There is a special no-op [`tls`](/docs/json/apps/http/servers/listener_wrappers/tls/) listener wrapper provided as a standard module which marks where TLS should be handled in the chain of listener wrappers. It should only be used if another listener wrapper must be placed in front of the TLS handshake.

The standard distribution of Caddy includes an [`http_redirect`](/docs/json/apps/http/servers/listener_wrappers/http_redirect/) listener wrapper, which can look at the first few bytes of an incoming request to determine if it's HTTP (instead of TLS), and trigger an HTTP->HTTPS redirect on the same port with the `https://` scheme. It must be placed _before_ the `tls` listener wrapper. For example:

```caddy-d
listener_wrappers {
	http_redirect
	tls
}
```

Another example, assuming you have the [`proxy_protocol`](/docs/json/apps/http/servers/listener_wrappers/proxy_protocol/) plugin installed, which must be used _before_ the `tls` listener wrapper:

```caddy-d
listener_wrappers {
	proxy_protocol {
		timeout 2s
		allow 192.168.86.1/24 192.168.86.1/24
	}
	tls
}
```


##### `timeouts`

- **read_body** is a [duration value](/docs/conventions#durations) that sets how long to allow a read from a client's upload. Setting this to a short, non-zero value can mitigate slowloris attacks, but may also affect legitimately slow clients. Defaults to no timeout.

- **read_header** is a [duration value](/docs/conventions#durations) that sets how long to allow a read from a client's request headers. Defaults to no timeout.

- **write** is a [duration value](/docs/conventions#durations) that sets how long to allow a write to a client. Note that setting this to a small value when serving large files may negatively affect legitimately slow clients. Defaults to no timeout.

- **idle** is a [duration value](/docs/conventions#durations) that sets the maximum time to wait for the next request when keep-alives are enabled. Defaults to 5 minutes to help avoid resource exhaustion.


##### `max_header_size`

The maximum size to parse from a client's HTTP request headers. It accepts all formats supported by [go-humanize](https://github.com/dustin/go-humanize/blob/master/bytes.go).


##### `log_credentials`

Since Caddy v2.5, by default, headers with potentially sensitive information (`Cookie`, `Set-Cookie`, `Authorization` and `Proxy-Authorization`) will be logged with empty values in access logs (see the [`log` directive](/docs/caddyfile/directives/log)).

If you wish to _not_ have these headers redacted, you may enable the `log_credentials` option.


##### `protocol`

- **allow_h2c** enables H2C ("Cleartext HTTP/2" or "H2 over TCP") support, which will serve HTTP/2 over plaintext TCP connections if a client support it. Because this is not implemented by the Go standard library, using H2C is incompatible with most of the other options for this server. Do not enable this only to achieve maximum client compatibility. In practice, very few clients implement H2C, and even fewer require it. This setting applies only to unencrypted HTTP listeners. ⚠️ Experimental feature; subject to change or removal.

- **experimental_http3** enables experimental draft HTTP/3 support. Note that HTTP/3 is not a finished spec and client support is extremely limited. This option will go away in the future. _This option is not subject to compatibility promises._

- **strict_sni_host** require that a request's `Host` header matches the value of the ServerName sent by the client's TLS ClientHello; often a necessary safeguard when using TLS client authentication. If there's a mismatch, an HTTP status `421 Misdirected Request` response is written to the client.
  
  This option will be implicitly turned on if [client authentication](/docs/caddyfile/directives/tls#client_auth) is configured. This disallows TLS client auth bypass (domain fronting) which could otherwise be exploited by sending an unprotected SNI value during a TLS handshake, then putting a protected domain in the Host header after establishing connection. This is a safe default, but you may explicitly turn it off with `insecure_off`, for example in the case of running a proxy where domain fronting is desired and access is not restricted based on hostname.



## PKI Options

The PKI (Public Key Infrastructure) app is the foundation for Caddy's [Local HTTPS](/docs/automatic-https#local-https) and [ACME server](/docs/caddyfile/directives/acme_server) features. The app defines certificate authorities (CAs) which are capable of signing certificates.

The default CA ID is `local`. If the ID is omitted when configuring the `ca`, then `local` is assumed.

##### `name`
The user-facing name of the certificate authority. Default: `Caddy Local Authority`

##### `root_cn`
The name to put in the CommonName field of the root certificate. Default: `{pki.ca.name} - {time.now.year} ECC Root`

##### `intermediate_cn`
The name to put in the CommonName field of the intermediate certificates. Default: `{pki.ca.name} - ECC Intermediate`

##### `root`
A key pair (certificate and private key) to use as the root for the CA. If not specified, one will be generated and managed automatically.

- **format** is the format in which the certificate and private key are provided. Currently, only `pem_file` is supported, which is the default, so this field is optional.
- **cert** is the certificate. This should be the path to a PEM file, when using `pem_file` format.
- **key** is the private key. This should be the path to a PEM file, when using `pem_file` format.

##### `intermediate`
A key pair (certificate and private key) to use as the intermediate for the CA. If not specified, one will be generated and managed automatically.

- **format** is the format in which the certificate and private key are provided. Currently, only `pem_file` is supported, which is the default, so this field is optional.
- **cert** is the certificate. This should be the path to a PEM file, when using `pem_file` format.
- **key** is the private key. This should be the path to a PEM file, when using `pem_file` format.
