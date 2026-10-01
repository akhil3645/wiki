# Deploy Flux on a small GCP VM

Walkthrough from our October 1, 2026 learning session: create a Mumbai VM, connect using OpenSSH, upload Flux, run it in tmux, and expose its health endpoint.

This is a **learning setup, not a production deployment**. Public IPs, project IDs, and key material are placeholders. The repository is public; never commit private keys, `.env` files, tokens, or Terraform state.

## 1. Understand the components

| Component      | Purpose                                                                 |
| -------------- | ----------------------------------------------------------------------- |
| Compute Engine | GCP's equivalent of an EC2 virtual machine                              |
| VPC network    | Connects resources inside your cloud environment                        |
| Subnet         | A regional private IP address range inside a VPC                        |
| Private IP     | Communication within private networking                                 |
| Public IP      | Internet-facing connectivity, subject to firewall rules                 |
| Firewall       | Controls which sources can connect to which protocols/ports             |
| SSH            | Encrypted remote login, normally over TCP port 22                       |
| SCP            | Secure file transfer over SSH                                           |
| tmux           | Keeps terminal sessions and their processes alive after SSH disconnects |

An IP identifies a network destination; a port identifies a service at that destination. DNS can map a readable domain name to an IP. A public IP alone does not make an application reachable: its process must be listening and the relevant firewall rules must allow traffic.

## 2. Create the VM

Choose **Terraform or manual Console creation** for the VM. Do not create duplicate resources through both. Manual resources do not automatically become Terraform-managed.

### Option A: Terraform

Install Terraform on your laptop, not as a project package or inside the deployed app. Create these files in the Flux project:

```text
infra/terraform/
├── main.tf
└── terraform.tfvars
```

