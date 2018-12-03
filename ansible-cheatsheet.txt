###############################################################################
## Lucian Maly <lmaly@odecee.com.au>                                 2018-12-04
###############################################################################

## Ansible works best if the passwordless access is configured on all nodes:
$ ssh-keygen
$ ssh-copy-id lmaly@<HOST>
$ ssh lmaly@<HOST>

## Configuration, Inventory default(s):
$ cat /etc/ansible/hosts
$ grep host_file ansible.cfg
  Priority in which the config files are processed:
   1) ANSIBLE_CONFIG (an environment variable)
   2) ./ansible.cfg (in the current directory)
   3) ~/.ansible.cfg
   4) /etc/ansible/ansible.cfg
  $ ansible --version     <-shows what config file is currently being used

Commonly Modified Settings in 'ansible.cfg':
inventory= Location of Ansible inventory file
remote user= Remote user account used to establish connections to managed hosts
become= Enables/disables privilege escalation for operations on managed hosts (NOT BY DEFAULT!)
become_method= Defines privilege escalation method
become_user= User account for escalating privileges
become_ask_pass= Defines whether privilege escalation prompts for password

# Default module (command, NOT SHELL!) is defined in 'ansible.cfg' under 'defaults' section:
$ grep module_name /etc/ansible/ansible.cfg
  !If no modules defined, predefined command module used

###############################################################################
## Run the playbook(s):
  $ ansible -i localhost, <playbook>.yaml
    !When using the '-i' parameter, if the value is a 'list' (it contains at least one comma) it will be used as the inventory LIST
    !It can be either FQDN or IP address
  $ ansible -i /etc/ansible/hosts <playbook>.yaml
    !While the variable is a string, it will be used as the inventory PATH

## Ansible verbosity:
$ ansible-playbook -v -i <HOST> <playbook>.yaml      <-displays output data
$ ansible-playbook -vv -i <HOST> <playbook>.yaml     <-displays output & input data
$ ansible-playbook -vvv -i <HOST> <playbook>.yaml    <-information about connections
$ ansible-playbook -vvvv -i <HOST> <playbook>.yaml   <-information about plugins

###############################################################################
## Inventory file examples:
    [webserver]          <-both hosts can be reduced to ws[01:02].lmaly.io using regexp
    ws01.lmaly.io   
    ws02.lmaly.io:1234   <-configured to listen on port 1234 

    [database]
    db01.lmaly.io
    192.168.[1:5].[0:255]  <-includes 192.168.1.0/24 through 192.168.5.0/24

    [nsw_cities:children] <-nested groups use the keyword :children
    webserver
    database
    others

    [others]
    localhost ansible_connection=local                        <-it will use local connection, without SSH
    192.168.3.7 ansible_connection=ssh ansible_user=ftaylor   <-use different settings example

  # Dynamic inventory - AWS custom script
  !Script must support '--list' an '--host' parameters, e.g.:
  $ ./inventiryscript --list
    $ sudo cp ec2.py /etc/ansible/
    $ sudo cp ec2.ini /etc/ansible/
    $ sudo chmod +x /etc/ansible/ec2.py
    $ ansible -i ec2.py all -m ping

!Multiple inventory files in one dir = All the files in the directory are merged and treated as one inventory

###############################################################################
## Which hosts will the playbook run against?
$ ansible-playbook <playbook>.yaml --list-hosts

## Which hosts are in the hosts file?
$ ansible all --list-hosts
$ ansible '192.168.2.*' -i <myinventory> --list-hosts                     <-everything that matches the wildcard
$ ansible lab:datacenter1 -i <myinventory> --list-hosts                   <-members of lab OR datacenter1
$ ansible 'lab:&datacenter1' -i <myinventory> --list-hosts                <-members of lab AND datacenter1
$ ansible 'datacenter:!test2.example.com' -i <myinventory> --list-hosts   <-exclude host

## Print all the tasks in short form that playbook would perform:
$ ansible-playbook <playbook>.yaml --list-tasks

###############################################################################
## See the metadata obtained during Ansible setup (module setup):
$ ansible all -i <HOST>, -m setup
$ ansible <group> -m setup
$ ansible <group> -m setup --tree facts
# Show the specific metadata:
$ ansible <group> -m setup -a 'filter=*ipv4*'
  # Examples:
  {{ ansible_hostname }}
  {{ ansible_default_ipv4.address }}
  {{ ansible_devices.vda.partitions.vda1.size }}
  {{ ansible_dns.nameservers }}
  {{ ansible_kernel }}
# To disable facts:
  gather_facts: no
# Custom facts are saved under ansible_local:
  $ cat /etc/ansible/facts.d/new.fact    <-ini or json file
  $ ansible demo1.example.com -m setup -a 'filter=ansible_local'
# Accessing facts of another node:
  {{ hostvars['demo2.example.com']['ansible_os_family'] }}

###############################################################################
# Run arbitrary command on all hosts as sudo:
$ ansible all -s -a "cat /var/log/messages"

# Run Ad Hoc module with arguments:
$ ansible host pattern -m module -a 'argument1 argument2' [-i inventory]

# Run ad hoc command that generates one line input for each operation:
$ ansible myhosts -m command -a /usr/bin/hostname -o

# Ad hoc Example: yum module checks if httpd installed on demohost:
$ ansible demohost -u devops -b -m yum -a 'name=httpd state=present'

# Ad hoc Example: Find available space on disks configured on demohost:
$ ansible demohost -a "df -h"

