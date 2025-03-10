---
title: tls (Caddyfile directive)
---

<script>
$(function() {
	// We'll add links to all the subdirectives if a matching anchor tag is found on the page.
	addLinksToSubdirectives();
});
</script>

# tls

Configures TLS for the site.

**Caddy's default TLS settings are secure. Only change these settings if you have a good reason and understand the implications.** The most common use of this directive will be to specify an ACME account email address, change the ACME CA endpoint, or to provide your own certificates.

Compatibility note: Due to its sensitive nature as a security protocol, deliberate adjustments to TLS defaults may be made in new minor or patch releases. Old or broken TLS versions, ciphers, features, etc. may be removed at any time. If your deployment is extremely sensitive to changes, you should explicitly specify those values which must remain constant, and be vigilant about upgrades. In almost every case, we recommend using the default settings.


## Syntax

```caddy-d
tls [internal|<email>] | [<cert_file> <key_file>] {
	protocols <min> [<max>]
	ciphers   <cipher_suites...>
	curves    <curves...>
	alpn      <values...>
	load      <paths...>
	ca        <ca_dir_url>
	ca_root   <pem_file>
	key_type  ed25519|p256|p384|rsa2048|rsa4096
	dns       <provider_name> [<params...>]
	dns_challenge_override_domain <domain>
	resolvers <dns_servers...>
	eab       <key_id> <mac_key>
	on_demand
	client_auth {
		mode                   [request|require|verify_if_given|require_and_verify]
		trusted_ca_cert        <base64_der>
		trusted_ca_cert_file   <filename>
		trusted_leaf_cert      <base64_der>
		trusted_leaf_cert_file <filename>
	}
	issuer          <issuer_name>  [<params...>]
	get_certificate <manager_name> [<params...>]
}
```

- **internal** means to use Caddy's internal, locally-trusted CA to produce certificates for this site.
- **&lt;email&gt;** is the email address to use for the ACME account managing the site's certificates.
- **&lt;cert_file&gt;** and **&lt;key_file&gt;** are the paths to the certificate and private key PEM files. Specifying just one is invalid.
- **protocols** <span id="protocols"/> specifies the minimum and maximum protocol versions. Default min: `tls1.2`. Default max: `tls1.3`
- **ciphers** <span id="ciphers"/> specifies the list of cipher suite names in descending preference order. It is recommended to not change these unless you know what you're doing. Note that cipher suites are not customizable for TLS 1.3; and not all TLS 1.2 ciphers are enabled by default. The supported names are (in no particular order here):
	- TLS_RSA_WITH_3DES_EDE_CBC_SHA
	- TLS_RSA_WITH_AES_128_CBC_SHA
	- TLS_RSA_WITH_AES_256_CBC_SHA
	- TLS_RSA_WITH_AES_128_GCM_SHA256
	- TLS_RSA_WITH_AES_256_GCM_SHA384
	- TLS_AES_128_GCM_SHA256
	- TLS_AES_256_GCM_SHA384
	- TLS_CHACHA20_POLY1305_SHA256
	- TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA
	- TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA
	- TLS_ECDHE_RSA_WITH_3DES_EDE_CBC_SHA
	- TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA
	- TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA
	- TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
	- TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
	- TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
	- TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
	- TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256
	- TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256
- **curves** <span id="curves"/> specifies the list of EC curves to support. It is recommended to not change these. Supported values are:
	- x25519
	- secp256r1
	- secp384r1
	- secp521r1
