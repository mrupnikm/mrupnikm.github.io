---
author: "Matic Rupnik"
authorLink: matic-rupnik
title: Let's encrypt a service with Taiscale and CloudFlare on Nixos
tags: [
  "Nixos",
  "selfhosted",
  "ssl",lg
  
]
date: 2025-01-15
toc: true
--- 

It took me some time to get to a setup for my local servers that I was truly happy with. While setting up my Kubernetes cluster a few months ago, I found this beautiful article that showcased the usage of the cert-manager operator. It demonstrated how to automatically create and renew local certificates in the cluster by simply creating an ingress resource. This seamless integration inspired me to explore a similar approach for my own setup.

I knew this was possible with Nix, but I couldn't find a definitive guide that explained it in simple terms or matched my requirements. The NixOS ACME documentation, while helpful in parts, lacked practical use cases (as is common with many NixOS wiki pages). Eventually, I stumbled upon a working method using Cloudflare. This setup leverages Tailscale's local networking capabilities combined with a public domain, Cloudflare, and a local DNS resolver like Pihole—or, in this case, a lightweight option such as Blocky.

Since I run most of my important services on Nix, I decided to replicate the ease of use described in that article. At the time, I was still relying on non-SSL proxying for various services, and this solution was a game-changer.

## Breaking Down the Core Concepts
Here are the main components of the setup:

1. Tailscale and Tailnet DNS Integration
Tailscale provides each device in your network (known as a Tailnet) with a unique IP address. This private network simplifies communication between devices, but I wanted to use a public SSL certificate issuer while keeping these IPs private. Tailscale also offers an option to define DNS resolvers for the Tailnet, which I utilized to integrate local and external resources.

2. Cloudflare and ACME Challenge
I own a public domain managed through Cloudflare, which I configured to handle ACME challenges. To understand the underlying mechanism, refer to the DNS-01 section of Let's Encrypt's documentation. For this setup, I generated an API token in Cloudflare to enable automated DNS entry updates for ACME.

3. Local DNS with Blocky
I started by setting up Blocky DNS in my Nix configuration. I added an entry to map photos.mydomain.com to the Tailnet IP of my proxy server. This way, devices in the Tailnet could resolve the domain correctly. The Blocky host IP was then added to Tailscale's DNS resolvers, ensuring that all connected devices could route queries appropriately.

Here’s a snippet of the Blocky configuration:

```nix

# ...
services.blocky = {
    enable = true;
    settings = {
        ports.dns = 53; # Port for incoming DNS Queries.
        upstreams.groups.default = [
            "https://dns.quad9.net/dns-query" # Using Cloudflare's DNS over HTTPS server for resolving queries.
        ];
        # For initially solving DoH/DoT Requests when no system Resolver is available.
        bootstrapDns = {
            upstream = "https://dns.quad9.net/dns-query";
            ips = [ "9.9.9.9" ];
        };
        # Custom DNS entries
        customDNS = {
            mapping = {
                "photos.mydomain.com" = "100.90.xxx.xxx";
            };
        };
    };
};
# ...
```
In this example, Quad9 is used as the upstream DNS resolver, but you can substitute it with another service if desired.

4. ACME and Nginx Integration
To tie everything together, I utilized the ACME and Nginx services in my Nix configuration. The ACME service required a file (cloudflare.sh) containing the authentication method for Cloudflare. This file includes the API token for managing DNS entries. For more details on generating this token, refer to the Cloudflare documentation: Create an API Token.
Here’s an example of the necessary configuration:

```sh
CLOUDFLARE_EMAIL="..."
CLOUDFLARE_API_KEY="..."
```

```nix
security.acme = {
    acceptTerms = true;
    certs."photos.mydomain.com" = {
        email = "my@email.com";
        dnsProvider = "cloudflare";
        dnsResolver = "1.1.1.1:53";
        environmentFile = "/tmp/pki/cloudflare.s";
        webroot = null; # Required in my case
    };
};

services.nginx.enable = true;
services.nginx.virtualHosts."photos.mydomain.com" = {
    enableACME = true; 
    forceSSL = true;
    locations."/" = {
        proxyPass = "http://[::1]:8181";
        proxyWebsockets = true;
    };
};
```

The result is a valid SSL certificate that renews automatically. Now, every device connected to my Tailnet can securely access photos.mydomain.com without additional configuration. This setup simplifies maintenance and enhances security across my infrastructure.