# Ad hoc Example: Find available free memory on demohost:
$ ansible demohost -a "free -m"

###############################################################################
## Playbook(s) basics:
---                                       <-indicates YAML, optional document marker
- hosts: all                              <-host or host group against we will run the task (example - hosts: webserver)
  remote_user: lmaly                      <-as what user will Ansible log in to the machine(s)
  tasks:                                  <-list of actions
  - name: Whatever you want to name it    <-name of the first task
    yum:                                  
      name: httpd                         
      state: present                      <-same as single line 'yum: name=httpd state=present become=True'
      become: True                        <-task will be executed with sudo
...                                       <-optional document marker indicating end of YAML

Notes:
  !Space characters used for indentation
  !Pipe (|) preservers line returns within string
  !Arrow (>) converts line returns to spaces, removes leading space in lines
  !Indentation rules:
   Elements at same level in hierarchy must have same indentation
   Child elements must be indented further than parents
   No rules about exact number of spaces to use
  !Optional: Insert blank lines for readability
  !Use YAML Lint http://yamllint.com to check the syntax correctness
  !Use 'ansible-playbook --syntax-check <MYAML.YML>'
  !Use 'ansible-playbook -C <MYAML.YML>' for dry run (what changes would occur if playbook is executed?)
  !Use 'always_run: true' to execute only some tasks in check mode (or 'always_run: false' for the opposite)
  !Use 'ansible-playbook --step <MYAML.YML>' to prompt each task (y=yes/n=no/c=exit and execute remaining without asking)
  !Use 'ansible-playbook <MYAML.YML> --start-at-task="start httpd service" to start execution from given task
  !For complex playbooks, use 'include' to include separate files in main playbook (include: tasks/env.yml)

###############################################################################
## Variables:
--- # Usually comment field describing what it does
- hosts: all
  remote_user: lmaly
  tasks:
  - name: Set variable 'name'
    set_fact:
      name: Test machine
  - name: Print variable 'name'
    debug:
      msg: '{{ name }}'

a/ In vars block:
 vars:
   set_fact: 'Test machine'

b/ Passed as arguments:
  include_vars: vars/extra_args.yml

  # Other alternative ways:
    (1) pass variable(s) in the CLI:
    $ ansible-playbook -i <HOST>, <playbook>.yaml  -e 'name=test01'

    (2) pass variable(s) to an inventory file:
      a/
      [webserver]   <- all playbooks running on webservers will be able to refer to the domain name variable
      ws01.lmaly.io domainname=example1.lmaly.io
      ws02.lmaly.io domainname=example2.lmaly.io
    
      b/
      [webserver:vars]    <- ! Host variables will override group variables in case tge same variable is used in both
      https_enabled=True

    (3) in variable file(s):
      a/ host_vars:
      $ cat host_vars/ws01.lmaly.io
        domainname=example1.lmaly.io
    
      b/ group_vars:
      $ cat group_vars/webserver
        https_enabled=True
  
  !Variable must start with letter; valid characters are: letters, numbers, underscores
  # Array definition example:
    users:
     bjones:
      first_name: Bob
      last_name: Jones
     acook:
      first_name: Anne
      last_name: Cook
  # Array accessing/reading example:  
    users.bjones.first_name             # Returns 'Bob'
    users['bjones']['first_name']       # Returns 'Bob'  <-preferable method

  # Inventory variables hierarchy/scope
    !Three levels: Global, Play, Host
    !If Ansible finds variables with the same name, it uses chain of preference (highest priority on the top):
 ^   A) Common variable file     - overrides everything, host/group/inventory variable files
 |      01) 'extra' variables via CLI (-e)
 |      02) Task variables (only for task itself)
 |      03) Defined in 'block' statement (- block:)
 |      04) 'role' and 'include' variables
 |      05) Included using 'vars_files' (vars_files: - /vars/env.yml)
 |      06) 'vars_prompt' variables (vars_prompt:)
 |      07) Defined with '-a' or '--args' command line
 |      08) Defined via 'set_facts' (- set_fact: user: joe)
 |      09) Registered variables with 'register' keyword for debugging
 |      10) Host facts discovered by Ansible (ansible_facts)
 |   B) Host variables           - overrides group variable files
 |      11) 'host_vars' in host_vars directory
 |   C) Group variables          - overrides inventory variable files
 |      12) 'group_vars' in group_vars directory
 |   D) Inventory variable files - lowest priority
 |      13) 'host_vars' in the inventory file ([hostgroup:vars])
 |      14) 'group_vars' in the inventory file ([hostgroup:children])
 |      15) Inventory file variables - global
 |      16) Role default variables - Set in roles vars directory

    !Variables passed to the CLI have priority over any other variable.
    !You can override the following parameters from an inventory file:
      ansible_user, ansible_port, ansible_host, ansible_connection, ansible_private_key_file, ansible_shell_type

