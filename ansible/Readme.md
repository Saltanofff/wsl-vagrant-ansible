## Ansible usage notes
### Build your inventory
The simplest inventory is a single file with a list of hosts and groups. The default location for this file is /etc/ansible/hosts. You can specify a different inventory source(s) at the command line using the `-i <path or expression>` option(s) or in using the configuration system. An inventory can be generated dynamically. For example, you can use an inventory plugin to list resources in one or more cloud providers (or other sources). 
Even if you do not define any groups in your inventory file, Ansible creates two default groups: all and ungrouped after integrating all inventory sources. The all group contains every host. The ungrouped group contains all hosts that don’t have another group aside from all. Every host will always belong to at least 2 groups (all and ungrouped or all and some other group)
You can create parent/child relationships among groups. This approach is usefull because:
  - Logical grouping of hosts
  - Reuse of variables
  - Simplified targeting
  - Inheritance of group_vars. Specific group's folder vars file, will be automatically included in the parent group if you add this as the child
  - Separation of responsibility
You can increment and automatically assign multiple hostnames with 
```sh
webservers:
    hosts:
      db-[a:f].example.com
```

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
- gathering facts `ansible -i inventory.yaml server1 -m ansible.builtin.setup -a "filter=*distribution*"`. Ansible's filter option for the setup module doesn't support nested keys like `ansible_default_ipv4.address`. It only matches top-level keys 
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
- always use quote filter to make sure your variables are safe to use with shell, for example in `{{ lookup('ansible.builtin.pipe', 'getent passwd ' ~ (myuser | quote)) }}`,  `quote` wraps the input string in single quotes '...' and escapes any existing single quotes inside the string. Therefore, before we execute command - `myuser` variable is expanded safely, e.g. `myuser | quote  →  'john; rm -rf /'` 

#### Special variables 
##### Magic variables 
These variables cannot be set directly by the user; Ansible will always override them to reflect internal state. Examples:
- `hostvars` - a dictionary/map with all the hosts in inventory and variables assigned to them, including `ansible_facts` dictionary
- `groups` - a dictionary/map with all the groups in inventory and each group has the list of hosts that belong to it `{'all': ['server1', 'server2', 'server3'], ..}`
- `inventory_hostname` - contains the name of the host as configured in the inventory file

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
#or you can use another way to access the value
hostvars['webserver1']['ansible_distribution']
```
With Facts, Ansible creates two parallel variable structures:
- `ansible_default_ipv4.address`
(this is a direct top-level flattened variable injected for convenience.)
- `ansible_facts['default_ipv4']['address']`
(this is the variable stored within the ansible_facts dictionary.)
In Ansible, a flattened top-level fact refers to a fact that has been extracted from the `ansible_facts` dictionary and exposed as a direct variable — one that you can use without referencing `ansible_facts` explicitly.


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
Example - gather IPs from all hosts in one place (by default ansible gather facts on each run per host, but here all at once)
```sh
- name: Gather facts from all hosts via delegation
  hosts: localhost
  gather_facts: no
  tasks:
    - name: Collect facts from all hosts
      ansible.builtin.setup:
      register: all_facts #The "results" key is automatically added by Ansible when you use loop and register. 
      #Each element in results corresponds to one loop iteration (i.e. one delegated host in this case)
      delegate_to: "{{ item }}" #Run this task on another host (not the current one, defined by hosts:..), specifically the one in {{ item }} and we loop over all
      loop: "{{ groups['all'] }}"

    - name: Show IP addresses of all hosts
      debug:
        msg: "{{ all_facts.results | map(attribute='ansible_facts.ansible_default_ipv4.address') | list }}" #map() is lazy generator, to get values use the list
