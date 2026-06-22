# Edge Image Builder: RKE2 & Rancher Deployment

This repository contains the configuration files and manifests required to build a custom Edge Image Builder (EIB) ISO. The resulting ISO deploys a highly available, airgapped RKE2 Kubernetes cluster with Rancher Manager, NeuVector 5.5.2, and CloudNativePG pre-configured.

It also deploys Harbor, Rancher Logging, Monitoring (Prometheus/Grafana/Alertmanager), Compliance Operator and NeuVector UI extension out of the box.

---

## 📁 Repository Structure

| File / Directory | Description |
| :--- | :--- |
| `eib-iso-definition.yaml.j2` | The Jinja2 template file defining the ISO build parameters with secret placeholders. |
| `generate_eib.yml` | The Ansible playbook used to render the final YAML definition file. |
| `install.txt` | Instructions and CLI commands for automated VM creation on Linux using virt-manager (QEMU/KVM). |
| `kubernetes/manifests/rke2-api-service.yaml` | Access point for the highly available Kubernetes API endpoint (supports the 3 control plane nodes). |
| `kubernetes/manifests/rke2-ingress-config.yaml` | Ingress definition to access Rancher Manager using the HA API endpoint IP. |
| `kubernetes/manifests/rke2-api-sharing.yaml` | Custom job allowing the Kubernetes API and Rancher Manager to share the same HA IP address. |
| `kubernetes/helm/values/rancher-values.yaml` | Rancher Helm chart values (defines URL, internal registry, and bootstrap admin password). |
| `kubernetes/helm/values/neuvector-crd.yaml` | NeuVector Custom Resource Definitions (CRDs) Helm chart values. |
| `kubernetes/helm/values/neuvector-values.yaml` | NeuVector Helm chart values. |
| `kubernetes/helm/values/kubernetes-csi-driver-nfs-values.yaml` | NFS provisioner setup to connect to the shared storage for NeuVector and PostgreSQL PVCs. |
| `network/rke2-*.demo.com.yaml` | Node-specific network configuration setups for `rke2-1`, `rke2-2`, and `rke2-3`. |

---

## 🛠️ Prerequisites

* **Compute:** A host machine running Linux with Podman installed. The machine building the ISO requires **at least 8GB of RAM**.
* **Base OS:** SUSE Linux Enterprise (SLE) Micro 6.2 Base ISO (`SL-Micro.x86_64-6.2-Base-SelfInstall-GM.install.iso`).
* **Automation:** Ansible installed on your build machine to manage secrets and render templates.

---

## ⚙️ Configuration & Setup

### 1. Prepare the Build Environment

First, clone the repository locally and set up your working directories:

```bash
# Clone the repository
git clone <your-repo-url> edge-dc-eib
cd edge-dc-eib

# Set the config directory to your cloned folder
export CONFIG_DIR=$(pwd)

# Create the base-images directory
mkdir -p $CONFIG_DIR/base-images

# Copy your SLE Micro ISO into the base-images directory
cp /path/to/SL-Micro.x86_64-6.2-Base-SelfInstall-GM.install.iso $CONFIG_DIR/base-images/slemicro.iso
```

### 2. Protect Secrets from Git (.gitignore)

To ensure your plaintext secrets and the generated configuration file are never committed to GitHub, create a `.gitignore` file in the root of your repository:

```bash
touch .gitignore
```

Add the following entries to your `.gitignore` file:

```text
# Ignore the local unencrypted secrets file
vars/secrets.yml

# Ignore the rendered configuration file containing plain secrets
eib-iso-definition.yaml
```

### 3. Create the Secrets File

Create a variables directory and your `secrets.yml` file:

```bash
mkdir -p $CONFIG_DIR/vars
touch $CONFIG_DIR/vars/secrets.yml
```

Populate `$CONFIG_DIR/vars/secrets.yml` with your specific credentials:

```yaml
scc_registration_code: "YOUR_60_DAY_EVAL_KEY"
appco_repo_username: "YOUR_APPCO_USERNAME"
appco_repo_password: "YOUR_BASE64_ENCODED_APPCO_PASSWORD"
```