###############################################################################
## Iterates (http://docs.ansible.com/ansible/playbooks_loops.html)
   (1) Standard - 'with_items' example:
       a/ Simple loop
       firewall
           firewalld:
             service: '{{ item }}'
             state: enabled
             permanent: True
             immediate: True
             become: True
          with_items:
          - http
          - https

       b/ List of hashes
       - user:
          name: {{ item.name }}
          state: present
          groups: {{ item.groups }}
         with_items:
          - { name: 'jane', groups: 'wheel' }
          - { name: 'joe', groups: 'root' }

   (2) Nested loops (iterate all elements of a list with all items from other lists) - 'with_nested' example:
       user:
         name: '{{ item }}'
       become: True
       with_items:
       - '{{ users }}'
       file:
         path: '/home/{{ item.0 }}/{{ item.1 }}'
         state: directory
       become: True
       with_nested:
       - '{{ users }}'
       - '{{ folders }}'
      
   (3) Fileglobs loop (action on every file present in a certain folder) - 'with_fileglobs' example:
   copy:
     src: '{{ item }}'
     dest: '/tmp/iproute2'
     remote_src: True
   become: True
   with_fileglob:
   - '/etc/iproute2/rt_*'

   (4) Integer loop (iterate over the integer numbers) - 'with_sequence' example:
   file:
     dest: '/tmp/dir{{ item }}'
     state: directory
   with_sequence: start=1 end=10
   become: True

   Notes:
   with_file - Takes list of file names; 'item' set to content of each file in sequence
   with_fileglob - Takes file name globbing pattern; 'item' set to each file in directory that matches pattern; in sequence, nonrecursively
   with_sequence - Generates sequence of items in increasing numerical order; Can take 'start' and 'end' arguments
                 - Supports decimal, octal, hexadecimal integer values
   with_random_choices - Takes list; 'item' set to one list item at random

   Note - Conditionals:
   Equal                                     "{{ max_memory }} == 512"
   Less than                                 "{{ min_memory }} < 128"
   Greater than                              "{{ min_memory }} > 256"
   Less than or equal                        "{{ min_memory }} <= 256"
   Greater than or equal                     "{{ min_memory }} >= 512"
   Not equal                                 "{{ min_memory }} != 512"
   Variable exists                           "{{ min_memory }} is defined"
   Variable does not exist                   "{{ min_memory }} is not defined"
   Variable set to 1, True, or yes           "{{ available_memory }}"
   Variable set to 0, False, or no           "not {{ available_memory }}"
   Value present in variable or array        "{{ users }} in users["db_admins"]"
   AND                                       ansible_kernel == 3.10.el7.x86_64 and inventory_hostname in groups['staging']
   OR                                        ansible_distribution == "RedHat" or ansible_distribution == "Fedora"

    (5) 'when' statement example
    - name: Create the database admin
       user:
        name: db_admin
      when: inventory_hostname in groups["databases"]   <-when is not a module variable, it must be placed outside of the module
     
    (6) When combining 'when' with 'with_items', the when statement is processed for each item:
     with_items: "{ ansible_mounts }"
     when: item.mount == "/" and item.size_available > 300000000 

###############################################################################
## Register - To capture output of module that uses array
   - shell: echo "{{ item }}"
   with_items:
     - one
     - two
   register: echo

###############################################################################
## Handlers - Task that responds to notification triggered by other tasks:
    notify:
     - restart_apache
    handlers:
     -name: restart_apache
       service:
        name: httpd:
        state: restarted
    !Task may call more than one handler in notify
    !Ansible treats notify as an array and iterates over handler names
    !Handlers run in order they are written in play, not in order listed by notify task
    !If two handlers have same name, only one runs
    !Handler runs only once, even if notified by more than one task
    !Handlers defined inside 'include' cannot be notified
    !--force-handlers

###############################################################################
## Resource tagging
    a/ Tasks
       tags:
        - packages
    b/ Roles - all tasks they define also tagged
       roles:
        - { role: user, tags: [production] }
    c/ Include - all tasks they define also tagged
       - include: common.yml
       tags: [testing]
    $ ansible-playbook --tags "production"
    $ ansible-playbook --skip-tags "development"
    Special tags: always, tagged, untagged, all (default)

###############################################################################
## Ignore and task errors
    Default: If task fails, play is aborted immediatelly, same with handlers
      To skip failed tasks, use 'ignore_errors: yes'
    To force handler to be called even if task fails, use 'force_handlers: yes'
    To mark task as failed even when it succeeds, use 'failed_when:
       cmd: /usr/local/bin/create_user.show
      register: command_result
      failed_when: "'Password missing' in command_result.stdout"
    Default: Task acquires 'changed' state after updating managed host, which causes handlers skipped if task does not make changed
      changed_when: "'Success' in command_result.stdout"

###############################################################################
## Environmental variables

   a/ Set env:
   environment:
    http_proxy: http://demo.lab.example.com:8080

   b/ Reference vars inside playbook:
   - block:
     environment:
      name: "{{ myname }}"

###############################################################################
## Blocks - logical grouping of tasks, controls how are they executed
    a/ block:        <-main tasks to run
    b/ rescue:       <-tasks to run if block tasks fail
    c/ always:       <-tasks to always run, independent on a/ or b/
    tasks:
     - block:
       - shell:
          cmd: /usr/local/lib/upgrade-database
         rescue:
       - shell:
          cmd: /usr/local/lib/create-user
         always:
       - service:
          name: mariadb
          state: restarted

