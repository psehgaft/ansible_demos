# OCM Access Token Ansible Role

This document provides a ready-to-use Ansible role to generate a short-lived Red Hat OpenShift Cluster Manager (OCM) access token from a Red Hat Hybrid Cloud Console service account.

> Important: OCM access tokens are not non-expiring. Use `client_id` and `client_secret` to generate short-lived tokens at runtime.

## Quick start

```bash
unzip ansible-ocm-token-role.zip
cd ansible-ocm-token-role

export OCM_CLIENT_ID="<client_id>"
export OCM_CLIENT_SECRET="<client_secret>"

ansible-playbook playbooks/generate-ocm-token.yml
```

## Optional: validate against the OCM clusters API

```bash
ansible-playbook playbooks/generate-ocm-token.yml \
  -e ocm_validate_token=true \
  -e ocm_query_clusters=true
```

## Files included

```text
ansible-ocm-token-role/
├── ansible.cfg
├── README.md
├── playbooks/
│   └── generate-ocm-token.yml
└── roles/
    └── ocm_access_token/
        ├── defaults/
        │   └── main.yml
        ├── meta/
        │   └── main.yml
        ├── tasks/
        │   ├── assert.yml
        │   ├── main.yml
        │   ├── request_token.yml
        │   └── validate.yml
        └── README.md
```


### `ansible.cfg`

```ini
[defaults]
roles_path = ./roles
host_key_checking = False
retry_files_enabled = False
stdout_callback = yaml

```

### `playbooks/generate-ocm-token.yml`

```yaml
---
- name: Generate OCM access token from Red Hat Hybrid Cloud Console service account
  hosts: localhost
  connection: local
  gather_facts: false

  roles:
    - role: ocm_access_token
      vars:
        # Set true if the service account has permission to read the validation endpoint.
        ocm_validate_token: false

  post_tasks:
    - name: Show non-sensitive token metadata
      ansible.builtin.debug:
        msg:
          - "OCM token generated successfully."
          - "Token type: {{ ocm_token_type }}"
          - "Expires in seconds: {{ ocm_expires_in }}"
          - "Generated at UTC: {{ ocm_token_generated_at }}"

    - name: Example OCM API call using generated token
      ansible.builtin.uri:
        url: "{{ ocm_api_url }}/api/clusters_mgmt/v1/clusters"
        method: GET
        headers:
          Authorization: "Bearer {{ ocm_access_token }}"
          Accept: "application/json"
        return_content: true
      register: ocm_clusters
      no_log: "{{ ocm_no_log }}"
      when: ocm_query_clusters | default(false) | bool

    - name: Show cluster count from example query
      ansible.builtin.debug:
        msg: "OCM returned {{ ocm_clusters.json.items | default([]) | length }} clusters."
      when:
        - ocm_query_clusters | default(false) | bool
        - ocm_clusters is defined

```

### `roles/ocm_access_token/defaults/main.yml`

```yaml
---
# Red Hat SSO endpoint for Red Hat Hybrid Cloud Console / OCM API tokens.
ocm_sso_host: "https://sso.redhat.com"

# OAuth2 token endpoint used by Red Hat service accounts.
ocm_token_endpoint: "{{ ocm_sso_host }}/auth/realms/redhat-external/protocol/openid-connect/token"

# Red Hat documented scope for Hybrid Cloud Console service account usage.
ocm_scopes: "openid api.iam.service_accounts"

# OCM API Gateway.
ocm_api_url: "https://api.openshift.com"

# Optional validation after token generation.
# Set to true only if the service account has permission for the selected endpoint.
ocm_validate_token: false
ocm_validate_endpoint: "/api/clusters_mgmt/v1/clusters"

# Hide secrets and generated tokens from Ansible output.
ocm_no_log: true

# The role can read credentials from variables or from the environment.
# Priority:
# 1. Explicit Ansible vars: ocm_client_id / ocm_client_secret
# 2. Environment vars: OCM_CLIENT_ID / OCM_CLIENT_SECRET
ocm_client_id: "{{ lookup('env', 'OCM_CLIENT_ID') | default('', true) }}"
ocm_client_secret: "{{ lookup('env', 'OCM_CLIENT_SECRET') | default('', true) }}"

```

### `roles/ocm_access_token/tasks/main.yml`

