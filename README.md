Directions for running playbooks in this repository

The NXOS playbooks are configured to use the Cisco dCloud (https://dcloud2-rtp.cisco.com)
`Cisco Datacenter Automation with Ansible v1` environment.  To use this environment you will
need to have a dcloud account, and then look in the catalog for the specified dcloud demo
environment.

Do a git fork to your git repo from github.com/taruch/ansible-network-examples and set up a token for the repo.

To use the gitops functions, you will need to specify the following variables in the group_vars/all file:
- git_url
- github_token


Order of Operations:
 - Connect to the dcloud environment using the credentials provided in the "details" section
 openconnect --user={{ some user name }} --passwd-on-stdin --server=dcloud-rtp-anyconnect.cisco.com

 - nxos_gather_data.yml - this playbook will pull the current configuration off the switch and will 
 create a branch of your repo, and submit a git push to the git_url using the github_token. You can click
 on branches and select the new branch to look at the differences between the config on the switch and 
 the config in the repo.

 - nxos_base_configuration.yml - this will take the configuration in the 