#### Credential Details:
* **SCC Registration (`scc_registration_code`):** Obtained from the SUSE Customer Center (SCC). This provides a 60-day evaluation key for registering your SLE Micro instances.
* **AppCo Authentication (`appco_*` & `artifact_registry_*`):** The pull credentials (username and base64-encoded secret token) required to download assets from internal SUSE/Rancher registry endpoints and Helm charts.

*(Optional)* If you want to safely track this file in Git instead of ignoring it completely, you can encrypt it using Ansible Vault:
```bash
ansible-vault encrypt vars/secrets.yml
```

### 4. Render the EIB Configuration File

To convert the template `eib-iso-definition.yaml.j2` into a usable `eib-iso-definition.yaml` file, you will use an Ansible playbook. 

Create a playbook named `generate_eib.yml` in your repository root:

```yaml
---
- name: Generate EIB definition from template
  hosts: localhost
  connection: local
  gather_facts: no
  vars_files:
    - vars/secrets.yml
  tasks:
    - name: Render the EIB ISO definition
      ansible.builtin.template:
        src: eib-iso-definition.yaml.j2
        dest: eib-iso-definition.yaml
```

Execute the playbook to generate your configuration file:

* **If your `secrets.yml` is unencrypted (plain text):**
  ```bash
  ansible-playbook generate_eib.yml
  ```
* **If you encrypted `secrets.yml` with Ansible Vault:**
  ```bash
  ansible-playbook generate_eib.yml --ask-vault-pass
  ```

---

## 🕒 OS Users

The default root encrypted password inside the template is hardcoded to `eib`, allowing console (non-SSH) login out-of-the-box. SSH for root is disabled by default. If you add standard users to the user block, they will be granted SSH access. 
*Note: You can generate new encrypted passwords using `openssl passwd -6 $PASSWORD`.*

---

## 🌐 Infrastructure & Networking Setup (QEMU/KVM)

The default configuration expects a specific QEMU NAT network (`host-nat` on `172.16.33.0/24`). If you use your own infrastructure, ensure you update the IP addresses in the configuration files accordingly. 

### Host Network Definition (`host-nat`)

```xml
<network connections='4'>
  <name>host-nat</name>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='virbr0' stp='on' delay='0'/>
  <mac address='52:54:00:8f:8c:6c'/>
  <dns enable='no'/>
  <ip address='172.16.33.1' netmask='255.255.255.0'>
  </ip>
</network>
```

### Utility Server (`utility.demo.com`)

The environment relies on a self-contained utility node (`172.16.33.2`) to provide DNS (BIND) and NFS storage for the airgapped cluster.

#### DNS Configuration (BIND)

**Forward Zone (`/var/lib/named/demo.com.zone`):**
```text
$ORIGIN .
$TTL 900	; 15 minutes
demo.com		IN SOA	utility.demo.com. hostadmin.demo.com. (
				642        ; serial
				900        ; refresh (15 minutes)
				300        ; retry (5 minutes)
				604800     ; expire (1 week)
				900        ; minimum (15 minutes)
				)
$TTL 3600	; 1 hour
			NS	utility.demo.com.
$ORIGIN demo.com.
harbor-dev		CNAME	rancher-dev
juice-shop		CNAME	rancher-dev
log4shell		CNAME	rancher-dev
$TTL 600	; 10 minutes
neuvector-dev		CNAME	rancher-dev
$TTL 3600	; 1 hour
rancher-dev		A	172.16.33.10
$TTL 900	; 15 minutes
rke2-1			A	172.16.33.47
rke2-2			A	172.16.33.46
rke2-3			A	172.16.33.45
$TTL 900	; 15 minutes
utility			A	172.16.33.2

```

**Reverse Zone (`/var/lib/named/33.16.172.in-addr.arpa.zone`):**
```text
$ORIGIN 33.16.172.in-addr.arpa.
@                     900       IN  SOA           utility.demo.com. hostadmin 594 900 300 604800 900
@                     3600      IN  NS            utility.demo.com.
2                     900       IN  PTR           utility.demo.com.
47                    900       IN  PTR           rke2-1.demo.com.
46                    900       IN  PTR           rke2-2.demo.com.
45                    900       IN  PTR           rke2-3.demo.com.
10                    900       IN  PTR           rancher-dev.demo.com.
```

#### NFS Storage Configuration