```yaml
---
- name: Validate required OCM service account inputs
  ansible.builtin.import_tasks: assert.yml

- name: Request OCM access token
  ansible.builtin.import_tasks: request_token.yml

- name: Validate generated OCM access token
  ansible.builtin.import_tasks: validate.yml
  when: ocm_validate_token | bool

```

### `roles/ocm_access_token/tasks/assert.yml`

```yaml
---
- name: Assert OCM Client ID is defined
  ansible.builtin.assert:
    that:
      - ocm_client_id is defined
      - ocm_client_id | length > 0
    fail_msg: >-
      ocm_client_id is required. Set it as an Ansible variable or export
      OCM_CLIENT_ID in the execution environment.

- name: Assert OCM Client Secret is defined
  ansible.builtin.assert:
    that:
      - ocm_client_secret is defined
      - ocm_client_secret | length > 0
    fail_msg: >-
      ocm_client_secret is required. Set it as an Ansible variable, Ansible Vault,
      AAP credential, or export OCM_CLIENT_SECRET in the execution environment.
  no_log: "{{ ocm_no_log }}"

```

### `roles/ocm_access_token/tasks/request_token.yml`

```yaml
---
- name: Request short-lived OCM access token from Red Hat SSO
  ansible.builtin.uri:
    url: "{{ ocm_token_endpoint }}"
    method: POST
    body_format: form-urlencoded
    body:
      grant_type: "client_credentials"
      client_id: "{{ ocm_client_id }}"
      client_secret: "{{ ocm_client_secret }}"
      scope: "{{ ocm_scopes }}"
    headers:
      Accept: "application/json"
    return_content: true
    status_code: 200
  register: ocm_token_response
  no_log: "{{ ocm_no_log }}"

- name: Assert token response contains access_token
  ansible.builtin.assert:
    that:
      - ocm_token_response.json is defined
      - ocm_token_response.json.access_token is defined
      - ocm_token_response.json.access_token | length > 0
    fail_msg: >-
      Red Hat SSO did not return an access_token. Validate the service account
      Client ID, Client Secret, scopes, and Red Hat User Access permissions.
  no_log: "{{ ocm_no_log }}"

- name: Expose OCM token facts for downstream tasks
  ansible.builtin.set_fact:
    ocm_access_token: "{{ ocm_token_response.json.access_token }}"
    ocm_token_type: "{{ ocm_token_response.json.token_type | default('Bearer') }}"
    ocm_expires_in: "{{ ocm_token_response.json.expires_in | default(900) }}"
    ocm_token_generated_at: "{{ lookup('pipe', 'date -u +%Y-%m-%dT%H:%M:%SZ') }}"
  no_log: "{{ ocm_no_log }}"

```

### `roles/ocm_access_token/tasks/validate.yml`

```yaml
---
- name: Validate OCM token against selected OCM API endpoint
  ansible.builtin.uri:
    url: "{{ ocm_api_url }}{{ ocm_validate_endpoint }}"
    method: GET
    headers:
      Authorization: "Bearer {{ ocm_access_token }}"
      Accept: "application/json"
    return_content: true
    status_code:
      - 200
  register: ocm_validation_response
  no_log: "{{ ocm_no_log }}"

- name: Expose OCM token validation metadata
  ansible.builtin.set_fact:
    ocm_token_validated: true
    ocm_validation_status: "{{ ocm_validation_response.status }}"
  no_log: "{{ ocm_no_log }}"

```

### `roles/ocm_access_token/meta/main.yml`

```yaml
---
galaxy_info:
  role_name: ocm_access_token
  namespace: openshift_automation
  author: Heber Romero / OpenShift Automation
  description: Generate a short-lived OCM access token from a Red Hat Hybrid Cloud Console service account.
  license: MIT
  min_ansible_version: "2.14"
  platforms:
    - name: EL
      versions:
        - "9"
        - "8"

dependencies: []

```


## Security notes

- Keep `ocm_no_log: true`.
- Store the service account secret in Ansible Automation Platform credentials, Ansible Vault, HashiCorp Vault, CyberArk, AWS Secrets Manager, or Azure Key Vault.
- Do not commit `client_secret` or generated access tokens to Git.
- Assign only the OCM/RBAC roles required for your automation.
- Regenerate the token per run; do not persist it.
