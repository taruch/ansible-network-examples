# **Network OS Image Deployment Strategies via Ansible Automation Platform**

Deploying large network OS images (often 1GB to 3GB+) poses specific challenges when utilizing Ansible Automation Platform (AAP). This document outlines the available strategies for transferring large binaries to network devices (such as Cisco IOS and Arista EOS), ranging from constrained local execution node SCP pushes to modern, scalable cloud-storage architectures.

## **Strategy 1: Direct Push Staged on a Single Execution Node (SCP)**

This approach is utilized when constrained strictly to pushing files through AAP via SCP/SFTP, without the ability to stand up internal web gateways. To bypass the complexities of SMB/CIFS mounts and Python-native SMB tasks, the image is manually transferred to the underlying Execution Node OS host, then exposed to the ephemeral Execution Environment pod.

### **Implementation Steps**

> 1. **Stage the Image on the Host:** Transfer the OS binary directly to a staging directory on the Execution Node host OS via SCP.  
> 2. **Expose Path in Controller UI:** Within the Automation Controller UI (Settings \-\> Job Settings), add the directory under "Paths to expose to isolated jobs" using standard podman volume syntax (e.g., /var/images:/var/images:ro).  
> 3. **Execute Playbook:** The playbook accesses the file locally from the mounted path and pushes it to the switch.

### **Playbook Example**

`---`  
`- name: Push Staged OS Image to Switch`  
  `hosts: eos_switches`  
  `gather_facts: false`  
  `vars:`  
    `image_name: "EOS-4.30.1F.swi"`  
    `ansible_command_timeout: 1800`

  `tasks:`  
    `- name: SCP image from local host path to switch`  
      `ansible.netcommon.net_put:`  
        `src: "/var/images/{{ image_name }}"`  
        `dest: "flash:{{ image_name }}"`  
        `protocol: scp`

### **Pros and Cons**

> * **Pros:** Bypasses Git repository bloat, avoids AAP memory limits, and sidesteps SMB/CIFS mount complexities. Keeps files strictly within the SSH channel.  
> * **Cons:** Requires manual management of host disk space. Utilizing multi-node Instance Groups would require manual file syncing across all hosts. Heavy load on network switch control plane during transfer. Risk of SSH timeouts during multi-gigabyte transfers. In Ansible Automation Platform (AAP), configuring **"Paths to expose to isolated jobs"** in the Automation Controller Job Settings is a **global, system-wide configuration**; it **cannot be scoped to a specific Execution Node** within the Controller UI settings.

## **Strategy 2: Direct CIFS Transfer via Execution Environment (EE)**

This approach fetches the image from the existing CIFS share directly into the ephemeral Execution Environment (EE) container's local storage at runtime using CIFS tools or SMB protocols, and then pushes it to the network switch. This eliminates the need for manual host-level staging.

### **Implementation Steps**

> 1. **EE Customization:** Build a custom EE image that includes cifs-utils or the Python smbprotocol dependencies alongside the smb.smb Ansible collection.  
> 2. **Fetch to Local Temp:** Use an Ansible task to pull the binary from the CIFS share into the EE pod's ephemeral /tmp directory.  
> 3. **Execute SCP Push:** Use ansible.netcommon.net\_put to transfer the file from the EE's /tmp directory to the switch via SCP.  
> 4. **Storage Cleanup:** Utilize an always block to guarantee deletion of the local file on the EE post-transfer to prevent storage exhaustion on the AAP runner nodes.

### 

### **Playbook Example**

`---`  
`- name: Transfer Image via EE CIFS Fetch`  
  `hosts: eos_switches`  
  `gather_facts: false`  
  `vars:`  
    `image_name: "EOS-4.30.1F.swi"`  
    `local_stage_path: "/tmp/{{ image_name }}"`  
    `ansible_command_timeout: 1800`

  `tasks:`  
    `- name: Fetch OS image from CIFS share to EE /tmp`  
      `smb.smb.get:`  
        `smb_host: "cifs-server.corp.local"`  
        `share: "network_images"`  
        `src: "arista/{{ image_name }}"`  
        `dest: "{{ local_stage_path }}"`  
        `username: "{{ cifs_username }}"`  
        `password: "{{ cifs_password }}"`  
      `delegate_to: localhost`

    `- name: Push image to Arista Switch Flash`  
      `ansible.netcommon.net_put:`  
        `src: "{{ local_stage_path }}"`  
        `dest: "flash:{{ image_name }}"`  
        `protocol: scp`

    `- name: Clean up temporary image file on EE`  
      `ansible.builtin.file:`  
        `path: "{{ local_stage_path }}"`  
        `state: absent`  
      `delegate_to: localhost`  
      `always: true`

### **Pros and Cons**

> * **Pros:** Eliminates manual file staging on the Execution Node host. Integrates natively with existing CIFS infrastructure without host-level mounts. Scales easily across multiple Execution Nodes.  
> * **Cons:** Incurs a "double network penalty" (fetching from CIFS to AAP, then pushing from AAP to the switch). Can cause high ephemeral storage and memory impact on the EN.

## **Strategy 3: Cloud Object Storage (Azure Blob or Amazon S3 \- Device-Initiated Pull)**

This is the industry best practice architecture. Instead of pushing the binary through the AAP SSH tunnel, Ansible orchestrates the transfer by instructing the network switch to download the binary directly from Azure Blob Storage over HTTPS. This offloads the heavy lifting from the AAP infrastructure entirely.

### **The Architectural Pitch**

