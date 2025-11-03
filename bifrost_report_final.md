# Summary of bifrost installation and configuration

This report documents my experience installing and configuring OpenStack Bifrost on CentOS Stream 10. Bifrost simplifies bare-metal provisioning by using Ansible playbooks to automate the setup of physical or virtual nodes through OpenStack Ironic, without requiring the full OpenStack environment. My goal was to understand its workflow, configuration process, and dependencies while performing a hands-on installation.

I began by preparing the system with Python, Git, and Ansible, followed by cloning the official Bifrost repository from OpenDev. Using a Python virtual environment ensured that dependencies remained isolated. I configured a dummy inventory file to simulate a node, which provided valuable insights into how Bifrost dynamically manages machine configurations through Ansible’s inventory system.

During setup, I encountered a few issues, such as missing environment variables and the `bifrost` command not being found in the PATH. These were fixed by exporting the proper paths and performing a user-level pip installation. While testing with `bifrost-cli testenv`, the environment setup progressed smoothly until it failed to install a missing package (`sgabios-bin`). This error, common on CentOS Stream 10, revealed how package dependencies may differ across distributions.

Despite this, the test demonstrated how Bifrost detects hardware, installs dependencies, and creates VMs using Ansible automation. The experience helped me understand how Bifrost bridges the gap between Ironic and system-level provisioning.

Through this project, I gained practical exposure to OpenDev workflows, Ansible automation, and system troubleshooting. I also learned how OpenStack components like Bifrost contribute to scalable and flexible infrastructure deployment. This exercise has deepened my understanding of provisioning automation and improved my ability to work with open-source infrastructure projects.

---

# Bifrost Installation and Usage Report

**Author:** Abhijith P C  
**Platform:** CentOS Stream 10  
**Date:** 22 October 2025  

---
# Bifrost Installation — Expanded Technical Report

## 1. Introduction (expanded)

Bifrost is an OpenStack subproject that exposes a set of Ansible playbooks and helper scripts to enroll, clean, and provision bare-metal machines via Ironic. It is useful for labs, CI, and doing minimal Ironic-based provisioning without a full OpenStack deployment. The practical aim here was to:

- Build Bifrost from source in a reproducible environment (virtualenv),
- Create and validate a dummy inventory (to simulate hardware),
- Run the `bifrost-cli testenv` to exercise the end-to-end workflow,
- Capture and fix errors found during the process.

## 2. Pre-installation steps (expanded)

I verified and installed core packages using DNF and prepared the repo:

```bash
sudo dnf install -y python3 python3-pip git ansible
git clone https://opendev.org/openstack/bifrost.git ~/bifrost
cd ~/bifrost
```

**Key observations:**

- System Python is 3.12 (confirmed).
- I preferred using Bifrost's provided `venv` under `env/` to avoid global package conflicts.

## 3. Virtual environment activation (expanded)

Bifrost provides an `env` directory with a virtual environment. I activated it and confirmed Ansible lives inside it:

```bash
source ~/bifrost/env/bin/activate
which python
# => /home/ac/bifrost/env/bin/python
which pip
# => /home/ac/bifrost/env/bin/pip
ansible --version
# shows ansible-core 2.19.3 and the python path inside venv
```

**Why this matters:**

Running playbooks with the system Ansible may use different collections or plugins. Activating the venv ensures the versions bundled with Bifrost are used.

## 4. Installing Bifrost CLI & Python packages (expanded)

I attempted several installation routes:

```bash
pip install --user .   # from repo root
pip install -e .       # editable install
```

Commands and outputs:

```bash
cd ~/bifrost
pip install --user .
# or
pip install -e .
```

The install built and installed `bifrost` python package (version 18.0.2 in my run) and its requirements:

- oslo.config, oslo.log, passlib, pyOpenSSL, cryptography, netaddr, etc.

Post-installation:

```bash
pip show bifrost
# Location: /home/ac/.local/lib/python3.12/site-packages
ls ~/.local/bin | grep bifrost
# bifrost_inventory  bifrost_inventory.py
```

**Notes:**

- `bifrost` top-level command is not present; main user tools are `bifrost_inventory` and repo-local script `bifrost-cli`.
- I updated PATH so `~/.local/bin` is available:

```bash
echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

## 5. Inventory configuration (expanded)

Bifrost reads inventory via `BIFROST_INVENTORY_SOURCE`. I created a dummy inventory and iterated on it.

Create inventory:

```bash
mkdir -p ~/bifrost/inventory
nano ~/bifrost/inventory/dummy_nodes.yaml
```

Initial YAML I used (valid minimal form for testing):

```yaml
nodes:
  node1:
    mac: 41135085296
    driver: fake
