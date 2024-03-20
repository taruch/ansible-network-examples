Go to your RHDP instance and create a new project from 
https://github.com/taruch/ansible-network-examples.git


Create a Setup Template
Add the controller credential to the setup template


Run the Setup Template.  This will add components to the controller.

Go and update passwords:\
	Gitlab Ansible_EE Token\
    ServiceNow ITSM
    CA Host Credential

Add the CA Host Credential to the Cisco / Trustpoint Create (issue)

Update the inventory item for the CA host

Create a machine credential for the CA host and add that to the 
CA / Generate Certificate template (issue)

Add the CA Host Credential the Cisco / Trustpoint Import Host Certificate (issue)

Juniper / Import CA Certificate to CA Profile
Juniper / Import Host Certificate to CA Profile
These both have the wrong playbook


Add the CA Cert to the CA Profile
Fix this - need to add the CA Host Credential to this playbook because it needs the CA_PASSWORD
ALSO
This is failing the first time you connect to a host because it asks if you are sure you want to connect...