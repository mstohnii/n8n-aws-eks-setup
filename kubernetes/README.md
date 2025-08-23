# n8n self-hosting at AWS

Official yaml manifests to setting up n8n at AWS EKS, but with my modifications

Modidication list:
- n8n-service.yaml
Added AWS Load Balancer annotations, including backend protocol, SSL certificate, SSL ports, and internet-facing scheme. Instead of a single port 5678, the service now exposes two ports — 80 (http) and 443 (https), both forwarding to 5678. Labels in metadata were removed, and the configuration is now oriented towards using an external load balancer.
- postgres-claim0-persistentvolumeclaim.yaml
Changed storage size from 300gb to 7gb
- postgres-deployment.yaml
Reduced resource request by half
- n8n-deployment.yaml
Changed protocol from HTTP to HTTPS, added N8N_HOST as njordops.net, and set WEBHOOK_TUNNEL_URL to https://njordops.net/.