The provider declaration in `main.tf` is:

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 8.0"
    }
  }
}
```

`hashicorp/google` is the Google Cloud plugin. `~> 8.0` permits provider versions from 8.0.0 up to, but not including, 9.0.0. This is not Terraform's own version.

Our configuration declared `project_id`, `region`, and `zone` string variables and passed them to the Google provider. Local `terraform.tfvars` supplies:

```hcl
project_id = "YOUR_GCP_PROJECT_ID"
region     = "asia-south1"
zone       = "asia-south1-a"
```

The infrastructure settings used were:

- VPC `flux-network`, with `auto_create_subnetworks = true`.
- GCP assigns subnet ranges automatically; the VM uses the subnet in its region. Auto mode creates regional subnets, not just one Mumbai subnet.
- VM `flux-vm`, zone `asia-south1-a`, machine type `e2-micro`: shared CPU and 1 GiB RAM.
- Network tag `flux-vm`. Tags and VM names are separate settings.
- Ubuntu image `ubuntu-os-cloud/ubuntu-2404-lts-amd64`: Ubuntu 24.04 LTS, x86-64.
- Boot disk: 10 GiB, `pd-standard`, with `auto_delete = true`.
- `network_interface` references the VPC; `access_config {}` requests an ephemeral external IPv4 address.
- Empty service-account configuration (`scopes = []`, no email) means no attached service account with this provider.
- Initially, metadata enabled OS Login and the firewall allowed IAP SSH only. The later direct-SSH setup changes those settings; see Section 3.

The boot disk persists when the VM stops, but is deleted with the VM. An ephemeral public IP can change after stopping and restarting. Neither the VM, disk, nor public IPv4 address should be assumed free in Mumbai.

Add these patterns to `.gitignore`:

```gitignore
**/.terraform/
*.tfstate
*.tfstate.*
*.tfvars
*.tfplan
```

Commit `.terraform.lock.hcl`. Initialization downloads providers into the local `.terraform/` directory and creates the lock file; it does not create cloud infrastructure.

Run on your laptop, from the project root:

```bash
gcloud auth application-default login
terraform -chdir=infra/terraform init
terraform -chdir=infra/terraform validate
terraform -chdir=infra/terraform plan
```

Review the plan before running `terraform -chdir=infra/terraform apply`. Only apply creates or changes infrastructure; it prompts for approval. Terraform does not need to stay running after deployment. With no remote backend configured, its state is local and must be preserved securely.

Prerequisites: project billing, enabled Compute Engine API, sufficient IAM permissions, and quota/capacity. In our session:

- `invalid_grant` / `invalid_rapt` was resolved by refreshing **Application Default Credentials** using `gcloud auth application-default login`.
- `SERVICE_DISABLED` was resolved by enabling **Compute Engine API** for the correct project in the Console, waiting for propagation, and retrying.

### Option B: Google Cloud Console

The documented creation UI uses a navigation menu with separate configuration panes:

1. Select your project and open **Compute Engine → VM instances → Create instance**.
2. **Machine configuration:** name `flux-vm`, region Mumbai (`asia-south1`), zone `asia-south1-a`, E2 series, `e2-micro`, standard provisioning rather than Spot.
3. **OS and storage:** choose Ubuntu 24.04 LTS x86/64, standard persistent boot disk, size 10 GB as displayed in the Console. Match deletion of the boot disk with the VM.
4. **Networking:** add network tag `flux-vm`; choose `flux-network`, its Mumbai subnet, and an ephemeral external IPv4 address. Leave automatic HTTP/HTTPS firewall checkboxes unchecked initially.
5. **Security:** choose no service account to match this learning configuration.
6. **Advanced → Metadata:** initially `enable-oslogin = TRUE` for IAP/OS Login, or use the direct-key settings below instead.
7. **Data protection / Observability:** review optional backups and agent installation; these were not part of the minimal Terraform setup.
8. Review charges and click **Create** only when you intend to create a billable VM.

If the VPC does not exist, create `flux-network` separately under **VPC network → VPC networks**, using automatic subnet creation. Do not accidentally substitute the `default` network.

## 3. Connect using ordinary OpenSSH

We initially allowed IAP SSH, then enabled direct SSH using a local key. These are different network paths. OpenSSH supports either identity model, but registering keys in VM metadata requires OS Login to be disabled.

### Generate the key on your laptop

```bash
ssh-keygen -t ed25519 -f ~/.ssh/flux_ed25519 -C "flux-vm"
chmod 700 ~/.ssh
chmod 600 ~/.ssh/flux_ed25519
chmod 644 ~/.ssh/flux_ed25519.pub
cat ~/.ssh/flux_ed25519.pub
```

Choose a passphrase. Do not overwrite an existing key. The file without `.pub` is the private key and stays on your laptop. Only the `.pub` file is registered on the VM.

### Register the public key in Console

Open **Compute Engine → VM instances → flux-vm → Edit**:

1. Under custom metadata, set `enable-oslogin = FALSE`.
2. Find **SSH Keys** (use the browser's Find function if needed).
3. Add a separate entry containing your complete public key, ending in the Linux username `flux-vm`:

   ```text
   ssh-ed25519 YOUR_ACTUAL_PUBLIC_KEY_DATA flux-vm
   ```

4. Save. The username is not your Google email; in this metadata workflow, the final username identifies the Linux account.

Do not replace unrelated keys. A `google-ssh` key containing `userName` and `expireOn` is a separate, time-limited Google-managed key, not your local private key's counterpart. With OS Login enabled, metadata SSH keys are ignored.

### Allow direct SSH

Open **VPC network → Firewall → Create firewall rule**. The Console may group firewall pages under Network Security.

| Field                         | Value                   |
| ----------------------------- | ----------------------- |
| Name                          | `flux-allow-direct-ssh` |
| Network                       | **`flux-network`**      |
| Priority                      | `1000`                  |
| Direction                     | Ingress                 |
| Action                        | Allow                   |
| Targets dropdown              | Specified target tags   |
| Target tags text field        | `flux-vm`               |
| Source IPv4 ranges            | `0.0.0.0/0`             |
| Specified protocols and ports | TCP **22**              |

`0.0.0.0/0` was deliberately used for this exercise to permit direct SSH without source-IP restrictions. It exposes SSH globally. Prefer trusted source IPs or IAP for real deployments; keep password authentication disabled and update the OS.

The original `flux-allow-iap-ssh` rule permits TCP 22 only from `35.235.240.0/20`. It does not permit direct connections from your laptop. Both rules can exist.

### Connect

Copy the VM's **External IP** from the VM instances list. On your laptop:

```bash
VM_IP="YOUR_VM_EXTERNAL_IP"
ssh -o IdentitiesOnly=yes -i ~/.ssh/flux_ed25519 "flux-vm@$VM_IP"
```

Verify the server's host-key fingerprint using a trusted channel before accepting it. SSH saves accepted host keys in `~/.ssh/known_hosts`; this file verifies server identity, not your login permissions. Type `exit` to leave the VM.

## 4. Upload the app

Build on your laptop to avoid compiling on a 1 GiB VM. From the local Flux project root:

```bash
pnpm build
mkdir -p /tmp/flux-upload
tar -czf /tmp/flux-upload/flux-app.tar.gz \
  dist package.json pnpm-lock.yaml pnpm-workspace.yaml

