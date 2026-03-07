---
title: "AWS ALB finally supports URL rewriting – goodbye nginx ingress"
date: 2026-03-07
draft: false
tags: ["aws", "eks", "kubernetes", "alb", "nginx", "ingress"]
ShowToc: true
TocOpen: false
cover:
    image: "images/eks/alb-hero.png"
    alt: "Network complexity transitioning to simplicity"
    caption: "Image generated with Amazon Nova Canvas"
---

If you've run services on EKS, you've probably had the same frustration I have: AWS Application Load Balancer is great for TLS termination and routing, but the moment you need URL rewriting, you're stuck deploying nginx as a reverse proxy behind it. Two load balancers, an extra pod to maintain, and one more thing to patch.

That's finally over. AWS added native URL rewriting to ALB in October 2025, and the timing couldn't be better – the Kubernetes community announced that Ingress NGINX is retiring on March 31, 2026. The last reason to run nginx on EKS just disappeared.

## The problem

I run a small test application on EKS called Yin-Yang – it displays different colors based on the hostname it's running on, which makes it easy to visualize load balancing across pods. The app expects to be served at `/`, but I want it under a `/yinyang` subpath on my domain. That means something needs to strip the prefix before the request hits the container.

![Yin-Yang app showing the pod name and a color-coded yin-yang symbol](/images/eks/yinyang-app.png)
*The Yin-Yang test app – each pod gets a unique color, making it easy to see which pod is serving your request. Written in Rust, because why not.*

With nginx, this is one annotation:

```yaml
nginx.ingress.kubernetes.io/rewrite-target: /
```

With ALB, this was impossible. ALB could route based on paths, but it couldn't modify the URL before forwarding. The workarounds were all bad:

1. **Give each service its own subdomain** – `yinyang.example.com` instead of `example.com/yinyang`. Avoids the rewrite problem entirely, but now you need a DNS record and TLS certificate per service. Doesn't scale.
2. **Hardcode the subpath into the container** – make the app serve from `/yinyang` instead of `/`. Couples your deployment topology to your application code. Move the app to a different path and you're rebuilding the container.
3. **Deploy nginx behind ALB** – ALB handles TLS, nginx handles the rewrite. Works, but you're running a whole reverse proxy for one annotation.

Most people, myself included, went with option 3. So the standard EKS pattern became:

```
Internet → ALB (TLS + routing) → nginx (URL rewrite) → Service → Pods
```

Two hops. Two things to configure. Two things that can break.

## The old setup

Here's what my config looked like. First, an ALB ingress that catches all traffic and forwards it to nginx:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: yinyang-alb-to-nginx
  namespace: ingress-nginx
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:region:account:certificate/your-cert-id
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS13-1-2-2021-06
    alb.ingress.kubernetes.io/inbound-cidrs: your-ip/32
