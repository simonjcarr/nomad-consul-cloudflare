# Traefik Nomad Role

This role deploys Traefik as a Docker container via a Nomad job, with configuration for Consul Catalog service discovery. It ensures Traefik is integrated with Consul and ready to load-balance requests for hostnames like `app1.carrtech.dev` to services registered by Nomad jobs.

## Files and Templates
- `templates/traefik.nomad.j2`: Nomad job spec for Traefik
- `templates/traefik.yml.j2`: Traefik static configuration
- `templates/acme.json.j2`: ACME storage for Traefik (for Let's Encrypt, if needed)

## Variables
- `traefik_config_dir`: Directory on the host for Traefik config (default: `/home/simon/traefik-config`)
- `traefik_nomad_job_path`: Path for rendered Nomad job spec (default: `/home/simon/jobs/traefik.nomad`)

## Usage
1. Add the `traefik` role to your playbook for the appropriate host/group.
2. Ensure Consul and Nomad are running and accessible from the Traefik host.
3. Deploy services in Nomad with Consul tags for Traefik (see below).

## Example Nomad Job Service Block
```
service {
  name = "app1"
  port = "http"
  tags = [
    "traefik.enable=true",
    "traefik.http.routers.app1.rule=Host(`app1.carrtech.dev`)",
    "traefik.http.services.app1.loadbalancer.server.port=8080"
  ]
}
```

## Notes
- This role expects the Consul agent to be available on `127.0.0.1:8500`.
- Traefik will listen on port 8080 for HTTP traffic.
- Cloudflare Tunnel should forward `*.carrtech.dev` to Traefik's HTTP port.
