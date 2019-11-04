# Instant HTTPS BOSH Release

> We have archived this repository as it is no longer actively used or maintained.  For your `caddy` + `bosh` needs, please consider [dpb587/caddy-bosh-release](https://github.com/dpb587/caddy-bosh-release) instead!

This BOSH release is designed as a simple, low-maintenance way to just add HTTPS to web services.  It's primarily designed for dashboards, status pages and other tooling used by dev and ops teams, but might be useful for other purposes (such as front-ending a Cloud Foundry lab environment).

## How it Works
It's simply a [Caddy webserver](https://caddyserver.com/) instance that reverse proxies to your service.  SSL certificates are automatically provided by [Let's Encrypt](https://letsencrypt.org/) (or another ACME-compatible provider).

The release can be added to existing deployments, and the proxy can run on the same machine as the service or...  you can run a standalone proxy that proxies to services running elsewhere on your network.

## Important Info About Let's Encrypt
If you use this with one of Let's Encrypt's API URLs, you'll be agreeing to the [Let's Encrypt Subscriber Agreement](https://letsencrypt.org/repository/).

When testing your configuration, use the Let's Encrypt's staging API's URL (https://acme-staging-v02.api.letsencrypt.org/directory).  Let's Encrypt's production API has a rate limit of 60 failed validations per hour, so it's recommended that you leave this setting until you're satisfied your deployment works.  (The staging API works just like the production one, but returns a certificate that's not trusted by browsers.)  To get production certs from Let's Encrypt, set the `tls.ca` Caddyfile property to `https://acme-v02.api.letsencrypt.org/directory`.

## Caddy Plugins

The following Caddy plugins have been baked into this release:

* [`tls.dns.googlecloud`](https://caddyserver.com/docs/tls.dns.googlecloud): Allows you to obtain certificates using DNS records for domains managed with Google Cloud.

## Usage
### Standalone Proxy
This snippet shows an instant-https deployment acting as a proxy to multiple backends:

```yaml
instance_groups:
- name: https_proxy
  jobs:
  - name: proxy
    release: instant-https
    properties:
      contact_email: me@example.com
      googlecloud:
        GCE_PROJECT: sample-project-201802
        GCE_SERVICE_ACCOUNT_FILE: |
          {
            "type": "sample_account",
            "project_id": "sample-project-201802",
            "private_key_id": "sample4c02016ac2d8abf5b1577993ded31626a8",
            "private_key": "-----BEGIN PRIVATE KEY-----\nMIIE...k8LAVeB==\n-----END PRIVATE KEY-----\n",
            "client_email": "instant-https@sample-project-201802.iam.gserviceaccount.com",
            "client_id": "sample4c02016ac2d8abf",
            "auth_uri": "https://accounts.google.com/o/oauth2/auth",
            "token_uri": "https://accounts.google.com/o/oauth2/token",
            "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
            "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/instant-https%sample-project-201802.iam.gserviceaccount.com"
          }
      caddyfile: |
        (wildcard_cert) {
          tls {
            dns googlecloud
            wildcard
          }
        }
        ci.example.com {
          import wildcard_cert
          log / /var/vcap/sys/log/proxy/ci.access.log "{combined} {scheme} {host}"
          proxy / http://10.0.10.10:8080 {
            websocket
            transparent
          }
          tls {
            ca https://acme-staging-v02.api.letsencrypt.org/directory
          }
        }
        opsman.example.com {
          import wildcard_cert
          log / /var/vcap/sys/log/proxy/opsman.access.log "{combined} {scheme} {host}"
          proxy / https://10.0.20.10:443 {
            transparent
            insecure_skip_verify
          }
          tls {
            ca https://acme-staging-v02.api.letsencrypt.org/directory
          }
        }
        *.cfapps.example.com *.run.example.com *.login.run.example.com *.uaa.run.example.com {
          log / /var/vcap/sys/log/proxy/pas.access.log "{combined} {scheme} {host}"
          proxy / https://10.0.30.10:443 {
            transparent
            websocket
            insecure_skip_verify
          }
          tls {
            ca https://acme-v02.api.letsencrypt.org/directory
            dns googlecloud
          }
        }
```
