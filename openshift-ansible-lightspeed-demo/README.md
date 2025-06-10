# OpenShift / Ansible Lightspeed & AI Demo

## Overview
This repository contains all the materials for a 2-hour hands-on demo showcasing the power of Ansible Lightspeed with Watson Code Assistant (WCA) on OpenShift.

## Structure

- `playbooks/nginx_pod`: Playbooks generated with WCA for deploying a simple NGINX pod.
- `playbooks/flask_ml_model`: Playbooks for deploying a Flask-based ML model using OpenShift AI.
- `pipelines`: GitOps-compatible CI/CD pipeline definitions for automated deployment.
- `docs`: Presentation slides and supporting documentation.
- `resources`: Example data, model files, and training scripts.

## Objectives

- Understand how to generate Ansible automation using natural language.
- Deploy workloads and AI models with OpenShift using Ansible Lightspeed.
- Showcase end-to-end GitOps automation with Lightspeed and OpenShift AI.

## Prerequisites

- Access to Red Hat OpenShift 4.x
- Watson Code Assistant enabled
- OpenShift GitOps operator installed
- User privileges for creating projects and pipelines

## Demo Scenarios

1. Generate and deploy an NGINX pod via prompt to WCA.
2. Deploy a pre-trained ML model using Flask and expose it via OpenShift.
3. Orchestrate the pipeline with GitOps integration.

## Resources

- [Ansible Lightspeed](https://www.ansible.com/lightspeed)
- [OpenShift AI](https://www.redhat.com/en/technologies/cloud-computing/openshift/openshift-ai)
