# GCP and Vault Integration Setup Notes

## Prerequisites

- Google Cloud SDK (gcloud CLI) - Required for Vault backend operations
- Configured gcloud CLI for service account creation
- Existing GCP project with project ID
- Environment variable: `export GOOGLE_PROJECT=<YOUR_GCP-project-id>`

## GCP Authentication Principals

Google APIs support two types of authentication principals:
1. User Accounts
2. Service Accounts

Reference: https://cloud.google.com/docs/authentication#service-accounts

## Environment Setup

### Vault Settings
- Deployment: Docker container
- Authentication Method: userpass
- User Policy: attached to user 'winyan'
- Secrets Engine: GCP

## 1. GCP Service Configuration

### Enable Required APIs
```bash
gcloud services enable \
  iam.googleapis.com \
  cloudresourcemanager.googleapis.com \
  storage-component.googleapis.com
```

### Set Project Environment
```bash
export GOOGLE_PROJECT=khaw-lab
```

### Configure Service Account
```bash
gcloud projects add-iam-policy-binding $GOOGLE_PROJECT \
  --member="serviceAccount:vault-integration@$GOOGLE_PROJECT.iam.gserviceaccount.com" \
  --role="roles/editor"
```

Result:
```yaml
Updated IAM policy for project [khaw-lab].
bindings:
- members:
  - serviceAccount:vault-integration@khaw-lab.iam.gserviceaccount.com
  role: roles/editor
- members:
  - user:[REDACTED]@gmail.com
  role: roles/owner
etag: BwYvDPRgk9M=
version: 1
```

### Create Service Account Key
```bash
gcloud iam service-accounts keys create vault-key.json \
  --iam-account=vault-integration@$GOOGLE_PROJECT.iam.gserviceaccount.com
```

Output:
```
created key [38d0a8e0c47354fbb5381c57f02193441bfdc3cb] of type [json] as [vault-key.json] for [vault-integration@khaw-lab.iam.gserviceaccount.com]
```

## 2. Vault Configuration

### Enable GCP Secrets Engine
```bash
vault secrets enable -path=khaw gcp
```

### Configure GCP Backend
```bash
vault write khaw/config credentials=@./vault-key.json project="khaw-lab"
```

### Rotate Root Credentials
```bash
vault write -f khaw/config/rotate-root
```

### Binding Configuration (khaw-binding.hcl)
```hcl
# GCP Resource binding for project-level access
resource "//cloudresourcemanager.googleapis.com/projects/khaw-lab" {
  roles = ["roles/viewer"]
}

# Optional: Add more granular bindings for specific services
resource "//storage.googleapis.com/projects/khaw-lab/buckets/my-bucket" {
  roles = ["roles/storage.objectViewer"]
}
```
Ref: https://developer.hashicorp.com/vault/docs/secrets/gcp#bindings

### Create Roleset
```bash
vault write khaw/roleset/khaw-token-roleset-3 \
    project="khaw-lab" \
    secret_type="access_token"  \
    token_scopes="https://www.googleapis.com/auth/cloud-platform" \
    bindings=@./vault-data/khaw-binding.hcl
```

## 3. Vault Policy Configuration

### Policy Definition
```hcl
# Allow reading and listing rolesets
path "khaw/roleset/*" {
  capabilities = ["read", "list"]
}

```

### Apply Policy to User
```bash
vault write -f auth/cohort-8/users/winyan policies=roleset-policy
```

### Generate Token
```bash
vault read gcp/roleset/my-token-roleset/token
```

## 4. Important Paths
- GCloud configurations: `~/.config/gcloud/configurations`
- Vault binding file: `./vault-data/khaw-binding.hcl`
- Service account key: `vault-key.json`

## 5. Directory Structure
```
vault-docker-gcp/
├── docker-compose.yml
├── vault-key.json
└── vault-data/
    └── khaw-binding.hcl
```

## 6. Docker Compose Configuration
```yaml
name: vault-gcp
services:
  vault-server-gcp:
    image: hashicorp/vault:1.18.0
    container_name: vault-gcp
    hostname:  vault-gcp
    environment:
      - VAULT_ADDR=http://localhost:8200
      - GOOGLE_APPLICATION_CREDENTIALS=/vault/config/vault-key.json
    ports:
      - "8200:8200"
    volumes:
      - ./vault-key.json:/vault/config/vault-key.json
      - ./vault-data:/vault/data
    cap_add:
      - IPC_LOCK
    networks:
      - vault-cluster-network

networks:
  vault-cluster-network:
    driver: bridge
```

## 7. Important Notes
- Always use least privilege principle when assigning roles
- Rotate service account keys regularly
- Keep vault-key.json secure and never commit to version control
- Use appropriate TTL for generated tokens
