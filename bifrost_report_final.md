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

## 1. Introduction
Bifrost is an OpenStack project that provides a collection of Ansible playbooks for deploying physical machines using **Ironic** without needing the full OpenStack environment.  
It automates provisioning through a lightweight setup suitable for bare-metal or virtual environments.

The goal of this task was to:
- Install Bifrost from source.  
- Configure the inventory for test nodes.  
- Run Bifrost environment setup and validate installation.  

---

## 2. Pre-Installation Steps
Before starting, the system was updated, and essential dependencies were installed:

```bash
sudo dnf install -y python3 python3-pip git ansible
```

Checked versions:
- **Python:** 3.12  
- **Pip:** 23.3.2  
- **OS:** CentOS Stream 10  

Cloned the Bifrost repository:
```bash
git clone https://opendev.org/openstack/bifrost.git
cd bifrost
```

---

## 3. Virtual Environment Setup
Activated the virtual environment provided by Bifrost:
```bash
source ~/bifrost/env/bin/activate
```

Verified that the environment prefix appeared:
```
(env) ac@linuxlabnew:~/bifrost$
```

---

## 4. Inventory Configuration
Created a dummy inventory to simulate bare-metal nodes for testing:

```bash
nano ~/bifrost/inventory/dummy_nodes.yaml
```

**Contents:**
```yaml
nodes:
  node1:
    mac: 41135085296
    driver: fake
```

Then exported the inventory source:
```bash
export BIFROST_INVENTORY_SOURCE=~/bifrost/inventory/dummy_nodes.yaml
```

To validate the configuration:
```bash
bifrost_inventory --list
```

**Output (JSON):**
```json
{
  "_meta": {
    "hostvars": {
      "127.0.0.1": {
        "ansible_connection": "local"
      },
      "all": {
        "nodes": {
          "node1": {
            "mac": 41135085296,
            "driver": "fake"
          }
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

This confirmed that the dummy node inventory was parsed correctly.

---

## 5. Installing Bifrost CLI
Attempted to activate environment variables:
```bash
source ~/bifrost/env-vars
```
This failed (file missing), so installation was done manually using pip:

```bash
cd ~/bifrost
pip install --user .
```

Installed successfully, but the `bifrost` command wasn’t found in PATH.  
Checked the installation location:
```bash
pip show bifrost
```
It was installed under:
```
/home/ac/.local/lib/python3.12/site-packages
```

Added to PATH:
```bash
echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

Verified available commands:
```bash
ls ~/.local/bin | grep bifrost
bifrost_inventory
bifrost_inventory.py
```

---

## 6. Test Environment Setup
Ran the built-in test environment:
```bash
./bifrost-cli testenv
```

This command:
- Detected the package manager (dnf)
- Installed dependencies (Ansible, Bindep)
- Attempted to create test VMs for provisioning

**Partial Playbook Output:**
```
TASK [bifrost-create-vm-nodes : Install required packages]
fatal: [127.0.0.1]: FAILED! => {"msg": "No package sgabios-bin available."}
```

The failure occurred because the `sgabios-bin` package was unavailable in CentOS Stream 10 repositories.  
This step is optional for virtual environments and can be fixed by skipping or replacing that dependency.

---

## 7. Troubleshooting Notes
- The CLI (`bifrost`) binary wasn’t installed in the global PATH — it’s available only in the virtual environment or under `.local/bin`.
- Some scripts like `bifrost-install` are deprecated in recent versions.
- The `sgabios-bin` dependency may need to be manually installed or skipped for successful VM testing.

---

## 8. Observations and Learning
- Bifrost uses **Ansible playbooks** to manage bare-metal nodes without needing full OpenStack deployment.
- **Dummy nodes** help simulate hardware for testing automation logic.
- Dependency management and environment activation are crucial for proper CLI availability.
- While installation succeeded, VM creation failed due to missing system packages — highlighting the need for environment-specific adjustments.

---

## 9. Conclusion
Despite minor issues, the installation validated core Bifrost functionality — from environment setup to inventory configuration.  
Future improvements include resolving repository dependencies and verifying integration with Ironic for end-to-end provisioning.

---

## 10. References
- Official Documentation: [https://docs.openstack.org/bifrost/latest/](https://docs.openstack.org/bifrost/latest/)
- OpenDev Repository: [https://opendev.org/openstack/bifrost](https://opendev.org/openstack/bifrost)