> * **Reliability:** Transfers over HTTPS handle network fluctuations natively and are optimized for streaming large files, eliminating the 15-30 minute fragile SSH pipe required by SCP.  
> * **Transfer Speeds:** Network switch HTTP client stacks process downloads 3x to 5x faster than SCP/SFTP over SSH, because SCP is aggressively throttled to protect device CPU.  
> * **Zero Maintenance:** Azure Blob Storage offers infinite scaling and automated Lifecycle Management Policies (e.g., auto-delete after 30 days). No more manual host storage management or cross-node synchronization.  
> * **Security & Auditing:** Storage accounts do not need to be public. AAP can dynamically generate time-limited Shared Access Signature (SAS) tokens valid only for the duration of the job, resulting in highly granular audit logs within Azure.

### **Playbook Example**

`---`  
`- name: Upgrade Arista EOS via Azure Blob HTTPS`  
  `hosts: eos_switches`  
  `gather_facts: false`  
  `vars:`  
    `image_name: "EOS-4.30.1F.swi"`  
    `blob_url: "[blob-storage-endpoint]/firmware/{{ image_name }}"`

  `tasks:`  
    `- name: Command switch to pull image over HTTPS directly`  
      `arista.eos.eos_command:`  
        `commands:`  
          `- command: "copy {{ blob_url }}?{{ sas_token }} flash:{{ image_name }}"`  
      `async: 1200`  
      `poll: 15`

    `- name: Verify checksum`  
      `arista.eos.eos_command:`  
        `commands:`  
          `- command: "verify /sha512 flash:{{ image_name }}"`  
      `register: checksum_output`

## **Strategy 4: Internal Web Gateway / Artifact Repository**

If cloud storage is not feasible due to internet restrictions, an internal Artifact Repository (like JFrog Artifactory, Sonatype Nexus) or a simple Nginx/Apache frontend placed over the existing CIFS share achieves the same "Device-Initiated Pull" benefits as Azure Blob Storage, but kept entirely on-premises.

## **Strategy 5: Device-Initiated SCP/SFTP Pull/Copy (RHEL Server or Localized Staging Switch)**

This method removes the AAP Execution Environment (EE) from the data transfer path entirely without requiring an HTTP/HTTPS infrastructure. Instead of pushing the file through Ansible, the playbook instructs the network device to copy or pull the OS image directly from an external RHEL-based SCP/SFTP server.

Additionally, this strategy allows for localized transfers in bandwidth-constrained environments: a single switch or router at a remote site can be loaded with the image and configured as a temporary SCP server, allowing all other devices at that branch to pull the image locally across the LAN.

#### **Implementation Steps**

* **Host the Image:** Stage the OS binary on a central RHEL SCP server, or transfer it to a single "staging" switch at a remote site and enable the SCP server feature on that device.  
* **Configure SSH Keys/Credentials:** Ensure the destination network devices have the appropriate SSH keys or credential configurations to seamlessly authenticate to the SCP server without interactive prompts blocking the transfer.  
* **Adjust Playbook Timeouts:** Because the switch control plane handles the transfer and AAP is simply waiting for the command to return, you must configure extended timeouts (`ansible_command_timeout`) or use asynchronous tasks (`async` and `poll`) to prevent the SSH control session from dropping while the large file copies.  
* **Execute Copy/Pull Command:** Use standard network command modules or (`ansible.netcommon.cli_command`) to trigger the device's native copy command.  
* Remember to no\_log any credentials if applicable..

**Playbook Example**

`---`  
`- name: Upgrade Arista EOS via Device-Initiated SCP Pull`  
  `hosts: eos_switches`  
  `gather_facts: false`  
  `vars:`  
    `image_name: "EOS-4.30.1F.swi"`  
    `# SCP server could be a RHEL host or a localized branch switch`  
    `scp_server: "10.0.50.100"`  
    `scp_user: "ansible_svc"`

  `tasks:`  
    `- name: Command switch to pull image via SCP directly`  
      `arista.eos.eos_command:`  
        `commands:`  
          `- command: "copy scp://{{ scp_user }}@{{ scp_server }}/images/{{ image_name }} flash:{{ image_name }}"`  
      `# Use async or timeout to prevent AAP from timing out during the long transfer`  
      `async: 1800`  
      `poll: 15`

    `- name: Verify image checksum`  
      `arista.eos.eos_command:`  
        `commands:`  
          `- command: "verify /sha512 flash:{{ image_name }}"`  
      `register: checksum_output`

## **Architecture Comparison Matrix**

| Feature | Staged Host Node (SCP Push) | Direct CIFS Fetch via EE (SCP Push) | Azure Blob Storage (HTTPS Pull)   |
| :---- | :---- | :---- | :---- |
| **Transfer Protocol** | SCP over SSH | CIFS to EE, then SCP to Switch | Direct HTTPS |
| **AAP Infrastructure Workload** | Heavy (Streams 2GB+ file per device) | Very Heavy (Downloads to EE pod, then streams) | Near Zero (Sends a single instruction command) |
| **Average Transfer Time (2GB)** | 15 to 30 minutes | 20 to 40+ minutes (Double network hop) | 3 to 6 minutes |
|  |  |  |  |
| **Storage Lifecycle Management** | Manual cron jobs on RHEL host | Requires strict playbook cleanup tasks | Automated Cloud Lifecycle Policies |
| **Multi-Node Scaling (Instance Groups)** | Requires manual sync across nodes | Native (Runs wherever the EE pod lands) | Works instantly; accessible globally |
| **Security & Audit** | Local file permissions on Execution Node | Requires SMB credentials stored in AAP Vault | Time-limited SAS tokens & Centralized Azure Logs |

## 