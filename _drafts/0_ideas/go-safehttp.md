---
layout: post
title: "go-safehttp"
subtitle: "..."
date:   2025-04-01 16:00:00 +0200
categories: general golang
---

Write about the SSRF issue and the go-safehttp library to solve it.
Our issue: We have some webhook capabilities in our system that allow our users to register callbacks that are triggered if specific events are happening in the system. To test those we allow our users to do a "test execution" of the webhook and for debugging purposes we are also showing the result of the execution to them.
This causes a problem. It users are configuring the webhook to target the GCP metadata layer they are able to extract data from the running service account. This contains user and permission details and also access tokens.
When looking into the topic I figured out that there are some good guidelines on how to solve this issue in general, those essentially boil down to resolving the IP addresses including redirects and then validating them against a block or allowlist.
There is currently no commonly used golang-package that provides features like that, so we went on writing our own drop in replacement for the go http client. It works by implementing a custom transport-function that validates the IP.
Additionally there are other related security issues that we wanted to tackle as well: TLS minimum version verification, redirect limits, scheme validation and decompressed response body size limits (to limit DoS risk)
The package should provide sensible defaults. For example for the IP blocking we are using those
```
var defaultBlockedCIDRs = mustPrefixes(
	"10.0.0.0/8",      // RFC1918 private-use
	"100.64.0.0/10",   // Carrier-grade NAT shared space
	"127.0.0.0/8",     // IPv4 loopback
	"169.254.0.0/16",  // IPv4 link-local
	"172.16.0.0/12",   // RFC1918 private-use
	"192.0.0.0/24",    // IETF protocol assignments (non-public)
	"192.0.2.0/24",    // TEST-NET-1 documentation range
	"192.168.0.0/16",  // RFC1918 private-use
	"198.18.0.0/15",   // Benchmarking range
	"198.51.100.0/24", // TEST-NET-2 documentation range
	"203.0.113.0/24",  // TEST-NET-3 documentation range
	"240.0.0.0/4",     // Reserved (includes limited broadcast)
	"::1/128",         // IPv6 loopback
	"fc00::/7",        // IPv6 unique local addresses
	"fe80::/10",       // IPv6 link-local unicast
	"2001:db8::/32",   // IPv6 documentation range
)
```
which are a combination of private and reserved CIDRs.
Also by default we only allow https scheme, min TLS 1.2 and max 10 MB of response body size. All of those should be easily customisable for the user.