VM_IP="YOUR_VM_EXTERNAL_IP"
scp -i ~/.ssh/flux_ed25519 \
  /tmp/flux-upload/flux-app.tar.gz "flux-vm@$VM_IP:~/"
```

This uploads compiled code and dependency manifests, not `.env`, `node_modules`, private keys, or infrastructure state. SSH runs remote commands; SCP transfers files using SSH. These commands are run locally, not inside the VM's SSH session.

On the VM:

```bash
mkdir -p ~/flux
tar -xzf ~/flux-app.tar.gz -C ~/flux
cd ~/flux
ls
```

This package is sufficient for this app's compiled entry point. If future features need templates, migrations, or other runtime assets, include those explicitly.

## 5. Install Node.js and dependencies on the VM

Run **inside the VM**, not on your laptop:

```bash
sudo apt-get update
sudo apt-get install -y curl ca-certificates
curl -fsSL https://deb.nodesource.com/setup_24.x -o /tmp/node-setup.sh
sudo bash /tmp/node-setup.sh
sudo apt-get install -y nodejs
node --version
npm --version
sudo npm install -g pnpm@11.22.0
cd ~/flux
pnpm install --prod --frozen-lockfile
```

The installer adds NodeSource's Node 24 repository. Review external installer scripts before running them with sudo. Version 11.22.0 matches Flux's package-manager declaration at the time of this walkthrough. Runtime dependencies are installed as the normal user; no VM build is needed.

`--prod` excludes development dependencies. `--frozen-lockfile` prevents changing the resolved dependency versions. Do not copy secrets into the package or Git. Shell environment variables can supply runtime settings; this app does not automatically load `.env` simply because that file exists.

## 6. Run in the background with tmux

On the VM:

```bash
sudo apt-get install -y tmux
tmux new -s flux
```

Inside the new tmux session:

```bash
cd ~/flux
NODE_ENV=production PORT=3000 pnpm start
```

Detach with **Ctrl+B**, release, then **D**. You may exit SSH afterward; the app continues running.

Useful VM commands:

```bash
tmux ls
tmux attach -t flux
```

To stop the app, attach and press Ctrl+C. To deploy another build, upload/extract the new package, install dependencies if needed, then restart the process.

**tmux is not a production process supervisor:** it does not restart a crashed app or automatically bring it back after a reboot. Use systemd for those guarantees. Do not run tmux and a systemd service on the same port simultaneously.

## 7. Test locally on the VM

From a second SSH session:

```bash
curl --max-time 5 http://127.0.0.1:3000/api/health
sudo ss -ltnp 'sport = :3000'
```

In our session, local curl returned:

```json
{"status":"ok"}
```

The listening socket appeared as `*:3000`, rather than localhost-only. This confirmed the application was running. It did not yet prove internet accessibility.

## 8. Allow browser/Postman access

For **direct testing on port 3000**, create a separate ingress firewall rule:

| Field                         | Value                             |
| ----------------------------- | --------------------------------- |
| Name                          | `flux-allow-app-3000`             |
| Network                       | **`flux-network`**, not `default` |
| Priority                      | `1000`                            |
| Action / direction            | Allow / Ingress                   |
| Targets dropdown              | Specified target tags             |
| Target tags text field        | `flux-vm`                         |
| Source IPv4 ranges            | `0.0.0.0/0`                       |
| Specified protocols and ports | TCP **3000**                      |

After creation, confirm `flux-vm` appears under **Applicable to instances**. A rule on `default` cannot apply to a VM on `flux-network`; matching tags alone are insufficient.

From your laptop:

```bash
VM_IP="YOUR_VM_EXTERNAL_IP"
curl --connect-timeout 10 --max-time 15 "http://$VM_IP:3000/api/health"
```

In a browser or Postman, send a GET request to:

```text
http://YOUR_VM_EXTERNAL_IP:3000/api/health
```

Expect `{"status":"ok"}`. The public test initially timed out; the app was healthy locally, but the first port-3000 firewall rule had accidentally been created on `default`. This guide does **not** claim the corrected public endpoint was verified during the recorded session; run the test to confirm your deployment.

**Security:** all app routes—not just health—become internet-accessible on this port. Flux's authentication was still a placeholder. Do not publish real financial data, passwords, or email credentials with this setup. For a safer test, keep port 3000 closed and forward it through SSH:

```bash
ssh -N -L 127.0.0.1:3000:127.0.0.1:3000 \
  -i ~/.ssh/flux_ed25519 "flux-vm@$VM_IP"
