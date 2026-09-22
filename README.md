# vCluster Platform on Amazon EKS: Setup with Traefik, Migration to Envoy Gateway

A hands-on runbook that provisions a real EKS cluster, installs vCluster Platform behind Traefik (TLS passthrough), creates a tenant cluster, verifies access, then migrates the exposure layer from Traefik to Envoy Gateway (Gateway API), running both in parallel so the existing path keeps working while the new one is validated.

## What this covers

- Provisioning an EKS cluster with `eksctl`, including the EBS CSI driver and a default `gp3` StorageClass
- Installing Traefik with an AWS Network Load Balancer, and exposing vCluster Platform through TLS passthrough
- Installing vCluster Platform and creating a tenant cluster
- Verifying access via `curl` and the browser
- Installing Envoy Gateway alongside Traefik, without disrupting the existing path
- Recreating the same TLS passthrough setup using Gateway API (`GatewayClass`, `Gateway`, `TLSRoute`), on a separate, independent Network Load Balancer
- Verifying the new path works end to end, independently of Traefik

## Before you start

This assumes an AWS account with permissions to create an EKS cluster and associated resources (IAM, EC2, ELB), and the following tools installed locally: `eksctl`, `kubectl`, `helm`, and the `vcluster` CLI.

## Note on production readiness

This is a lab/testing runbook, not a production deployment guide. A few things are deliberately simplified here and would need to change for a real deployment: no dedicated domain is used (both the Traefik and Envoy Gateway paths rely on auto-generated NLB hostnames and self-signed certificates), and the two exposure paths (Traefik and Envoy Gateway) are meant to run in parallel only for validation, cutover and decommissioning Traefik afterward is a separate step not covered here.

## Full runbook

See [runbook.md](runbook.md) for the complete, step-by-step commands.
