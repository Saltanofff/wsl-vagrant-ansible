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

### Ansible variables
Reference to variable – `{{ variable_name }}`.  Variables may be defined in inventory, in external YAML files, special standard locations and during runtime. Types of variables:
- string
- int
- bool
- list
- dictionary
```shell
- name: "Adding user {{ item.name }} using default var loop name - item"
 #quote strings in YAML if they contain Jinja ({{ }}), especially in `name`, `command`, or `msg` fields.
  ansible.builtin.user:
    name: "{{ item.name }}" # "{{ user.name}}" - e.g. when to use custom var
    uid: "{{ item.uid }}" # "{{ user.uid}}"
    shell: "{{ item.shell }}" # "{{ user.shell}}"
    state: present
  loop:
    - { name: "alice", uid: 1010, shell: "/bin/bash" }
    - { name: "bob", uid: 1011, shell: "/bin/zsh" }
  #loop_control:
  #  loop_var: user
```
- check and test the variable by `ansible -i inventory.yaml all -m debug -a "var=variable_name"`  
#### Special variables 
##### Magic variables 
These variables cannot be set directly by the user; Ansible will always override them to reflect internal state. Examples:
- `hostvars` - a dictionary/map with all the hosts in inventory and variables assigned to them, including `ansible_facts`
- `groups` - a dictionary/map with all the groups in inventory and each group has the list of hosts that belong to it

Example on usage magic variables to configure a database connection for your app using the IP address of the DB server
```sh

    - name: Configure app with DB IP
    template:
        src: app.conf.j2
        dest: /etc/myapp/app.conf
    vars:
        db_host_ip: "{{ hostvars[groups['db_servers'][0]]['ansible_host'] }}" 
        #get data directly from ansible connection variable
        db_os: "{{ hostvars[groups['db_servers'][0]]['ansible_facts']['distribution'] }}" 
        #get data from ansible_facts
```

##### Facts
When you run a playbook, Ansible executes tasks for each host specified in the `hosts` directive. For each host, it gathers facts and stores them in the `ansible_facts` dictionary under the corresponding `inventory_hostname` magic variable. If you need to access facts for a host other than the one currently being managed, you can use the `hostvars` magic variable.
```sh
hostvars['webserver1']['ansible_facts']['distribution']
```
##### Connection variables
Connection variables are normally used to set the specifics on how to execute actions on a target Examples:
`ansible_host` - The ip/name of the target host to use instead of `inventory_hostname`.
`ansible_user` - The user Ansible ‘logs in’ as
### Ansible facts
Facts gathering is a default zero task of any Ansible execution but may be turned off if necessarily. Variable `ansible_facts` contains host's system related data. By default, you can also access some Ansible facts as top-level variables with the `ansible_` prefix.
Disabling facts may particularly improve performance in push mode with very large numbers of systems:
```sh
- hosts: whatever
  gather_facts: false
```
- get facts on the local machine `ansible localhost -m ansible.builtin.setup -c local`
- get facts from the remote host `ansible -i inventory.yaml server1 -m setup -a "filter=ansible_hostname"`

### Ansible playbooks
At a minimum, each play defines two things - the managed nodes to target and at least one task to execute.
By default, Ansible executes each task in order, one at a time, against all machines matched by the host pattern. Each task executes a module with specific arguments. When a task has executed on all target machines, Ansible moves on to the next task. You can use strategies to change this default behavior. Within each play, Ansible applies the same task directives to all hosts. If a task fails on a host, Ansible takes that host out of the rotation for the rest of the playbook.
You may want to verify your playbooks to catch syntax errors and other problems before you run them. The ansible-playbook command offers several options for verification, including `--check`, `--diff`, `--list-hosts`, `--list-tasks`, and `--syntax-check`. 
You can use `ansible-lint` for detailed, Ansible-specific feedback on your playbooks before you execute them.

- test playbook in check mode `ansible-playbook --check playbook.yaml`
- test syntax with linting `ansible-lint verify-apache.yml`. The ansible-lint default rules [*page*](https://ansible.readthedocs.io/projects/lint/rules/) describes each error.


### Ansible lookup plugins
Lookup plugins are an Ansible-specific extension to the Jinja2 templating language. You can use lookup plugins to access data from outside sources (files, databases, key/value stores, APIs, and other services) within your playbooks. You can use lookup plugins anywhere you can use templating in Ansible: in a play, in variables file, or a Jinja2 template for the template module.
You can control how errors behave in all lookup plugins by setting errors to `ignore`, `warn`, or `strict`. The default setting is strict, which causes the task to fail if the lookup returns an error.
```sh
- name: if this file does not exist, I do not care .. file plugin itself warns anyway ...
  debug: msg="{{ lookup('file', '/nosuchfile', errors='ignore') }}"
```
- list of available plugins `ansible-doc -t lookup -l`
- to see plugin-specific documentation and examples `ansible-doc -t lookup <plugin name>`

### Templating (Jinja2)
Ansible uses Jinja2 templating to enable dynamic expressions and access to variables and facts. You can use templating with the [*template*](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html#template-module) module.

Content for test.j2
```sh
My name is {{ ansible_facts['hostname'] }}
```
```sh
- name: Write hostname
  hosts: all
  tasks:
  - name: write hostname using jinja2
    ansible.builtin.template:
       src: templates/test.j2
       dest: /tmp/hostname
```
### Tests
Tests in Jinja are a way of evaluating template expressions and returning True or False. The main difference between tests and filters are that Jinja tests are used for comparisons, whereas filters are used for data manipulation, and have different applications in jinja. Tests can also be used in list processing filters, like `map()` and `select()` to choose items in the list.
```sh
users:
  - name: alice
    enabled: true
  - name: null
    enabled: true

- name: Add only enabled users with valid names
  ansible.builtin.user:
    name: "{{ item.name }}"
    state: present
  loop: "{{ users | selectattr('enabled', 'equalto', true)| rejectattr('name', 'equalto', none) | list }}"
```

### Conditionals
Create the task, then add a `when` statement that applies for example a test. When you run the task or playbook, Ansible evaluates the test for all hosts. On any host where the test passes (returns a value of True), Ansible runs that task. You can apply conditions based on *ansible_facts* values, *registered variables* etc.
Using multiple logic with `when`:
```sh
when: ansible_facts['distribution'] == 'Ubuntu'
when: my_list | length > 0
when: <some_var> is defined
```
In Jinja2 `is defined` is a test operator used to check whether a variable exists in the current context.
 Other similar standard Jinja2 tests include:
  - is not defined
  - is none
  - is not none
  - is string
  - is iterable
  - is mapping

#### Debugging conditionals
If your conditional when statement is not behaving as you intended, you can add a debug statement to determine if the condition evaluates to `true` or `false`.
```sh
- name: check value of return code
  ansible.builtin.debug:
    var: bar_status.rc

- name: check test for rc value as string
  ansible.builtin.debug:
    var: bar_status.rc == "127"
```

### Loops
Example of using loops with conditional. The task loops through all 3 items. It skips installing `cowsay`, because of the `when` condition
```sh
- name: Install only important packages
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - vim
    - net-tools
    - cowsay
  when: item != 'cowsay'
  ```