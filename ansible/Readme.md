## Ansible usage
- Test inventory file with to view groups and variable inheritance within the file `ansible-inventory -i inventory.yaml --list`
- Test ssh connection with all defined hosts `ansible -i inventory.yaml all -m ping`