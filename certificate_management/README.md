This project requires a RHEL-9 host called `ca-host` in the workshop inventory, which
contains the network devices in the workshop.  This host is used to generate certificates
for the network devices.  If you use another project, such as Ansible Product-Demos 
(github.com/ansible/product-demos) to create the VM, it will be located in the Demo inventory
and won't be usable in this demo; this will require you to manually create the inventory host
in the workshop inventory.

Create the project "Ansible Network Examples"
 - Name: Ansible Network Examples
 - Organization: Default
 - Source Control Type: Git
 - Source Control URL: https://github.com/taruch/ansible-network-examples.git
Save, and wait for the job to sync

Create the Setup Job Template "Ansible Network Examples Setup"
 - Name: Setup - Ansible Network Examples
 - Inventory: Workshop Inventory
 - Project: Ansible Network Exampels
 - Playbook: controller_setup.yml
 - Credentials: Red Hat Ansible Automation Platform / Controller Credential
LAUNCH!

Update the credential	"CA Host Credential"
 - Host: Host for the CA host (FQDN or IP)
 - Username: ca-admin
 - Password: password for CA host

Update the credential "ServiceNow ITSM"
 - Password: password for PSA ServiceNow instance

Update the hostname for the ca-host in the workshop inventory
 - Change the "Change Me" to be the ca host ip or FQDN

=====CISCO=====
Show the rtr beforehand - no trustpoints, no certificates:
sho crypto pki certificates verbose

Show the workflow run with below survey answers, it should run without issue.
  "trustpoint": "CACERT",
  "renewal_days": "12",
  "minumum_days": "10"
then run it again (from the same job), this time it should succeed and stop at checking the certificate.
Run the workflow again but this time change the min days to 15

Show the rtr after with trustpoints and certificates:
sho crypto pki certificates verbose

=====JUNIPER=====
Update the number of date variable attributes - done - check
Add the CA credential to the juniper 

