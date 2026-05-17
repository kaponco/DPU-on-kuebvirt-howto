# ROSA CLI Setup on New Laptop

Quick checklist to get `rosa` CLI working on a new machine for an existing cluster.

## Phase 1: Install ROSA CLI

- [ ] Install rosa CLI
  **macOS (Homebrew):**
  ```bash
  brew install rosa-cli
  ```
  
  **macOS (Manual):**
  ```bash
  # Download latest release
  curl -L https://mirror.openshift.com/pub/openshift-v4/clients/rosa/latest/rosa-darwin.tar.gz -o rosa-darwin.tar.gz
  tar -xzf rosa-darwin.tar.gz
  sudo mv rosa /usr/local/bin/rosa
  chmod +x /usr/local/bin/rosa
  ```
  
  **Linux:**
  ```bash
  curl -L https://mirror.openshift.com/pub/openshift-v4/clients/rosa/latest/rosa-linux.tar.gz -o rosa-linux.tar.gz
  tar -xzf rosa-linux.tar.gz
  sudo mv rosa /usr/local/bin/rosa
  chmod +x /usr/local/bin/rosa
  ```

- [ ] Verify installation
  ```bash
  rosa version
  ```

## Phase 2: Install AWS CLI (if not already installed)

- [ ] Check if AWS CLI is installed
  ```bash
  aws --version
  ```

- [ ] If not installed, install AWS CLI
  **macOS (Homebrew):**
  ```bash
  brew install awscli
  ```
  
  **Other methods:** https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

## Phase 3: Configure AWS Credentials

- [ ] Configure AWS credentials for the account where ROSA cluster is running
  ```bash
  aws configure
  ```
  Enter:
  - AWS Access Key ID
  - AWS Secret Access Key
  - Default region (the region where your ROSA cluster is)
  - Default output format (json)

- [ ] Verify AWS credentials work
  ```bash
  aws sts get-caller-identity
  ```

- [ ] Verify AWS region
  ```bash
  aws configure get region
  ```

## Phase 4: Log in to Red Hat Account

- [ ] Initialize rosa and log in with Red Hat account
  ```bash
  rosa login
  ```
  
  This will:
  - Open a browser to https://console.redhat.com/openshift/token/rosa
  - Prompt you to log in with your Red Hat account
  - Provide a token to paste back in the terminal

- [ ] Alternatively, use token directly
  ```bash
  rosa login --token=<your-token>
  ```
  Get token from: https://console.redhat.com/openshift/token/rosa

- [ ] Verify login successful
  ```bash
  rosa whoami
  ```

## Phase 5: Verify Cluster Access

- [ ] List your ROSA clusters
  ```bash
  rosa list clusters
  ```

- [ ] Describe your cluster
  ```bash
  rosa describe cluster --cluster=<cluster-name>
  ```

- [ ] Get cluster API URL
  ```bash
  rosa describe cluster --cluster=<cluster-name> | grep "API URL"
  ```

## Phase 6: Install and Configure oc CLI

- [ ] Install oc CLI (if not already installed)
  **macOS (Homebrew):**
  ```bash
  brew install openshift-cli
  ```
  
  **Manual download:**
  ```bash
  # Get oc from ROSA cluster or downloads page
  rosa download oc
  # Or download from: https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/
  ```

- [ ] Verify oc installation
  ```bash
  oc version
  ```

- [ ] Create cluster-admin user (if you don't have one)
  ```bash
  rosa create admin --cluster=<cluster-name>
  ```
  
  This will output login credentials and command like:
  ```
  oc login https://api.<cluster>.<domain>:6443 --username cluster-admin --password <password>
  ```

- [ ] Or use existing credentials to log in
  ```bash
  oc login <api-url> --username <username> --password <password>
  ```

- [ ] Verify oc is connected
  ```bash
  oc whoami
  oc get nodes
  ```

## Success Criteria

- [ ] `rosa version` shows installed version
- [ ] `rosa whoami` shows your Red Hat account
- [ ] `rosa list clusters` shows your cluster
- [ ] `oc whoami` shows your authenticated user
- [ ] `oc get nodes` shows cluster nodes

## Quick Reference Commands

```bash
# Login to Red Hat account
rosa login

# List clusters
rosa list clusters

# Describe cluster
rosa describe cluster --cluster=<cluster-name>

# Create admin user
rosa create admin --cluster=<cluster-name>

# Login to cluster with oc
oc login <api-url> --username <username> --password <password>

# Verify cluster access
oc get nodes
```

## Troubleshooting

### "rosa login" fails
- Check internet connection
- Visit https://console.redhat.com/openshift/token/rosa manually
- Verify Red Hat account is active

### AWS credentials error
- Run `aws sts get-caller-identity` to verify AWS access
- Check AWS credentials: `cat ~/.aws/credentials`
- Verify correct AWS region: `aws configure get region`

### "rosa list clusters" shows no clusters
- Verify AWS credentials are for the correct account
- Check AWS region matches where cluster was created
- Run `rosa describe cluster --cluster=<cluster-name>` with exact cluster name

### "oc login" fails
- Verify API URL is correct: `rosa describe cluster --cluster=<cluster-name> | grep "API URL"`
- Check credentials are valid
- Create new admin user: `rosa create admin --cluster=<cluster-name>`

## Notes

- ROSA CLI authentication (via `rosa login`) is separate from cluster authentication (via `oc login`)
- You need both `rosa` (for managing ROSA resources) and `oc` (for managing cluster resources)
- AWS credentials must match the account where the ROSA cluster was created
- Admin users created with `rosa create admin` are temporary (24 hours by default)