- **alpn** <span id="alpn"/> is the list of values to advertise in the ALPN extension of the TLS handshake.
- **load** <span id="load"/> specifies a list of folders from which to load PEM files that are certificate+key bundles.
- **ca** <span id="ca"/> changes the ACME CA endpoint. This is most often used to set [Let's Encrypt's staging endpoint](https://letsencrypt.org/docs/staging-environment/) when testing, or an internal ACME server. (To change this value for the whole Caddyfile, use the `acme_ca` [global option](/docs/caddyfile/options) instead.)
- **ca_root** <span id="ca_root"/> specifies a PEM file that contains a trusted root certificate for the ACME CA endpoint, if not in the system trust store.
- **key_type** <span id="key_type"/> is the type of key to use when generating CSRs. Only set this if you have a specific requirement.
- **dns** <span id="dns"/> enables the [DNS challenge](/docs/automatic-https#dns-challenge) using the specified provider plugin, which must be plugged in from one of the [`caddy-dns`](https://github.com/caddy-dns) repositories. Each provider plugin may have their own syntax following their name; refer to their docs for details. Maintaining support for each DNS provider is a community effort. [Learn how to enable the DNS challenge for your provider at our wiki.](https://caddy.community/t/how-to-use-dns-provider-modules-in-caddy-2/8148)
- **dns_challenge_override_domain** <span id="dns_challenge_override_domain"/> overrides the domain to use for the DNS challenge. This is to delegate the challenge to a different domain, e.g. one whose DNS provider has a [`caddy-dns`](https://github.com/caddy-dns) plugin.
- **resolvers** <span id="resolvers"/> customizes the DNS resolvers used when performing the DNS challenge; these take precedence over system resolvers or any default ones. If set here, the resolvers will propagate to all configured certificate issuers.
- **eab** <span id="eab"/> configures ACME external account binding (EAB) for this site, using the key ID and MAC key provided by your CA.
- **on_demand** <span id="on_demand"/> enables [on-demand TLS](/docs/automatic-https#on-demand-tls) for the hostnames given in the site block's address(es). **Security warning:** Doing so in production is insecure unless you also configure the [`on_demand_tls` global option](https://caddyserver.com/docs/caddyfile/options#on-demand-tls) to mitigate abuse.
- **client_auth** <span id="client_auth"/> enables and configures TLS client authentication:
	- **mode** <span id="mode"/> is the mode for authenticating the client. Allowed values are:

		| Mode               | Description                                                                              |
		|--------------------|------------------------------------------------------------------------------------------|
		| request            | Ask clients for a certificate, but allow even if there isn't one; do not verify it       |
		| require            | Require clients to present a certificate, but do not verify it                           |
		| verify_if_given    | Ask clients for a certificate; allow even if there isn't one, but verify it if there is  |
		| require_and_verify | Require clients to present a valid certificate that is verified                          |

	Default: `require_and_verify` if any `trusted_ca_cert` or `trusted_leaf_cert` are provided; otherwise, `require`.
	
	- **trusted_ca_cert** <span id="trusted_ca_cert"/> is a base64 DER-encoded CA certificate against which to validate client certificates.
	- **trusted_ca_cert_file** <span id="trusted_ca_cert_file"/> is a path to a PEM CA certificate file against which to validate client certificates.
	- **trusted_leaf_cert** <span id="trusted_leaf_cert"/> is a base64 DER-encoded client leaf certificate to accept.
	- **trusted_leaf_cert_file** <span id="trusted_leaf_cert_file"/> is a path to a PEM CA certificate file against which to validate client certificates.

	Multiple `trusted_*` directives may be used to specify multiple CA or leaf certificates. Client certificates which are not listed as one of the leaf certificates or signed by any of the specified CAs will be rejected according to the **mode**.

- **issuer** <span id="issuer"/> configures a custom certificate issuer, or a source from which to obtain certificates. Which issuer is used and the options that follow in this segment depend on the issuer modules that are available (see below for the standard issuers; plugins may add others). Some of the other subdirectives such as `ca` and `dns` are actually shortcuts for configuring the `acme` issuer (and this subdirective was added later), so specifying this directive and some of the others is confusing and thus prohibited. This subdirective can be specified multiple times to configure multiple, redundant issuers; if one fails to issue a cert, the next one will be tried.
- **get_certificate** <span id="get_certificate"/> enables getting certificates from a _manager module_ at handshake-time. [See below for standard certificate manager modules.](#certificate-managers)

### Issuers

These issuers come standard with the `tls` directive:

#### acme

Obtains certificates using the ACME protocol.

```caddy
... acme [<directory_url>] {
	dir      <directory_url>
	test_dir <test_directory_url>
	email    <email>
	timeout  <duration>
	disable_http_challenge
	disable_tlsalpn_challenge
	alt_http_port    <port>
	alt_tlsalpn_port <port>
	eab <key_id> <mac_key>
	trusted_roots <pem_files...>
	dns <provider_name> [<options>]
	propagation_timeout <duration>
	propagation_delay <duration>
	resolvers <dns_servers...>
	preferred_chains [smallest] {
		root_common_name <common_names...>
		any_common_name  <common_names...>
	}
}
```

- **dir** <span id="dir"/> is the URL to the ACME CA's directory. Default: `https://acme-v02.api.letsencrypt.org/directory`
- **test_dir** <span id="test_dir"/> is an optional fallback directory to use when retrying challenges; if all challenges fail, this endpoint will be used during retries; useful if a CA has a staging endpoint where you want to avoid rate limits on their production endpoint. Default: `https://acme-staging-v02.api.letsencrypt.org/directory`
- **email** <span id="email"/> is the ACME account contact email address.
- **timeout** <span id="timeout"/> is a [duration value](/docs/conventions#durations) that sets how long to wait before timing out an ACME operation.
- **disable_http_challenge** <span id="disable_http_challenge"/> will disable the HTTP challenge.
- **disable_tlsalpn_challenge** <span id="disable_tlsalpn_challenge"/> will disable the TLS-ALPN challenge.
- **alt_http_port** <span id="alt_http_port"/> is an alternate port on which to serve the HTTP challenge; it has to happen on port 80 so you must forward packets to this alternate port.
- **alt_tlsalpn_port** <span id="alt_tlsalpn_port"/> is an alternate port on which to serve the TLS-ALPN challenge; it has to happen on port 443 so you must forward packets to this alternate port.
- **eab** <span id="eab"/> specifies an External Account Binding which may be required with some ACME CAs.
- **trusted_roots** <span id="trusted_roots"/> is one or more root certificates (as PEM filenames) to trust when connecting to the ACME CA server.
- **dns** <span id="dns"/> configures the DNS challenge.
- **propagation_timeout** <span id="propagation_timeout"/> is a [duration value](/docs/conventions#durations) that sets the maximum time to wait for the DNS TXT records to appear when using the DNS challenge. Set to `-1` to disable propagation checks. Default 2 minutes.
- **propagation_delay** <span id="propagation_delay"/> is a [duration value](/docs/conventions#durations) that sets how long to wait before starting DNS TXT records propagation checks when using the DNS challenge. Default 0 (no wait).
- **resolvers** <span id="resolvers"/> customizes the DNS resolvers used when performing the DNS challenge; these take precedence over system resolvers or any default ones.
- **preferred_chains** <span id="preferred_chains"/> specifies which certificate chains Caddy should prefer; useful if your CA provides multiple chains. Use one of the following options:
	- **smallest** <span id="smallest"/> will tell Caddy to prefer chains with the fewest amount of bytes.
	- **root_common_name** <span id="root_common_name"/> is a list of one or more common names; Caddy will choose the first chain that has a root that matches with at least one of the specified common names.
	- **any_common_name** <span id="any_common_name"/> is a list of one or more common names; Caddy will choose the first chain that has an issuer that matches with at least one of the specified common names.


#### zerossl

Obtains certificates using the ACME protocol, specifically with ZeroSSL.

```caddy
... zerossl [<api_key>] {
	...
}
```

The syntax for `zerossl` is exactly the same as for `acme`, except that its name is `zerossl` and it can optionally take your ZeroSSL API key.

The functionality of the `zerossl` issuer is the same as the `acme` issuer, except that it will use ZeroSSL's directory by default and it can automatically negotiate EAB credentials (whereas with the `acme` issuer, you have to manually provide EAB credentials and set the directory endpoint).

When explicitly configuring `zerossl`, an email address is required so that your certificates can appear in your ZeroSSL dashboard.

Note that ZeroSSL is a default issuer, so configuring it explicitly is usually unnecessary.

#### internal

Obtains certificates from an internal certificate authority.

```caddy
... internal {
	ca       <name>
	lifetime <duration>
	sign_with_root
}
```

- **ca** <span id="ca"/> is the name of the internal CA to use. Default: `local`. See the [PKI app global options](/docs/caddyfile/options#pki-options) to configure alternate CAs.
- **lifetime** <span id="lifetime"/> is a [duration value](/docs/conventions#durations) that sets the validity period for interally issued leaf certificates. Default: 12h. It is NOT recommended to not change this, unless absolutely necessary.
- **sign_with_root** <span id="sign_with_root"/> forces the root to be the issuer instead of the intermediate. This is NOT recommended and should only be used when devices/clients do not properly validate certificate chains (very uncommon).



### Certificate Managers

Certificate manager modules are distinct from issuer modules in that use of manager modules implies that an external tool or service is keeping the certificate renewed, whereas an issuer module implies that Caddy itself is managing the certificate. (Issuer modules take a Certificate Signing Request (CSR) as input, but certificate manager modules take a TLS ClientHello as input.)

These manager modules come standard with the `tls` directive:

#### tailscale

Get certificates from a locally-running [Tailscale](https://tailscale.com) instance. [HTTPS must be enabled in your Tailscale account](https://tailscale.com/kb/1153/enabling-https/) (or your open source [Headscale server](https://github.com/juanfont/headscale)); and the Caddy process must either be running as root, or you must configure `tailscaled` to give your Caddy user [permission to fetch certificates](https://github.com/caddyserver/caddy/pull/4541#issuecomment-1021568348).

_**NOTE: This is usually unnecessary!** Caddy automatically uses Tailscale for all `*.ts.net` domains without any extra configuration._

```caddy-d
get_certificate tailscale  # often unnecessary!
```


#### http

Get certificates by making an HTTP(S) request. The response must have a 200 status code and the body must contain a PEM chain including the full certificate (with intermediates) as well as the private key.

```caddy-d
get_certificate http <url>
```

- **url** <span id="url"/> is the fully-qualified URL to which to make the request. It is strongly advised that this be a local endpoint for performance reasons. The URL will be augmented with the following query string parameters: `server_name` = SNI value, `signature_schemes` = comma-separated list of hex IDs of signature algorithms, and `cipher_suites` = comma-separated list of hex IDS of cipher suites.



## Examples

Use a custom certificate and key:

```caddy-d
tls cert.pem key.pem
```

Use locally-trusted certificates for all hosts on the current site block, rather than public certificates via ACME / Let's Encrypt (useful in dev environments):

```caddy-d
tls internal
```

Use locally-trusted certificates, but managed on-demand intead of in the background:

```caddy-d
tls internal {
	on_demand
}
```

Use custom options for the internal CA (cannot use the `tls internal` shortcut):

```caddy-d
tls {
	issuer internal {
		ca foo
	}
}
```

Specify an email address for your ACME account (but if only one email is used for all sites, we recommend the `email` [global option](/docs/caddyfile/options) instead):

```caddy-d
tls your@email.com
```

Enable the DNS challenge for a domain managed on Cloudflare with account credentials in an environment variable:

```caddy-d
tls {
	dns cloudflare {env.CLOUDFLARE_API_TOKEN}
}
```

Get the certificate chain via HTTP, instead of having Caddy manage it:

```caddy-d
tls {
	get_certificate http http://localhost:9007/certs
}
```

Enable TLS Client Authentication and require clients to present a valid certificate that is verified against all the provided CA's via `trusted_ca_cert_file`

```caddy-d
tls {
	client_auth {
		mode                 require_and_verify
		trusted_ca_cert_file ../caddy.ca.cer
		trusted_ca_cert_file ../root.ca.cer
	}
}
```