```

Then open `http://localhost:3000/api/health` locally while the tunnel stays open.

For production, use a static IP/domain, HTTPS via a reverse proxy or load balancer, proper authentication, systemd, and appropriate data persistence/backups. With Nginx on port 80/443 forwarding to 3000, public requests use the proxy's port; port 3000 need not be exposed.

## 9. Troubleshooting lessons

| Symptom                                           | Meaning / next action                                                                                                                    |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `-chdir ... no such file or directory`            | Use the real path, `infra/terraform`, from the project root; inside that directory omit `-chdir`.                                        |
| `invalid_grant` / `invalid_rapt`                  | Refresh ADC for Terraform. `gcloud auth login` separately refreshes CLI credentials; the accounts may differ.                            |
| Compute API `SERVICE_DISABLED`                    | Enable Compute Engine API in the correct project and wait for propagation.                                                               |
| SSH timeout / blinking cursor                     | Connectivity failure before key authentication; check public IP, VM status, network, firewall, tags, and higher-level policies.          |
| Firewall allows `tcp:20`                          | Wrong port: ordinary SSH listens on TCP **22**. A firewall does not change the server's listening port.                                  |
| Invalid tag `Specified target tags`               | That is the dropdown option; enter only `flux-vm` in the tag text field.                                                                 |
| `Permission denied (publickey)`                   | Network path works; check username, matching public/private key, OS Login setting, and metadata propagation.                             |
| Metadata fingerprint mismatch while editing       | Refresh/reopen Edit and retry; metadata changed since the page loaded. It is not an SSH-key fingerprint error. Avoid simultaneous edits. |
| Local health succeeds, public port 3000 times out | Check port-3000 ingress on the **VM's actual VPC**, matching target tag, effective firewall policies, and guest firewall.                |
| No applicable instances on firewall details       | The network or target selection may be wrong. In our case, the rule was on `default` instead of `flux-network`.                          |
| Port already in use                               | Another process owns port 3000; inspect `ss` and stop the intended old process before restarting.                                        |

To inspect SSH authentication without exposing private key contents:

```bash
ssh -v -o ConnectTimeout=10 -o IdentitiesOnly=yes \
  -i ~/.ssh/flux_ed25519 "flux-vm@$VM_IP"
```

To derive the public key from your private key and compare it with the registered key:

```bash
ssh-keygen -y -f ~/.ssh/flux_ed25519
```

Compare the key type and key data; comments may differ. Never paste the private key into a chat or Console. If public access still fails despite a matching GCP rule, check `sudo ufw status`; an enabled guest firewall may also need the relevant port allowed.

## 10. Keep Terraform and manual changes consistent

We made direct-SSH changes in Console after creating infrastructure with Terraform. Reflect the intended OS Login setting and SSH public-key metadata in the Terraform VM configuration before a later apply; otherwise it may restore the old metadata. Public-key metadata is not a private secret, but it is an access-control setting and should be reviewed.

Manually created firewall rules remain unmanaged unless deliberately represented and imported into Terraform. Do not add a same-named resource and assume Terraform will adopt it. Review every plan; never delete state or blindly apply to fix drift.

Preserve local state and the original Terraform root. Use a secure shared backend before collaborating. Stopping the VM does not necessarily eliminate disk/IP costs; plan cleanup intentionally and remember that deleting this VM deletes its auto-deleting boot disk.

## References

- [Current Console VM creation guide](https://docs.cloud.google.com/compute/docs/instances/create-start-instance#console)
- [Metadata SSH keys](https://docs.cloud.google.com/compute/docs/connect/add-ssh-keys)
- [OS Login](https://docs.cloud.google.com/compute/docs/oslogin)
- [VPC firewall rules](https://docs.cloud.google.com/firewall/docs/firewalls)
- [Terraform CLI workflow](https://developer.hashicorp.com/terraform/cli)

This walkthrough is stored in the repository's normal Markdown folder, not in GitHub's separate Wiki Git repository.