spec:
  ingressClassName: alb
  rules:
    - host: my-domain.example.com
      http:
        paths:
          - path: /*
            pathType: ImplementationSpecific
            backend:
              service:
                name: ingress-nginx-controller
                port:
                  number: 80
```

Then a separate nginx ingress that does the actual rewrite:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: yinyang-nginx-simple
  namespace: yinyang
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: my-domain.example.com
      http:
        paths:
          - path: /yinyang
            pathType: Prefix
            backend:
              service:
                name: yinyang-service
                port:
                  number: 80
```

Plus the nginx ingress controller itself – a Helm chart, a deployment, a service, RBAC rules. All for one URL rewrite.

## What changed

On October 15, 2025, AWS [announced](https://aws.amazon.com/about-aws/whats-new/2025/10/application-load-balancer-url-header-rewrite) native URL and host header rewrite for ALB. The feature uses regex-based pattern matching and is configured through listener rule transforms.

The AWS Load Balancer Controller for Kubernetes picked this up via the `transforms` annotation. Here's my entire setup now – one file:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: yinyang-alb-rewrite
  namespace: yinyang
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:region:account:certificate/your-cert-id
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/transforms.yinyang-service: |
      [
        {
          "type": "url-rewrite",
          "urlRewriteConfig": {
            "rewrites": [{
              "regex": "^/yinyang/?(.*)$",
              "replace": "/$1"
            }]
          }
        }
      ]
spec:
  ingressClassName: alb
  rules:
    - host: my-domain.example.com
      http:
        paths:
          - path: /yinyang
            pathType: Prefix
            backend:
              service:
                name: yinyang-service
                port:
                  number: 80
```

The key is the `transforms` annotation. It takes a JSON array of transform rules – in this case a regex that strips the `/yinyang` prefix and ensures a leading slash. The annotation name includes the service name (`transforms.yinyang-service`) so you can have different rewrites for different backends.

## Before and after

<div style="display:grid; grid-template-columns:1fr 1fr; gap:1em; margin:1.5em 0;">
<div>
<div class="mermaid">
graph LR
    subgraph "Before"
        A1[Internet] --> B1[ALB<br/>TLS + routing]
        B1 --> C1[nginx pod<br/>URL rewrite]
        C1 --> D1[Service]
    end
</div>
</div>
<div>
<div class="mermaid">
graph LR
    subgraph "After"
        A2[Internet] --> B2[ALB<br/>TLS + routing<br/>+ rewrite]
        B2 --> D2[Service]
    end
</div>
</div>
</div>

What I removed:
- The nginx ingress controller Helm chart
- The nginx ingress controller deployment and service
- The ALB-to-nginx ingress resource
- The nginx ingress resource
- All the RBAC for nginx

What I kept:
- One ALB ingress resource with the `transforms` annotation

## The nginx retirement

The timing of this is worth noting. In November 2025, the Kubernetes SIG Network and Security Response Committee [announced](https://www.kubernetes.dev/blog/2025/11/12/ingress-nginx-retirement/) that the community Ingress NGINX controller is retiring. Best-effort maintenance continues until March 31, 2026 – after that, no releases, no bugfixes, no security patches.

This isn't about NGINX the web server (that's fine) or the commercial NGINX Ingress Controller from F5 (that's a separate product). It's specifically the community-maintained Kubernetes Ingress NGINX controller that most of us have been using.

The recommended migration path is to the Kubernetes Gateway API, which is the successor to the Ingress API. But if your main reason for running nginx was URL rewriting behind ALB – as mine was – you can skip that entirely and go ALB-native.

## Things to watch out for

A few notes from the migration:

**The `transforms` annotation is per-service.** The annotation key includes the backend service name: `alb.ingress.kubernetes.io/transforms.yinyang-service`. If you have multiple backends with different rewrite rules, each gets its own annotation.

**Regex syntax.** The rewrite uses standard regex. My pattern `^/yinyang/?(.*)$` → `/$1` handles both `/yinyang` and `/yinyang/` and `/yinyang/anything/else`. Test your patterns.

**AWS Load Balancer Controller version.** Make sure you're running a recent version that supports the `transforms` annotation. I'm using v2.12+ which has full support.

**EKS Auto Mode.** If you're on EKS Auto Mode, the transforms work slightly differently through the built-in ingress handling rather than the standalone controller. Check the [AWS documentation](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/tasks/url_rewrite/) for the specifics.

## The bottom line

For years, the answer to "how do I rewrite URLs on EKS" was "deploy nginx." Now it's one annotation. If you're still running nginx purely for URL rewriting, this is a good time to simplify – especially with the March 31 retirement deadline approaching.

---

*The configs shown here are from a real EKS cluster I use for testing AWS features. If you're doing something similar, I'd love to hear about your setup – find me on [LinkedIn](https://www.linkedin.com/in/ilkka-anttonen) or [Bluesky](https://bsky.app/profile/sirile.bsky.social).*