###############################################################################
## Delegation using 'delegate_to' and 'local_action'
   
   a/ To localhost using delegate_to:
   - name: Running Local Process
     command: ps
     delegate_to: localhost
     register: local_process
   
   b/ To localhost using local_action:
   - name: Running Local Process
     local_action: command 'ps'
     register: local_process

   c/ Host outside play in inventory - example of removing managed hosts from ELB, upgrading and the adding them back:
   - hosts: webservers
     tasks:
      - name: Remove server from load balancer
        command: remove-from-lb {{ inventory_hostname }}
        delegate_to: localhost
      - name: Deploy the latest version of Webstack
        git:repo=git://git.example.com/path/repo.git dest=/srv/www
      - name: Add server back to ELB pool
        command: add-to-lb {{ inventory_hostname }}
        delegate_to: localhost
  Inventory data used when creating connection to target:
   ansible_connection
   ansible_host
   ansible_port
   ansible_user

   d/ Host outside play NOT in inventory using 'add_host':
   - name: Add delegation host
     add_host: name=demo ansible_host=172.25.250.10 ansible_user=devops
   - name: Silly echo
     command: echo {{ inventory_hostname }}
     delegate_to: demo
      args:                                             <-complex arguments can be added
       chdir: somedir/
       creates: /path/to/database

   e/ 'run_once: True'

!Facts gathered by delegated tasks assigned to 'delegate_to' host, not host that produced facts
!To gather facts from delegated host, set 'delegate_facts: True' in the play

