## Ansible usage notes
### Ansible configuration tests
- Test inventory file to view groups and variable inheritance within the file `ansible-inventory -i inventory.yaml --list`
- Test ssh connection with all defined hosts `ansible -i inventory.yaml all -m ping`
### Ansible ad-hoc commands
They achieve a form of idempotence by checking the current state before they begin and doing nothing unless the current state is different from the specified final state. 
The default module for the ansible command-line utility is the [*ansible.builtin.command*](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/command_module.html#command-module) module.
By default, Ansible uses only five simultaneous processes. If you have more hosts than the value set for the fork count, it can increase the time it takes for Ansible to communicate with the hosts. For example, to reboot the `atlanta` servers with 10 parallel forks, connect as specific user and become remotely as root for privelege escalation: `$ ansible atlanta -a "/sbin/reboot" -f 10 -u username --become [--ask-become-pass]`. If you add `--ask-become-pass` or `-K`, Ansible prompts you for the password to use for privilege escalation

Some examples for execution ad-hoc commands on nodes: 
- check disk utilization  `ansible -i inventory.yaml all -a "df -h"`
- transfer files `ansible -i inventory.yaml server1 -m ansible.builtin.copy -a "src=~/test.txt dest=/home/vagrant/test1.txt"`
- to ensure a package is installed without updating it, however if not installed ansible install the package `ansible -i inventory.yaml server1 -m ansible.builtin.apt -a "name=ssh state=present" --become`
- gathering facts `ansible -i inventory.yaml server1 -m ansible.builtin.setup`
- ensure service is started `ansible -i inventory.yaml server1 -m ansible.builtin.service -a "name=ssh state=started"`
- use the check mode `ansible -i inventory.yaml server1 -m copy -a "content=foo dest=/vagrant/bar.txt" -C`. Using `-C` or `--check` will only verify not doing any changes 