```

### Ansible playbooks
At a minimum, each play defines two things - the managed nodes to target and at least one task to execute.
By default, Ansible executes each task in order, one at a time, against all machines matched by the host pattern. Each task executes a module with specific arguments. When a task has executed on all target machines, Ansible moves on to the next task. You can use strategies to change this default behavior. Within each play, Ansible applies the same task directives to all hosts. If a task fails on a host, Ansible takes that host out of the rotation for the rest of the playbook.
You may want to verify your playbooks to catch syntax errors and other problems before you run them. The ansible-playbook command offers several options for verification, including `--check`, `--diff`, `--list-hosts`, `--list-tasks`, and `--syntax-check`. 
You can use `ansible-lint` for detailed, Ansible-specific feedback on your playbooks before you execute them.

- test playbook in check mode `ansible-playbook --check playbook.yaml`
- test syntax with linting `ansible-lint verify-apache.yml`. The ansible-lint default rules [*page*](https://ansible.readthedocs.io/projects/lint/rules/) describes each error.
- to see output results from the each run of the module, use `-v`, e.g. `ansible-playbook playbook.yaml -v`


### Ansible lookup plugins
Lookup plugins are an Ansible-specific extension to the Jinja2 templating language. You can use lookup plugins to access data from outside sources (files, databases, key/value stores, APIs, and other services) within your playbooks. You can use lookup plugins anywhere you can use templating in Ansible: in a play, in variables file, or a Jinja2 template for the template module.
You can control how errors behave in all lookup plugins by setting errors to `ignore`, `warn`, or `strict`. The default setting is strict, which causes the task to fail if the lookup returns an error.
```sh
- name: if this file does not exist, I do not care .. file plugin itself warns anyway ...
  debug: msg="{{ lookup('file', '/nosuchfile', errors='ignore') }}"
```
- example of using plugin `pipe` to execute command on the host `{{ lookup('pipe', 'hostname') }}`
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
### Tests and Filters
Tests in Jinja are a way of evaluating template expressions and returning True or False. The main difference between tests and filters are that Jinja tests are used for comparisons, whereas filters are used for data manipulation, and have different applications in jinja. Tests can also be used in list processing filters, like `map()` and `select()` to choose items in the list.

Common Jinja Tests:
  - defined
  - none
  - equalto
  - string
  - number
  - mapping (for dicts)
  - sequence (for lists)
  - in

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
If you need to debug filters, you can use `ansible-console`, e.g. run in the terminal `ansible-console -i localhost` then inside `!debug msg="{{ [1, 2, 3] | map('int') | list }}"`


### Return Values and Registering Variables
Ansible modules normally return a data structure that can be registered into a variable, or seen directly when output by the ansible program. Each module can optionally document its own unique return values (visible through ansible-doc)
You can create variables from the output of an Ansible task with the task keyword `register`. You can use registered variables in any later tasks in your play.
```sh
- hosts: web_servers
  tasks:
     - name: Run a shell command and register its output as a variable
       ansible.builtin.shell: /usr/bin/foo
       register: foo_result
       ignore_errors: true
     - name: Run a shell command using output of the previous task
       ansible.builtin.shell: /usr/bin/bar
       when: foo_result.rc == 5
```
When you use register with a loop, the [*data structure*](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_loops.html#playbooks-loops) placed in the variable will contain a results attribute that is a list of all responses from the module. This differs from the data structure returned when using register without a loop. The changed/failed/skipped attribute that’s beside the results will represent the overall state. changed/failed will be true if at least one of the iterations triggered a change/failed, while skipped will be true only if all iterations were skipped.

#### Accessing Registered Variables Across Plays
Registered variables are stored in memory. You cannot cache registered variables for use in future playbook runs. Registered variables are only valid on the host for the rest of the current playbook run, including subsequent plays within the same playbook run.
If you need to access a registered variable from a different host in another play, you can use the hostvars dictionary to reference variables from other hosts. For example, if you have a variable foo registered on localhost, you can access it from another host like this - `{{ hostvars['localhost']['foo'] }}`


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
If your conditional `when` statement is not behaving as you intended, you can add a debug statement to determine if the condition evaluates to `true` or `false`.
Below `bar_status` could be a `registered` variable used in the condition within the task as ` when: bar_status.rc == 127` , but we got unexepcted results, so we can test it with additional debugs.  To debug a conditional statement, add the *entire statement* as the `var:` value in a debug task. Ansible then shows the test and how the statement evaluates. So, based on `false` or `true` received from conditional added in debug -> var, we can get the idea of the issue:
```sh
- name: check value of return code
  ansible.builtin.debug:
    var: bar_status.rc

- name: test for rc value as integer 127 with results true of false. integer 127 in bash corresponds to "command not found" 
  ansible.builtin.debug:
    var: bar_status.rc == 127
```

### Loops
Example of using loops with conditional. If you combine a when statement with a loop, Ansible processes the condition separately for each item. The task loops through all 3 items. It skips installing `cowsay`, because of the `when` condition. 
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