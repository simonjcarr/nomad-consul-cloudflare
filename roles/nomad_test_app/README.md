# Nomad Test App Role

This role deploys a test application (`nginxdemos/hello`) via a Nomad job with 3 replicas. It is configured to be accessible at https://nomad-test.carrtech.dev via Traefik and the Cloudflare tunnel.

## Features
- 3 replicas for load balancing
- Traefik router for `nomad-test.carrtech.dev`
- Health check for the service

## How to test
After running the playbook, visit https://nomad-test.carrtech.dev in your browser. You should see the nginx demo app, and requests will be load balanced across the 3 replicas.
