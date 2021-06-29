# Finsweet's Reverse Proxy

This repository contains the Cloudflare Worker script for reverse proxying all the Finsweet subdomains under the main `finsweet.com` domain, as well as documentation on how to properly manage them.

## DNS Management

The DNS of the `finsweet.com` domain are managed from Cloudflare. You can access it by asking @alexiglesias to include you as a team member of the Cloudflare account.

In there, you will notice that some DNS records are set to be proxied, and some not. Continue reading to know the differences between them.

![Proxied DNS](./images/proxied-dns-list.png)

### Difference between Proxied / Unproxied DNS

The main difference resides wether Cloudflare will intercept any request to the destination before it reaches it.

As a rule of thumb, only the reverse-proxied subdomains need to be proxied to intercept any request to them before it reaches Webflow's servers.

Any DNS that is set to point a non-Webflow destination, like `cdn.finsweet.com` should be set to `DNS only`.

### Why are the Webflow sites poiting to proxy.webflow.com instead of proxy-ssl.webflow.com?

There are two reasons for this:

1.  How Webflow generates the SSL certificate to all the binded domains.
2.  The requirements for an SSL Handshake to be valid.

#### How Webflow generates the SSL certificate to all the binded domains

When pointing a DNS record to `proxy-ssl.webflow.com`, Webflow automatically creates an SSL certificate that is valid for 3 months, and auto-renews it when this period is coming to an end.

The problem comes when we proxy this DNS record, as it is no longer pointed directly to Webflow's servers, and instead it goes through Cloudflare first. This causes [Webflow not being able to generate a valid SSL certificate](https://forum.webflow.com/t/error-525-ssl-handshake-failed/73756/2).

The only solution to this problem is uploading custom SSL certificates to Webflow, which is currently something that is only offered to [Enterprise customers](https://university.webflow.com/lesson/ssl-hosting#upload-a-custom-ssl-certificate).

#### The requirements for an SSL Handshake to be valid

An SSL Handshake requires both parties (Cloudflare and Webflow) to share an SSL certificate to determine that a connection is secure.

As this is not possible, we need to point the connection to the plain `proxy.webflow.com` origin and rely on [Cloudflare's Flexible mode](https://support.cloudflare.com/hc/en-us/articles/200170416-End-to-end-HTTPS-with-Cloudflare-Part-3-SSL-options#h_4e0d1a7c-eb71-4204-9e22-9d3ef9ef7fef) to encrypt the connection between the browser and Cloudflare.

## Setting up a new Webflow site

The process is split in the following steps:

1. Correctly configuring the Webflow project.
2. Creating the subdomain's DNS record.
3. Adding the to-be-proxied subdomain to the Cloudflare Worker environment.

### Correctly configuring the Webflow project

#### Hosting Tab

In the **Hosting** tab, set up the project domain as `SUBDOMAIN_NAME.finsweet.com`. After the setup is done correctly, all the traffic will be served under `finsweet.com/SUBDOMAIN_NAME/`.

![Hosting Tab](./images/hosting-tab.PNG)

#### SEO Tab

In the **SEO** tab, add `https://www.finsweet.com/SUBDOMAIN_NAME` as the **Global Canonical Tag URL** (note the `https://www.` in front of the domain, this is mandatory).
This will make sure Google doesn't consider any content outside of the `www.finsweet.com` main domain to be duplicated.

![SEO Tab](./images/canonical-tag.PNG)

#### Custom Code Tab

In the **Custom Code** tab, add the same `https://www.finsweet.com/SUBDOMAIN_NAME` URL as the **Href Prefix**.
This will assure that all relative URLs accross the project point correctly to the reverse-proxied path instead of the original subdomain.

![Hosting Tab](./images/custom-code-tab.PNG)

#### [Optional] Purge Cache on publish

By default, Cloudflare adds a 4h cache to all the assets that are being reverse-proxied. This means that, if you publish any changes to the Webflow project (the `.webflow.io` staging domain is not cached, only the live `SUBDOMAIN_NAME.finsweet.com` version), it can take up to 4h for users to see the new changes.

To prevent this, there's the option of immediately purging the cache whenever a Finsweet site is published.

To do so, move to the **Integrations** tab and create a **Site Publish** Webhook with the following URL:

```
https://reverse-proxy.finsweet.com/purge-cache
```

![Hosting Tab](./images/site-publish-webhook.PNG)