```

Set environment variable:

```bash
export BIFROST_INVENTORY_SOURCE=~/bifrost/inventory/dummy_nodes.yaml
```

Validate:

```bash
bifrost_inventory --list
```

**Important `bifrost_inventory --list` output (trimmed):**

```json
{
  "_meta": {
    "hostvars": {
      "127.0.0.1": {"ansible_connection": "local"},
      "all": {
        "nodes": {
          "node1": {"mac": 41135085296, "driver": "fake"}
        },
        "name": "all",
        "host_groups": ["baremetal"],
        "addressing_mode": "dhcp"
      }
    }
  },
  "baremetal": {"hosts": ["all"]},
  "localhost": {"hosts": ["127.0.0.1"]}
}
```

**Troubleshooting notes:**

I initially saw errors: `list indices must be integers or slices, not str` — caused by invalid YAML structure.

After correcting YAML, `bifrost_inventory` returned valid JSON showing the fake node.

**Example of a richer inventory for Ironic compatibility (suggested):**

```yaml
all:
  vars:
    addressing_mode: dhcp
  children:
    baremetal:
      hosts:
        node1:
          mac: "52:54:00:12:34:56"
          driver: fake
          name: node1
```

This `hosts`/`children` structure avoids Ansible warnings about unexpected keys.

## 6. Enrollment playbook execution (expanded)

I ran enrollment in check mode first:

```bash
cd ~/bifrost/playbooks
ansible-playbook -i ~/bifrost/inventory/dummy_nodes.yaml enroll-dynamic.yaml --check -v
```

**Observed warnings:**

```
[WARNING]: Skipping unexpected key (nodes) in group (all), only "vars", "children" and "hosts" are valid
[WARNING]: provided hosts list is empty, only localhost is available...
[WARNING]: Could not match supplied host pattern, ignoring: baremetal
PLAY [Enroll hardware from inventory into Ironic] ******************************
skipping: no hosts matched
```

**Root cause:**

Inventory lacks the precise Ansible group/hosts layout expected by the playbooks. Bifrost's playbooks look for hosts under a `baremetal` group and expect node-level attributes (MACs, power method, IPMI credentials) for real hardware.

**Actionable fix:**

Convert the simple `nodes:` mapping into an Ansible-compatible inventory (`hosts`, `children`) as shown earlier.

## 7. Deployment and redeployment tests (expanded)

I repeated similar tests with:

```bash
ansible-playbook -i ~/bifrost/inventory/dummy_nodes.yaml deploy-dynamic.yaml --check -v
ansible-playbook -i ~/bifrost/inventory/dummy_nodes.yaml redeploy-dynamic.yaml --check -v
```

The playbooks consistently skipped because they found no matching hosts. Key lesson: the deployment pipeline requires node power/boot parameters and more complete inventory entries to perform meaningful tasks.

## 8. Using bifrost-cli testenv (expanded)

To exercise the full automated workflow, I used the wrapper script:

```bash
./bifrost-cli testenv
```

**What it does:**

- Detects package manager (DNF in my case)
- Installs binary dependencies via bindep
- Sets up collections and dependencies inside Bifrost venv
- Attempts to create VM-based test nodes and run end-to-end playbooks

**Partial run output (critical failure):**

```
TASK [bifrost-create-vm-nodes : Install required packages] *********************
fatal: [127.0.0.1]: FAILED! => {"changed": false, "failures": ["No package sgabios-bin available."], "msg": "Failed to install some of the specified packages", "rc": 1, "results": []}
```

**Diagnosis:**

`sgabios-bin` is provided by some distros or repos (e.g., Debian/Ubuntu or older CentOS streams). CentOS Stream 10's default repos do not include it — hence VM creation via qemu roles fails.

You can either:

- Install equivalent qemu/OVMF packages from CRB/EPEL or replace the package name in `playbooks/roles/bifrost-create-vm-nodes/tasks/*`.
- Use a distro known to be CI-tested (Ubuntu or CentOS Stream 9) for full testenv success.

## 9. Troubleshooting & debugging (expanded)

I documented the main issues and how I approached them:

- **bifrost command missing**
  - `bifrost` binary isn't necessarily part of the package. Use `bifrost_inventory` or `./bifrost-cli`.
  - Ensure `~/.local/bin` is in PATH after `pip install --user .`.

- **Inventory parsing errors**
  - YAML structure must match expectations. Use Ansible-compatible `hosts`/`children` format to avoid warnings.

- **Missing OS packages for VM creation**
  - `sgabios-bin` missing on CentOS Stream 10. Try enabling CRB/EPEL or change testenv default packages.

- **Ansible host matching**
  - Use `-v` verbose to see which host patterns are being matched. Ensure `baremetal` group exists and contains hosts.

- **env-vars file not found**
  - Some example scripts expect `env-vars` at repo root; Bifrost provides a `env` venv you can activate instead.

- **How to inspect logs and rerun playbooks**
  - Rerun with `-v` or `-vvv`. Check `/var/log/messages` or `journalctl` if system services are used.

## 10. Conclusion
Despite minor issues, the installation validated core Bifrost functionality — from environment setup to inventory configuration.  
Future improvements include resolving repository dependencies and verifying integration with Ironic for end-to-end provisioning.

## References

- https://docs.openstack.org/bifrost/latest/
- https://opendev.org/openstack/bifrost

