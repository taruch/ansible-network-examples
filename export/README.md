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