###############################################################################
## Jinja2 Template(s) - no need for file extension:
---
- hosts: all
  remote_user: lmaly
  tasks:
  - name: Ensure the website is present and updated
    template:
      src: index.html.j2                           <-the J2 template source
      dest: /var/www/html/index.html               <-destination
      owner: root
      group: root
      mode: 0644
    become: True

  # Variables in J2 template(s):
    '{{ VARIABLE_NAME }}'
    '{{ ARRAY_NAME['KEY'] }}'
    '{{ OBJECT_NAME.PROPERTY_NAME }}'

    # Example of index.html.j2 using variable:
      # {{ ansible_managed }}                <-recommended to include it in each template
      <html>
       <body>
        <h1>Hello World!</h1>
        <p>This page was created on {{ ansible_date_time.date }}.</p>
       </body>
      </html>

  # Comments:
    {# ... #}

  # Built-in Filtres in J2 template(s):
    '{{ VARIABLE_NAME | capitalize }}'
    '{{ output | to_json }}
    '{{ output | to_yaml }}
    '{{ output | to_nice_json }}
    '{{ output | to_nice_yaml }}
    '{{ output | from_json }}
    '{{ output | from_yaml }}
    '{{ forest_blockers|split('-') }}'

  # Conditionals:
    { % ... % }
    a/ Equal to example A
    {% if ansible_eth0.active == True %}
       <p>eth0 address {{ ansible_eth0.ipv4.address }}.</p>
    {% endif %}

    b/ Equal to example B
    {% if ansible_eth0.active is equalto True %}
       <p>eth0 address {{ ansible_eth0.ipv4.address }}.</p>
    {% endif %}
    
  # Cycles/loops:
    {% for address in ansible_all_ipv4_addresses %}
       <li>{{ address }}</li>
    {% endfor %}

  # Issues
    a/ When value after : starts with { you need to " the whole object
       app_path: "{{ base_path }}/bin"
    b/ When you have nested {{...}} elements, remove the inner set:
       msg: Host {{ params[{{ host_ip }}] }}            <-wrong
       msg: Host {{ params[ host_ip] }}                 <-fine

More information:
http://jinja.pocoo.org/docs/dev/templates/#builtin-filters
http://jinja.pocoo.org/docs/dev/templates/#builtin-tests

###############################################################################
## Roles
# Structure:
  It looks for different roles at 'roles' subdirectory or 'roles_path' dir in ansible.cfg (default /etc/ansible/roles)
  Top-level = specifically named role name
  Subdirs   = main.yml
  files     = contains objects referenced by main.yml
  templates = contains objects referenced by main.yml
!Role tasks execute before tasks of play in which they appear
!To override default, use 'pre_tasks' (performed before any roles applied) and 'post_tasks' (performed after all roles completed)

# Example - folder structure:
$ tree user.example
user.example/
├── defaults
│   └── main.yml                 <-define default variables, easily overriden
├── files                        <-fixed-content files, empty subdir is ignored
├── handlers
│   └── main.yml
├── meta
│   └── main.yml                 <-define dependency roles ('allow_duplicates: yes')
├── README.md
├── tasks
│   └── main.yml                 <-role content (defines modules to call on managed hosts where the role is applied
├── templates                    <-contain templates
├── tests
│   ├── inventory
│   └── test.yml
└── vars                         
│   └── main.yml                 <-vars for this module (best practice)
└── main.yml

# Use roles in play example:
---
- hosts: remote.example.com
  roles:
   - role1
   - role2
   - davidkarban.git            <-role from Ansible Galaxy using _AUTHOR.NAME_ 

# Override default variables example:
---
- hosts: remote.example.com
  roles:
   - { role: role1 }
   - { role: role2, var1: val1, var2: val2 }

# Define dependency roles in meta/main.yml example:
---
dependencies:
 - { role: apache, port: 8080 }
 - { role: postgress, dbname: serverlist, admin_user: felix }
!Role added as dependency to play once, to override default, set 'allow_duplicates=yes' in meta/main.yml

# Content example of MOTD role (tasks/main.yml)
---
# tasks file for MOTD
- name: deliver motd file
  template:
   src: templates/motd.j2
   dest: /etc/motd
   owner: root
   group: root
   mode: 0444

# Role sources
  $ cat roles2install.yml
    # From Galaxy:
    - src: author.rolename

    # From different source:
    - src: https://webserver.example.com/files.tgz
      name: ftpserver-role
  $ ansible-galaxy init -r roles2install.yml


###############################################################################
## Modules
 # Custom modules
   Priority in which the custom module is being processed:
   1, ANSIBLE_LIBRARY environment variable
   2, 'library' in the ansible.cfg
   3, ./library/ relative to location of playbook in use
 # Default modules
   $ cd /usr/lib/python2.7/site-packages/ansible/modules
   $ ansible-doc -l                       <-list all modules
   $ ansible-doc <MODULE>                 <-help for the module
   $ ansible-doc -s <MODULE>              <-simple view for the module

###############################################################################
## Optimizations
(1) Default 'smart' settings (transport=smart in ansible.cfg):
    a/ check if locally installed SSH supports 'ControlPersist', if not, 'paramiko' is used
    b/ ControlPersist=60s (listed as comment in /etc/ansible/ansible.cfg)
(2) Other settings include:
    a/ paramiko (Python implementation of SSHv2, does not have ControlPersist)
    b/ local    (runs locally and not over SSH)
    c/ ssh      (uses OpenSSH)
    d/ docker   (uses docker exec)
    e/ plug-ins (not based on SSH, e.g. chroot, libvirt_lxc...)
(3) Change on-the-fly settings with 
    a/ $ ansible-playbook <PLAYBOOK> -c <TRANSPORT_NAME>
    b/ $ ansible <PLAYBOOK> -c <TRANSPORT_NAME>
(4) SSH settings are under [ssh_connection]
(5) Paramiko settings are under [paramiko_connection]
(6) To specify connection type in the inventory file, use 'ansible_connection':
    [targets]
    localgost ansible_connection=local
    demo.lab.example.com ansible_connection=ssh
(7) To specify connection type in play:
    ---
    - name: Connection type
      hosts: 127.0.0.01
      connection: local
(8) To limit concurrent connection, use SSH server's 'MaxStartups' option
(9) Parallelism - by default 5 different machines at once:
    a/ setting 'forks' in 'ansible.cfg'
    b/ $ ansible-playbook --forks
    c/ 'serial' in the play overrides 'ansible.cfg' - either number or %
    d/ 'async' & 'async_status' - value is time that Ansible waits for command to complete [default 3600s, long tasks 0]
    e/ 'pool' - sets how often Ansible checks if command has completed [default 10s, long tasks 0]
    f/ 'wait_for'
    g/ pause module

###############################################################################
## Ansible vault
(1a) Create encrypted file:
     $ ansible-vault create <FILENAME>
     $ ansible-vault create --vault-password-file=.secret_file <FILENAME>
(2a) Enter and confirm new vault password

(1b) Or encrypt and existing file(s):
     $ ansible-vault encrypt <FILE1> <FILE2> ...
     $ ansible-vault encrypt --output=NEW_FILE <OLDFILE>
(2b) Enter and confirm new vault password

(3) View file:
    $ ansible-vault view <FILENAME>
(4) Edit file:
    $ ansible-vault edit <FILENAME>
(5) Change password:
    $ ansible-vault rekey <FILENAME1> <FILENAME2>
    $ ansible-vault rekey --new-vault-password-file=.secret_file <FILENAME>
(6) Decrypt file:
    $ ansible-vault decrypt <FILENAME> --output=<FILENAME>
Variable types:
    a/ Defined in 'group_vars' or 'host_vars'
    b/ Loaded by 'include_vars' or 'vars_files'
    c/ Passed on 'ansible-playbook -e @file.yml'
    d/ Defined as role variables & defaults
    
    $ ansible-playbook --ask-vault-pass <PLAY.YML>
    $ ansible-playbook --vault-password-file=.secret_file <PLAY.YML>
      or 'EXPORT ANSIBLE_VAULT_PASSWORD_FILE=~/.secret_file'

###############################################################################
## Troubleshooting
   By default no log, but you can enable it:
   a/ 'log_path' parameter under [default] in 'ansible.cfg'
   b/ ANSIBLE_LOG_PATH environment variable

# Debug mode examples:
  - debug: msg="The free memory for this system is {{ ansible_memfree_mb }}"
  - debug: var=output verbosity=2

# Report changes made to templated files:
  $ ansible-playbook --check --diff <MYAML.YML>

# 'URI' module to check if RESTfuk API is returning required content:
  tasks:
  - action: uri url=http://api.myapp.com return_content=yes
  register: apiresponse
  - fail: msg='version was not provided'
  when: "'version' not in apiresponse.content"

# 'script' module to execute script on managed host (module fails if $? is other then 0):
  taks:
  - script: check_free_memory

# 'stat' module to see if files/dirs not managed by Ansible are present
  'assert' module to see if file exists in managed host
  tasks:
  - stat: path=/var/run/app.lock
  register: lock
  - assert:
  that:
  - lock.stat.exists

###############################################################################
## Ansible Tower
   Web-based interface
   Enterprise solution for IT automation
   Dashboard for managing deployments and monitoring resources
   Adds automation, visual management, monitoring capabilities to Ansible
   Gives administrators control over user access
    Uses SSH credentials
    Blocks access to or transfer of credentials
   Implements continuous delivery and configuration management
   Integrates management in single tool

# Installation - Configuration file:
  tower_setup_conf.yml under setup bundle directory

  Installation - 'configure' options:
  -l                      <-install on local machine with internal PostgreSQL   
  --no-secondary-prompt   <-skip prompts regarding secondary Tower nodes to be added  
  -A                      <-Disable aut-generation of PostgreSQL password, prompt user for passwords
  -o <FILE>               <-source for configuration answers

  Installation - after the config file is ready:
  ./setup.sh
  -c <FILE>               <-specify file that stores the configuration
  -i <FILE>               <-
  -p                      <-specify file to use for host inventory
  -s                      <-require Ansible to prompt for SSH passwords
  -u                      <-require Ansible to prompt for sudo passwords
  -e                      <-set additional variables during installation
  -b                      <-perform database backup instead of installing Tower
  -r <BACKUP_FILE>        <-perform database restore instead of installing Tower

  Changing your password:
  $ sudo tower-manage changepassword admin

# SSL certificate:
  /etc/tower/awx.cert
  /etc/tower/awx.key

# REST API from CLI example:
  $ curl -s http://demo.lab.example.com/api/v1/ping | json_reformat

# User types:
  normal
  organization admin
  superuser

# User permissions:
  read
  write
  admin
  execute commands
  check
  run
  create

# Projects:
  /var/lib/awx/projects
  $ sudo mkdir /var/lib/awx/projects/demoproject
  $ sudo cp demo.yml /var/lib/awx/projects/demoproject
  $ sudo chown -R awx /var/lib/awx/projects/demoproject

###############################################################################
## CLI:
(1) Run a command somewhere else using Ansible
$ ansible
Usage: ansible <host-pattern> [options]
Options:
  -a MODULE_ARGS, --args=MODULE_ARGS
                        module arguments
  --ask-vault-pass      ask for vault password
  -B SECONDS, --background=SECONDS
                        run asynchronously, failing after X seconds
                        (default=N/A)
  -C, --check           don't make any changes; instead, try to predict some
                        of the changes that may occur
  -D, --diff            when changing (small) files and templates, show the
                        differences in those files; works great with --check
  -e EXTRA_VARS, --extra-vars=EXTRA_VARS
                        set additional variables as key=value or YAML/JSON
  -f FORKS, --forks=FORKS
                        specify number of parallel processes to use
                        (default=5)
  -h, --help            show this help message and exit
  -i INVENTORY, --inventory-file=INVENTORY
                        specify inventory host path
                        (default=/etc/ansible/hosts) or comma separated host
                        list.
  -l SUBSET, --limit=SUBSET
                        further limit selected hosts to an additional pattern
  --list-hosts          outputs a list of matching hosts; does not execute
                        anything else
  -m MODULE_NAME, --module-name=MODULE_NAME
                        module name to execute (default=command)
  -M MODULE_PATH, --module-path=MODULE_PATH
                        specify path(s) to module library (default=None)
  --new-vault-password-file=NEW_VAULT_PASSWORD_FILE
                        new vault password file for rekey
  -o, --one-line        condense output
  --output=OUTPUT_FILE  output file name for encrypt or decrypt; use - for
                        stdout
  -P POLL_INTERVAL, --poll=POLL_INTERVAL
                        set the poll interval if using -B (default=15)
  --syntax-check        perform a syntax check on the playbook, but do not
                        execute it
  -t TREE, --tree=TREE  log output to this directory
  --vault-password-file=VAULT_PASSWORD_FILE
                        vault password file
  -v, --verbose         verbose mode (-vvv for more, -vvvv to enable
                        connection debugging)
  --version             show program's version number and exit

  Connection Options:
    control as whom and how to connect to hosts

    -k, --ask-pass      ask for connection password
    --private-key=PRIVATE_KEY_FILE, --key-file=PRIVATE_KEY_FILE
                        use this file to authenticate the connection
    -u REMOTE_USER, --user=REMOTE_USER
                        connect as this user (default=None)
    -c CONNECTION, --connection=CONNECTION
                        connection type to use (default=smart)
    -T TIMEOUT, --timeout=TIMEOUT
                        override the connection timeout in seconds
                        (default=10)
    --ssh-common-args=SSH_COMMON_ARGS
                        specify common arguments to pass to sftp/scp/ssh (e.g.
                        ProxyCommand)
    --sftp-extra-args=SFTP_EXTRA_ARGS
                        specify extra arguments to pass to sftp only (e.g. -f,
                        -l)
    --scp-extra-args=SCP_EXTRA_ARGS
                        specify extra arguments to pass to scp only (e.g. -l)
    --ssh-extra-args=SSH_EXTRA_ARGS
                        specify extra arguments to pass to ssh only (e.g. -R)

  Privilege Escalation Options:
    control how and which user you become as on target hosts

    -s, --sudo          run operations with sudo (nopasswd) (deprecated, use
                        become)
    -U SUDO_USER, --sudo-user=SUDO_USER
                        desired sudo user (default=root) (deprecated, use
                        become)
    -S, --su            run operations with su (deprecated, use become)
    -R SU_USER, --su-user=SU_USER
                        run operations with su as this user (default=root)
                        (deprecated, use become)
    -b, --become        run operations with become (does not imply password
                        prompting)
    --become-method=BECOME_METHOD
                        privilege escalation method to use (default=sudo),
                        valid choices: [ sudo | su | pbrun | pfexec | runas |
                        doas | dzdo ]
    --become-user=BECOME_USER
                        run operations as this user (default=root)
    --ask-sudo-pass     ask for sudo password (deprecated, use become)
    --ask-su-pass       ask for su password (deprecated, use become)
    -K, --ask-become-pass
                        ask for privilege escalation password


###############################################################################
(2) Run Ansible playbook
$ ansible-playbook
Usage: ansible-playbook playbook.yml
Options:
  --ask-vault-pass      ask for vault password
  -C, --check           don't make any changes; instead, try to predict some
                        of the changes that may occur
  -D, --diff            when changing (small) files and templates, show the
                        differences in those files; works great with --check
  -e EXTRA_VARS, --extra-vars=EXTRA_VARS
                        set additional variables as key=value or YAML/JSON
  --flush-cache         clear the fact cache
  --force-handlers      run handlers even if a task fails
  -f FORKS, --forks=FORKS
                        specify number of parallel processes to use
                        (default=5)
  -h, --help            show this help message and exit
  -i INVENTORY, --inventory-file=INVENTORY
                        specify inventory host path
                        (default=/etc/ansible/hosts) or comma separated host
                        list.
  -l SUBSET, --limit=SUBSET
                        further limit selected hosts to an additional pattern
  --list-hosts          outputs a list of matching hosts; does not execute
                        anything else
  --list-tags           list all available tags
  --list-tasks          list all tasks that would be executed
  -M MODULE_PATH, --module-path=MODULE_PATH
                        specify path(s) to module library (default=None)
  --new-vault-password-file=NEW_VAULT_PASSWORD_FILE
                        new vault password file for rekey
  --output=OUTPUT_FILE  output file name for encrypt or decrypt; use - for
                        stdout
  --skip-tags=SKIP_TAGS
                        only run plays and tasks whose tags do not match these
                        values
  --start-at-task=START_AT_TASK
                        start the playbook at the task matching this name
  --step                one-step-at-a-time: confirm each task before running
  --syntax-check        perform a syntax check on the playbook, but do not
                        execute it
  -t TAGS, --tags=TAGS  only run plays and tasks tagged with these values
  --vault-password-file=VAULT_PASSWORD_FILE
                        vault password file
  -v, --verbose         verbose mode (-vvv for more, -vvvv to enable
                        connection debugging)
  --version             show program's version number and exit

  Connection Options:
    control as whom and how to connect to hosts

    -k, --ask-pass      ask for connection password
    --private-key=PRIVATE_KEY_FILE, --key-file=PRIVATE_KEY_FILE
                        use this file to authenticate the connection
    -u REMOTE_USER, --user=REMOTE_USER
                        connect as this user (default=None)
    -c CONNECTION, --connection=CONNECTION
                        connection type to use (default=smart)
    -T TIMEOUT, --timeout=TIMEOUT
                        override the connection timeout in seconds
                        (default=10)
    --ssh-common-args=SSH_COMMON_ARGS
                        specify common arguments to pass to sftp/scp/ssh (e.g.
                        ProxyCommand)
    --sftp-extra-args=SFTP_EXTRA_ARGS
                        specify extra arguments to pass to sftp only (e.g. -f,
                        -l)
    --scp-extra-args=SCP_EXTRA_ARGS
                        specify extra arguments to pass to scp only (e.g. -l)
    --ssh-extra-args=SSH_EXTRA_ARGS
                        specify extra arguments to pass to ssh only (e.g. -R)

  Privilege Escalation Options:
    control how and which user you become as on target hosts

    -s, --sudo          run operations with sudo (nopasswd) (deprecated, use
                        become)
    -U SUDO_USER, --sudo-user=SUDO_USER
                        desired sudo user (default=root) (deprecated, use
                        become)
    -S, --su            run operations with su (deprecated, use become)
    -R SU_USER, --su-user=SU_USER
                        run operations with su as this user (default=root)
                        (deprecated, use become)
    -b, --become        run operations with become (does not imply password
                        prompting)
    --become-method=BECOME_METHOD
                        privilege escalation method to use (default=sudo),
                        valid choices: [ sudo | su | pbrun | pfexec | runas |
                        doas | dzdo ]
    --become-user=BECOME_USER
                        run operations as this user (default=root)
    --ask-sudo-pass     ask for sudo password (deprecated, use become)
    --ask-su-pass       ask for su password (deprecated, use become)
    -K, --ask-become-pass
                        ask for privilege escalation password


###############################################################################
(3) Set up a remote copy of ansible on each managed node (clone Ansible configuration files from Git repository)
$ ansible-pull 
Usage: ansible-pull -U <repository> [options]

Options:
  --accept-host-key     adds the hostkey for the repo url if not already added
  --ask-vault-pass      ask for vault password
  -C CHECKOUT, --checkout=CHECKOUT
                        branch/tag/commit to checkout.  Defaults to behavior
                        of repository module.
  -d DEST, --directory=DEST
                        directory to checkout repository to
  -e EXTRA_VARS, --extra-vars=EXTRA_VARS
                        set additional variables as key=value or YAML/JSON
  -f, --force           run the playbook even if the repository could not be
                        updated
  --full                Do a full clone, instead of a shallow one.
  -h, --help            show this help message and exit
  -i INVENTORY, --inventory-file=INVENTORY
                        specify inventory host path
                        (default=/etc/ansible/hosts) or comma separated host
                        list.
  -l SUBSET, --limit=SUBSET
                        further limit selected hosts to an additional pattern
  --list-hosts          outputs a list of matching hosts; does not execute
                        anything else
  -m MODULE_NAME, --module-name=MODULE_NAME
                        Repository module name, which ansible will use to
                        check out the repo. Default is git.
  -M MODULE_PATH, --module-path=MODULE_PATH
                        specify path(s) to module library (default=None)
  --new-vault-password-file=NEW_VAULT_PASSWORD_FILE
                        new vault password file for rekey
  -o, --only-if-changed
                        only run the playbook if the repository has been
                        updated
  --output=OUTPUT_FILE  output file name for encrypt or decrypt; use - for
                        stdout
  --purge               purge checkout after playbook run
  --skip-tags=SKIP_TAGS
                        only run plays and tasks whose tags do not match these
                        values
  -s SLEEP, --sleep=SLEEP
                        sleep for random interval (between 0 and n number of
                        seconds) before starting. This is a useful way to
                        disperse git requests
  -t TAGS, --tags=TAGS  only run plays and tasks tagged with these values
  -U URL, --url=URL     URL of the playbook repository
  --vault-password-file=VAULT_PASSWORD_FILE
                        vault password file
  -v, --verbose         verbose mode (-vvv for more, -vvvv to enable
                        connection debugging)
  --verify-commit       verify GPG signature of checked out commit, if it
                        fails abort running the playbook. This needs the
                        corresponding VCS module to support such an operation
  --version             show program's version number and exit

  Connection Options:
    control as whom and how to connect to hosts

    -k, --ask-pass      ask for connection password
    --private-key=PRIVATE_KEY_FILE, --key-file=PRIVATE_KEY_FILE
                        use this file to authenticate the connection
    -u REMOTE_USER, --user=REMOTE_USER
                        connect as this user (default=None)
    -c CONNECTION, --connection=CONNECTION
                        connection type to use (default=smart)
    -T TIMEOUT, --timeout=TIMEOUT
                        override the connection timeout in seconds
                        (default=10)
    --ssh-common-args=SSH_COMMON_ARGS
                        specify common arguments to pass to sftp/scp/ssh (e.g.
                        ProxyCommand)
    --sftp-extra-args=SFTP_EXTRA_ARGS
                        specify extra arguments to pass to sftp only (e.g. -f,
                        -l)
    --scp-extra-args=SCP_EXTRA_ARGS
                        specify extra arguments to pass to scp only (e.g. -l)
    --ssh-extra-args=SSH_EXTRA_ARGS
                        specify extra arguments to pass to ssh only (e.g. -R)

  Privilege Escalation Options:
    control how and which user you become as on target hosts

    --ask-sudo-pass     ask for sudo password (deprecated, use become)
    --ask-su-pass       ask for su password (deprecated, use become)
    -K, --ask-become-pass
                        ask for privilege escalation password

###############################################################################
(4) Accessing documentation locally
$ ansible-doc
Usage: ansible-doc [options] [module...]

Options:
  -h, --help            show this help message and exit
  -l, --list            List available modules
  -M MODULE_PATH, --module-path=MODULE_PATH
                        specify path(s) to module library (default=None)
  -s, --snippet         Show playbook snippet for specified module(s)
  -v, --verbose         verbose mode (-vvv for more, -vvvv to enable
                        connection debugging)
  --version             show program's version number and exit

###############################################################################
(5) Ansible Galaxy tool
$ ansible-galaxy

Usage: ansible-galaxy [delete|import|info|init|install|list|login|remove|search|setup] [--help] [options] ...

Options:
  -h, --help     show this help message and exit
  -v, --verbose  verbose mode (-vvv for more, -vvvv to enable connection
                 debugging)
  --version      show program's version number and exit
Examples:
 ansible-galaxy search --author <AUTHOR>
 ansible-galaxy search --platforms <PLATFORM>
 ansible-galaxy search --galaxy-tags <TAG>
 ansible-galaxy info <ROLE>
 ansible-galaxy install <ROLE> -p <ROLE_DIRECTORY>
 ansible-galaxy install -r <ROLE1> <ROLE2> <ROLE3> ...
 ansible-galaxy list
 ansible-galaxy remove <ROLE>
 ansible-galaxy init <ROLE>
 ansible-galaxy init --offline <ROLE>

###############################################################################
(6) Hiding secrets
$ ansible-vault

Usage: ansible-vault [create|decrypt|edit|encrypt|rekey|view] [--help] [options] vaultfile.yml

Options:
  --ask-vault-pass      ask for vault password
  -h, --help            show this help message and exit
  --new-vault-password-file=NEW_VAULT_PASSWORD_FILE
                        new vault password file for rekey
  --output=OUTPUT_FILE  output file name for encrypt or decrypt; use - for
                        stdout
  --vault-password-file=VAULT_PASSWORD_FILE
                        vault password file
  -v, --verbose         verbose mode (-vvv for more, -vvvv to enable
                        connection debugging)
  --version             show program's version number and exit

###############################################################################
(7) Windows
$ pip install pywinrm       <-- on the control machine

Authentication:
a/ Certificate: authentication similar to SSH
b/ Kerberos: python-kerberos
c/ CredSSP: for local and domain accounts

On the client(s):
a/ PowerShell 3.0 or higher
b/ Enable PowerShell
c/ Set up WinRM:
   https://github.com/ansible/ansible/blob/devel/examples/scripts/ConfigureRemotingForAnsible.ps1

For Kerberos, on the client(s):
python-devel, krb5-devel, krb5-libs, krb5-workstation, pywinrm
/etc/krb5.conf.d/ansible.conf
[realms]
ad1.${GUID}.example.com = {
  kdc = ad1.${GUID}.example.opentlc.com
}

'ansible.cfg' example:
ansible_connection=winrm
ansible_user=Administrator

Windows Ansible modules examples:
win_ping
win_chocolatey
win_service
win_firewall
win_firewall_rule
win_user
win_domain_user
win_domain_controller