The NFS server automatically provisions PVCs via the NFS CSI driver. Ensure UDP is enabled.

**`/etc/exports.d/storage.exports`:**
```text
/mnt/storage/rancher-dev 172.16.33.0/24(rw,sync,no_subtree_check,no_root_squash,insecure)
```

**`/etc/sysconfig/nfs` (Key Variables):**
```text
NFS3_SERVER_SUPPORT="yes"
NFS4_SUPPORT="yes"
NFS_SERVER_UDP="yes"
USE_KERNEL_NFSD_NUMBER="4"
```

---

## 🏗️ Building the ISO

Once your `eib-iso-definition.yaml` file has been successfully rendered by Ansible, you can build the deployment image using Podman.

> **Note:** The initial build takes approximately 30-40 minutes as it pulls down all required resources and containers. Subsequent runs use cached layers and finish significantly faster.

```bash
podman run --rm -it --privileged \
  -v $CONFIG_DIR:/eib \
  [registry.suse.com/edge/3.6/edge-image-builder:1.3.3.1](https://registry.suse.com/edge/3.6/edge-image-builder:1.3.3.1) build \
  --definition-file eib-iso-definition.yaml
```

This command will outputs a flashable or mountable `eib-image.iso` inside your directory.

---

## 🚀 Deploying the Cluster

You can start the nodes using the command line via `virt-install` (reference `install.txt` for the full node list parameters. Define VMFOLDER variable to point to the folder with the ISO file and VM files). 

> **Crucial:** Ensure the MAC addresses in your KVM configurations match the exact MAC addresses defined in your node network YAML files.

**Start `rke2-1` First:** This host is flagged as the cluster initializer. Let it fully boot up and establish the cluster before turning on `rke2-2` and `rke2-3` to ensure proper cluster discovery.

*** rke2-1 ***
```bash
virt-install \
  --connect qemu:///system \
  --name rke2-1 \
  --memory 12288 \
  --vcpus 8 \
  --cpu host-passthrough \
  --disk path=$VMFOLDER/qcows/rke2-1.qcow2,size=80,format=qcow2,bus=virtio \
  --os-variant slem6.2 \
  --network network=host-nat,model=virtio,mac=34:8A:B1:4B:16:E1 \
  --cdrom $VMFOLDER/eib-image.iso \
  --graphics vnc \
  --boot uefi,hd,cdrom \
  --noautoconsole
```

*** rke2-2 ***
```bash
virt-install \
  --connect qemu:///system \
  --name rke2-2 \
  --memory 12288 \
  --vcpus 8 \
  --cpu host-passthrough \
  --disk path=$VMFOLDER/qcows/rke2-2.qcow2,size=80,format=qcow2,bus=virtio \
  --os-variant slem6.2 \
  --network network=host-nat,model=virtio,mac=34:8A:B1:4B:16:E2 \
  --cdrom $VMFOLDER/eib-image.iso \
  --graphics vnc \
  --boot uefi,hd,cdrom \
  --noautoconsole
```

*** rke2-3 ***
```bash
virt-install \
  --connect qemu:///system \
  --name rke2-3 \
  --memory 12288 \
  --vcpus 8 \
  --cpu host-passthrough \
  --disk path=$VMFOLDER/qcows/rke2-3.qcow2,size=80,format=qcow2,bus=virtio \
  --os-variant slem6.2 \
  --network network=host-nat,model=virtio,mac=34:8A:B1:4B:16:E3 \
  --cdrom $VMFOLDER/eib-image.iso \
  --graphics vnc \
  --boot uefi,hd,cdrom \
  --noautoconsole
```

---

## ✅ Post-Deployment & Verification

1.  **Access Rancher:** After approximately 10-15 minutes, the environment will settle. Access the Rancher console at:
    * **URL:** `https://rancher-dev.demo.com`
    * **Bootstrap Password:** `admin`

2.  **Verify Workloads:** NeuVector and CloudNativePG are configured to automatically instantiate during installation.

3.  **Deploy a Sample Database:** Validate that cluster storage and CloudNativePG operators are running correctly by deploying a 3-replica PostgreSQL test database:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: cluster-example
spec:
  instances: 3
  storage:
    size: 1Gi
```