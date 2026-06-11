# Ansible — Complete Study Guide
### All Topics · Detailed Q&A · Master Cheatsheet
**2025 Edition — Comprehensive coverage with examples, commands & cheatsheet**

---

## Table of Contents
- [Ansible Basics](#ansible-basics)
- [Inventory: Static & Dynamic](#inventory)
- [Playbooks & Tasks](#playbooks)
- [Modules Deep Dive](#modules)
- [Variables & Facts](#variables)
- [Jinja2 Templates](#jinja2)
- [Conditionals & Loops](#conditionals)
- [Handlers](#handlers)
- [Error Handling: block/rescue/always](#error-handling)
- [Roles](#roles)
- [Tags](#tags)
- [Ansible Vault](#vault)
- [Galaxy & Collections](#galaxy)
- [Strategy Plugins](#strategy)
- [Testing with Molecule](#molecule)
- [Tower / AWX](#tower)
- [Performance Tuning](#performance)
- [Master Cheatsheet](#master-cheatsheet)

---

## Ansible Basics

### 🟢 Q1. What is Ansible and how does it work?

**Explanation:**
Ansible is an agentless automation tool. It connects to target hosts via SSH (or WinRM for Windows), copies Python code to the host, executes it, and returns results. No agent needs to be installed on managed hosts. Ansible is idempotent — running the same playbook multiple times produces the same result.

```bash
# Install
pip install ansible              # Core
pip install ansible-navigator    # Navigator UI
pip install molecule             # Testing

# Configuration (priority: env → ./ansible.cfg → ~/.ansible.cfg → /etc/ansible/ansible.cfg)
cat > ansible.cfg << 'EOF'
[defaults]
inventory       = inventory/
remote_user     = ubuntu
private_key_file = ~/.ssh/id_rsa
host_key_checking = False
retry_files_enabled = False
roles_path      = roles/
collections_path = collections/
stdout_callback = yaml          # Readable output
callback_whitelist = timer, mail
forks           = 20            # Parallel connections
timeout         = 30
interpreter_python = auto_silent

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s  # SSH multiplexing
pipelining = True               # Major performance boost
control_path = /tmp/ansible-ssh-%%h-%%p-%%r

[privilege_escalation]
become = True
become_method = sudo
become_user = root
EOF

# Ad-hoc commands
ansible all -i inventory/ -m ping
ansible webservers -m command -a "uptime"
ansible webservers -m shell -a "df -h | grep /dev/sda1"
ansible all -m setup                               # Gather facts
ansible all -m setup -a "filter=ansible_os_family"
ansible webservers -m apt -a "name=nginx state=present" -b
ansible all -m copy -a "src=./file.txt dest=/tmp/file.txt"
ansible webservers -m service -a "name=nginx state=restarted" -b
ansible all -m gather_facts --tree /tmp/facts/    # Save facts to files
```

---

## Inventory

### 🟢 Q2. How do you write Ansible inventory (static & dynamic)?

```ini
# inventory/hosts.ini — static inventory

# Ungrouped hosts
mail.example.com

[webservers]
web1.example.com ansible_host=10.0.0.1
web2.example.com ansible_host=10.0.0.2 ansible_port=2222
web3.example.com ansible_user=ec2-user ansible_ssh_private_key_file=~/.ssh/ec2.pem

[databases]
db1.example.com  ansible_host=10.0.1.1 mysql_port=3306
db2.example.com  ansible_host=10.0.1.2

[loadbalancers]
lb1.example.com

# Groups of groups
[production:children]
webservers
databases
loadbalancers

# Group vars
[webservers:vars]
nginx_port=80
nginx_worker_processes=4

[production:vars]
env=production
deploy_user=deployer
```

```yaml
# inventory/hosts.yaml — YAML format (preferred)
all:
  children:
    production:
      children:
        webservers:
          hosts:
            web1:
              ansible_host: 10.0.0.1
              nginx_port: 80
            web2:
              ansible_host: 10.0.0.2
          vars:
            nginx_worker_processes: 4
        databases:
          hosts:
            db1:
              ansible_host: 10.0.1.1
          vars:
            mysql_port: 3306
    staging:
      children:
        webservers:
          hosts:
            web-stg1:
              ansible_host: 10.1.0.1
      vars:
        env: staging
```

```yaml
# inventory/group_vars/all.yaml — vars for ALL hosts
ansible_python_interpreter: /usr/bin/python3
timezone: UTC
ntp_servers:
  - 0.pool.ntp.org
  - 1.pool.ntp.org

# inventory/group_vars/webservers.yaml
nginx_version: "1.25"
nginx_config_dir: /etc/nginx
ssl_enabled: true

# inventory/host_vars/web1.yaml — host-specific
nginx_port: 8080            # Override group var for this host
```

```python
#!/usr/bin/env python3
# inventory/dynamic_aws.py — dynamic inventory script

import json
import boto3

def get_inventory():
    ec2 = boto3.client('ec2', region_name='us-east-1')
    
    response = ec2.describe_instances(
        Filters=[{'Name': 'instance-state-name', 'Values': ['running']}]
    )
    
    inventory = {'_meta': {'hostvars': {}}}
    
    for reservation in response['Reservations']:
        for instance in reservation['Instances']:
            hostname = instance.get('PublicIpAddress') or instance.get('PrivateIpAddress')
            if not hostname:
                continue
            
            # Get tags
            tags = {t['Key']: t['Value'] for t in instance.get('Tags', [])}
            role = tags.get('Role', 'ungrouped')
            env  = tags.get('Environment', 'unknown')
            
            # Add to groups
            for group in [role, env, f"{env}_{role}"]:
                if group not in inventory:
                    inventory[group] = {'hosts': []}
                inventory[group]['hosts'].append(hostname)
            
            # Host vars
            inventory['_meta']['hostvars'][hostname] = {
                'instance_id':   instance['InstanceId'],
                'instance_type': instance['InstanceType'],
                'ansible_host':  hostname,
                'aws_tags':      tags,
            }
    
    return inventory

if __name__ == '__main__':
    print(json.dumps(get_inventory(), indent=2))
```

```bash
# Use dynamic inventory
chmod +x inventory/dynamic_aws.py
ansible all -i inventory/dynamic_aws.py --list-hosts
ansible-inventory -i inventory/dynamic_aws.py --graph

# AWS dynamic inventory (official plugin — preferred)
# inventory/aws_ec2.yaml
plugin: amazon.aws.aws_ec2
regions: [us-east-1, eu-west-1]
filters:
  instance-state-name: running
  tag:Environment: production
keyed_groups:
  - key: tags.Role
    prefix: role
  - key: placement.region
    prefix: region
compose:
  ansible_host: public_ip_address

ansible-inventory -i inventory/aws_ec2.yaml --list
ansible -i inventory/aws_ec2.yaml tag_Role_webserver -m ping
```

---

## Playbooks & Tasks

### 🟡 Q3. What is the complete structure of an Ansible Playbook?

```yaml
# site.yaml — main playbook
---
- name: Configure web servers
  hosts: webservers
  become: true
  become_user: root
  gather_facts: true          # Default true — collects system info
  serial: "30%"               # Rolling update — 30% of hosts at a time
  max_fail_percentage: 20     # Fail playbook if > 20% of hosts fail
  any_errors_fatal: false     # Continue other hosts on failure
  order: sorted               # Host execution order: default/sorted/reverse_sorted/shuffle

  # Import vars
  vars_files:
    - vars/main.yaml
    - vars/secrets.yaml

  # Inline vars (lowest priority)
  vars:
    app_name: my-web-app
    app_port: 8080

  # Prompt user for input
  vars_prompt:
    - name: admin_password
      prompt: "Enter admin password"
      private: true            # Don't echo
      confirm: true

  # Run tasks before roles
  pre_tasks:
    - name: Update apt cache
      apt:
        update_cache: true
        cache_valid_time: 3600
      tags: always

  roles:
    - common
    - nginx
    - { role: app, app_port: 8080 }
    - { role: monitoring, when: monitoring_enabled }

  # Run tasks after roles
  post_tasks:
    - name: Verify nginx is running
      service_facts:
    - name: Assert nginx state
      assert:
        that:
          - "'nginx' in services"
          - "services['nginx']['state'] == 'running'"
        fail_msg: "Nginx is not running!"

  tasks:
    - name: Install packages
      apt:
        name:
          - nginx
          - curl
          - htop
        state: present
        update_cache: true
      tags:
        - packages
        - nginx

  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

---

## Modules

### 🟡 Q4. What are the most important Ansible modules?

```yaml
# ===== FILE / COPY / TEMPLATE =====
- name: Create directory
  file:
    path: /etc/myapp
    state: directory
    owner: www-data
    group: www-data
    mode: '0750'

- name: Copy file
  copy:
    src: files/nginx.conf
    dest: /etc/nginx/nginx.conf
    owner: root
    mode: '0644'
    backup: true

- name: Copy content inline
  copy:
    content: |
      server_name {{ inventory_hostname }};
      listen {{ nginx_port }};
    dest: /etc/nginx/conf.d/server.conf

- name: Template file
  template:
    src: templates/app.conf.j2
    dest: /etc/myapp/app.conf
    owner: www-data
    mode: '0640'
  notify: Restart app

- name: Create symlink
  file:
    src: /etc/nginx/sites-available/myapp
    dest: /etc/nginx/sites-enabled/myapp
    state: link

- name: Remove file
  file:
    path: /tmp/old-file.txt
    state: absent

- name: Get file stats
  stat:
    path: /etc/nginx/nginx.conf
  register: nginx_conf

- name: Show if exists
  debug:
    msg: "Nginx config exists: {{ nginx_conf.stat.exists }}"

# ===== PACKAGE MANAGERS =====
- name: Install via apt
  apt:
    name: nginx=1.25.*
    state: present              # present / latest / absent / build-dep
    update_cache: true
    install_recommends: false

- name: Install via yum/dnf
  dnf:
    name: ['httpd', 'mod_ssl']
    state: latest
    enablerepo: epel

- name: Install Python package
  pip:
    name:
      - requests==2.31.0
      - boto3
    state: present
    virtualenv: /opt/myapp/venv

# ===== SERVICES =====
- name: Start and enable nginx
  service:
    name: nginx
    state: started
    enabled: true

- name: Use systemd directly
  systemd:
    name: myapp
    state: restarted
    daemon_reload: true         # Run daemon-reload first
    enabled: true

# ===== COMMANDS =====
- name: Run a command
  command: /opt/myapp/migrate.sh
  args:
    chdir: /opt/myapp
    creates: /opt/myapp/.migrated    # Skip if this file exists

- name: Run shell command
  shell: "ps aux | grep nginx | wc -l"
  register: process_count
  changed_when: false             # Never report as changed

- name: Run command become specific user
  command: /opt/myapp/setup.sh
  become: true
  become_user: www-data

# ===== GIT =====
- name: Clone repo
  git:
    repo: https://github.com/org/app.git
    dest: /opt/myapp
    version: "{{ app_version }}"    # Tag, branch, or commit SHA
    depth: 1                        # Shallow clone
    force: false

# ===== USER =====
- name: Create user
  user:
    name: deployer
    groups: sudo,docker
    shell: /bin/bash
    home: /home/deployer
    create_home: true
    system: false
    password: "{{ 'secretpass' | password_hash('sha512') }}"

- name: Add SSH key
  authorized_key:
    user: deployer
    key: "{{ lookup('file', 'files/deployer.pub') }}"
    state: present

# ===== CRON =====
- name: Schedule backup job
  cron:
    name: "Daily backup"
    minute: "0"
    hour: "2"
    job: "/opt/backup.sh >> /var/log/backup.log 2>&1"
    user: root
    state: present

# ===== LINEINFILE / BLOCKINFILE =====
- name: Ensure sysctl setting
  lineinfile:
    path: /etc/sysctl.conf
    regexp: '^vm.swappiness='
    line: 'vm.swappiness=10'
    state: present
  notify: Apply sysctl

- name: Add config block
  blockinfile:
    path: /etc/hosts
    block: |
      10.0.0.1 db-primary
      10.0.0.2 db-replica
    marker: "# {mark} DATABASE HOSTS"   # Idempotent marker

# ===== URI (HTTP requests) =====
- name: Call health check API
  uri:
    url: "http://{{ inventory_hostname }}:{{ app_port }}/health"
    method: GET
    status_code: 200
    timeout: 10
    return_content: true
  register: health_response
  until: health_response.status == 200
  retries: 10
  delay: 5

# ===== WAIT_FOR =====
- name: Wait for port to open
  wait_for:
    host: localhost
    port: 8080
    timeout: 60
    state: started

- name: Wait for file
  wait_for:
    path: /tmp/setup-done
    timeout: 120

# ===== AWS MODULES =====
- name: Create EC2 instance
  amazon.aws.ec2_instance:
    name: my-server
    instance_type: t3.micro
    image_id: ami-0abcdef1234567890
    region: us-east-1
    key_name: my-keypair
    security_groups: [sg-12345]
    vpc_subnet_id: subnet-12345
    tags:
      Environment: production

- name: Get RDS facts
  amazon.aws.rds_instance_info:
    db_instance_identifier: my-db
  register: rds_info
```

---

## Variables & Facts

### 🟡 Q5. How do variables work in Ansible (precedence)?

```
Variable precedence (lowest to highest):
1.  Role defaults (role/defaults/main.yaml)
2.  Inventory file group_vars/all
3.  Inventory group_vars/*
4.  Inventory host_vars/*
5.  Playbook group_vars/all
6.  Playbook group_vars/*
7.  Playbook host_vars/*
8.  Host facts (gathered)
9.  Play vars (vars: in play)
10. Play vars_prompt
11. Play vars_files
12. Role vars (role/vars/main.yaml)
13. Block vars
14. Task vars
15. include_vars
16. set_fact / registered vars
17. Role params (when including role)
18. Include params
19. Extra vars (-e flag — HIGHEST PRIORITY)
```

```yaml
# Register task output
- name: Get disk usage
  command: df -h /
  register: disk_usage
  changed_when: false

- name: Show output
  debug:
    var: disk_usage.stdout_lines

- name: Use return code
  debug:
    msg: "Command succeeded"
  when: disk_usage.rc == 0

# set_fact — create/override variable
- name: Set version fact
  set_fact:
    app_version: "{{ lookup('file', 'VERSION') | trim }}"
    deploy_time: "{{ ansible_date_time.iso8601 }}"
    cacheable: true             # Cache fact for subsequent plays

# Facts from setup module
- name: Show system facts
  debug:
    msg:
      - "OS: {{ ansible_distribution }} {{ ansible_distribution_version }}"
      - "Arch: {{ ansible_architecture }}"
      - "Memory: {{ ansible_memtotal_mb }} MB"
      - "CPUs: {{ ansible_processor_vcpus }}"
      - "IP: {{ ansible_default_ipv4.address }}"
      - "Hostname: {{ ansible_hostname }}"
      - "Python: {{ ansible_python_version }}"

# Custom facts (stored on remote host)
# Create /etc/ansible/facts.d/app.fact on remote:
- name: Deploy custom fact
  copy:
    content: |
      [app]
      version=2.0
      deployed_by=ansible
    dest: /etc/ansible/facts.d/app.fact
    mode: '0644'

- name: Refresh facts
  setup:
    filter: ansible_local

- name: Use custom fact
  debug:
    msg: "App version: {{ ansible_local.app.app.version }}"

# Magic variables
- debug:
    msg:
      - "Inventory hostname: {{ inventory_hostname }}"
      - "Short hostname: {{ inventory_hostname_short }}"
      - "Host groups: {{ group_names }}"
      - "All groups: {{ groups.keys() | list }}"
      - "Playbook dir: {{ playbook_dir }}"
      - "Role path: {{ role_path }}"
      - "Hostvars of web1: {{ hostvars['web1']['ansible_host'] }}"
```

---

## Jinja2 Templates

### 🟡 Q6. How do you write Jinja2 templates in Ansible?

```jinja2
{# templates/nginx.conf.j2 #}

{# Variables from inventory/vars #}
user {{ nginx_user | default('www-data') }};
worker_processes {{ nginx_worker_processes | default(ansible_processor_vcpus) }};

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections {{ nginx_worker_connections | default(1024) }};
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    {# Conditional block #}
    {% if nginx_log_format is defined %}
    log_format main '{{ nginx_log_format }}';
    {% else %}
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent"';
    {% endif %}

    sendfile        on;
    keepalive_timeout {{ nginx_keepalive_timeout | default(65) }};

    {# Gzip conditionally #}
    {% if nginx_gzip_enabled | default(true) %}
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
    gzip_min_length 1000;
    {% endif %}

    {# Loop over upstream servers #}
    {% if app_servers is defined %}
    upstream backend {
        {% for server in app_servers %}
        server {{ server.host }}:{{ server.port | default(8080) }} {{ server.options | default('') }};
        {% endfor %}
        keepalive 32;
    }
    {% endif %}

    {# Include all vhost configs #}
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

```jinja2
{# templates/app.conf.j2 — application config #}

[database]
host     = {{ db_host }}
port     = {{ db_port | default(5432) }}
name     = {{ db_name }}
user     = {{ db_user }}
password = {{ db_password }}

[cache]
{% if redis_enabled | default(false) %}
backend  = redis
host     = {{ redis_host }}
port     = {{ redis_port | default(6379) }}
db       = {{ redis_db | default(0) }}
{% else %}
backend  = memory
{% endif %}

[logging]
level    = {{ log_level | default('info') | upper }}
file     = {{ log_dir }}/{{ app_name }}.log

[features]
{% for feature, enabled in features.items() %}
{{ feature }} = {{ enabled | lower }}
{% endfor %}

{# Complex filters #}
server_names = {{ server_names | join(', ') }}
allowed_ips  = {{ allowed_ips | map('regex_replace', '^(.*)$', '\\1/32') | join(' ') }}
deploy_time  = {{ ansible_date_time.iso8601 }}
server_hash  = {{ inventory_hostname | hash('md5') | truncate(8, True, '') }}
```

```yaml
# Using template task
- name: Generate nginx config
  template:
    src: templates/nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
    validate: nginx -t -c %s    # Validate before saving
  notify: Reload nginx

# Jinja2 filters reference
"{{ 'hello' | upper }}"                     # HELLO
"{{ items | join(', ') }}"                  # a, b, c
"{{ value | default('fallback') }}"
"{{ value | default(omit) }}"               # Remove key if undefined
"{{ '2024-01-01' | to_datetime }}"
"{{ dict | dict2items }}"                   # Convert to list of {key,value}
"{{ list | items2dict }}"
"{{ list | selectattr('state', 'eq', 'active') | list }}"
"{{ list | map(attribute='name') | list }}"
"{{ number | int | abs }}"
"{{ string | b64encode }}"
"{{ string | b64decode }}"
"{{ string | to_json }}"
"{{ string | from_json }}"
"{{ string | regex_replace('old', 'new') }}"
"{{ string | regex_findall('\\d+') }}"
"{{ password | password_hash('sha512') }}"
"{{ 'secret' | vault(vault_id='default') }}"
```

---

## Conditionals & Loops

### 🟡 Q7. How do conditionals and loops work in Ansible?

```yaml
# ===== WHEN (conditionals) =====
- name: Install on Debian
  apt:
    name: nginx
    state: present
  when: ansible_os_family == 'Debian'

- name: Install on RedHat
  dnf:
    name: nginx
    state: present
  when: ansible_os_family == 'RedHat'

# Multiple conditions
- name: Only on production Ubuntu 22.04
  apt:
    name: mypackage
  when:
    - env == 'production'
    - ansible_distribution == 'Ubuntu'
    - ansible_distribution_version is version('22.04', '>=')

# OR condition
- name: When A or B
  debug:
    msg: "condition met"
  when: condition_a or condition_b

# Check if variable is defined
- name: Only if var is defined
  debug:
    msg: "{{ optional_var }}"
  when: optional_var is defined

- name: Check file exists
  command: /usr/bin/mycommand
  when: ansible_local.app is defined

# Check result of previous task
- name: Run if previous task changed
  command: systemctl restart app
  when: config_file.changed

# ===== LOOPS =====

# Simple loop (with_items deprecated → use loop)
- name: Install packages
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - curl
    - htop
    - jq

# Loop with index
- name: Create numbered files
  file:
    path: "/tmp/file-{{ item.0 }}-{{ item.1 }}"
    state: touch
  loop: "{{ range(3) | list | zip(['a','b','c']) | list }}"

# Loop over list of dicts
- name: Create users
  user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
    shell: "{{ item.shell | default('/bin/bash') }}"
  loop:
    - { name: alice, groups: "sudo,docker" }
    - { name: bob,   groups: "docker" }
    - { name: carol, groups: "sudo" }
  loop_control:
    label: "{{ item.name }}"        # Only show name in output (not full dict)

# Loop with dict2items
- name: Set sysctls
  sysctl:
    name: "{{ item.key }}"
    value: "{{ item.value }}"
    state: present
  loop: "{{ sysctl_settings | dict2items }}"
  vars:
    sysctl_settings:
      vm.swappiness: 10
      net.core.somaxconn: 65535
      net.ipv4.ip_local_port_range: "1024 65535"

# Loop with until (retry)
- name: Wait for service to respond
  uri:
    url: http://localhost:8080/health
    status_code: 200
  register: result
  until: result.status == 200
  retries: 20
  delay: 5

# Nested loops
- name: Create user .ssh dirs
  file:
    path: "/home/{{ item[0] }}/.ssh"
    state: directory
    owner: "{{ item[0] }}"
    mode: '0700'
  loop: "{{ users | product(['']) | list }}"

# with_subelements (loop over nested list)
- name: Add SSH keys for all users
  authorized_key:
    user: "{{ item.0.name }}"
    key: "{{ item.1 }}"
  loop: "{{ users | subelements('ssh_keys', skip_missing=True) }}"
```

---

## Handlers

### 🟡 Q8. How do handlers work in Ansible?

```yaml
# Handlers are tasks triggered by notify — run once at end of play
# handlers/main.yaml (in role) or inline in playbook

handlers:
  - name: Restart nginx
    service:
      name: nginx
      state: restarted

  - name: Reload nginx
    service:
      name: nginx
      state: reloaded

  - name: Restart application
    systemd:
      name: myapp
      state: restarted
      daemon_reload: true

  # Handler chaining — handler notifying another handler
  - name: Validate nginx config
    command: nginx -t
    notify: Reload nginx        # If validation succeeds, reload

  # Handler with listen — multiple tasks can notify one handler
  - name: Handle config change
    command: /opt/myapp/reload.sh
    listen: "app config changed"

tasks:
  - name: Update nginx config
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: Restart nginx

  - name: Update nginx sites
    template:
      src: site.conf.j2
      dest: /etc/nginx/sites-available/mysite
    notify: Reload nginx           # Only reload for site config

  - name: Update app config
    template:
      src: app.conf.j2
      dest: /etc/myapp/app.conf
    notify: "app config changed"   # Uses listen

# Force handler execution immediately (don't wait for end of play)
- name: Run pending handlers now
  meta: flush_handlers

# Conditional handler
handlers:
  - name: Restart nginx
    service:
      name: nginx
      state: restarted
    when: nginx_running.stat.exists
```

---

## Error Handling

### 🟡 Q9. How do block/rescue/always work for error handling?

```yaml
# ===== IGNORE ERRORS =====
- name: Try to stop service (may not exist)
  service:
    name: old-service
    state: stopped
  ignore_errors: true

# ===== CHANGED_WHEN / FAILED_WHEN =====
- name: Check free disk space
  command: df -BG / --output=avail
  register: disk_space
  changed_when: false              # Command never modifies state
  failed_when: disk_space.stdout | int < 5  # Custom failure condition

- name: Run database migration
  command: ./migrate.sh
  register: migration
  failed_when:
    - migration.rc != 0
    - "'No changes' not in migration.stdout"  # OK if no migrations needed

# ===== BLOCK / RESCUE / ALWAYS =====
- name: Deploy application
  block:
    - name: Pull Docker image
      docker_image:
        name: "myapp:{{ version }}"
        source: pull

    - name: Stop old container
      docker_container:
        name: myapp
        state: stopped

    - name: Start new container
      docker_container:
        name: myapp
        image: "myapp:{{ version }}"
        state: started
        ports:
          - "8080:8080"

  rescue:
    - name: Rollback — restart old container
      docker_container:
        name: myapp
        image: "myapp:{{ previous_version }}"
        state: started

    - name: Send alert
      uri:
        url: "{{ slack_webhook }}"
        method: POST
        body_format: json
        body:
          text: "❌ Deployment failed! Rolled back to {{ previous_version }}"

    - name: Fail after rollback
      fail:
        msg: "Deployment failed, rolled back to {{ previous_version }}"

  always:
    - name: Cleanup temp files
      file:
        path: /tmp/deploy-temp
        state: absent

    - name: Log deployment attempt
      lineinfile:
        path: /var/log/deployments.log
        line: "{{ ansible_date_time.iso8601 }} - {{ version }} - {{ ansible_failed_result | default('success') }}"

# ===== ANY_ERRORS_FATAL =====
- name: Critical configuration
  any_errors_fatal: true           # Stop ALL hosts if any fails
  tasks:
    - name: Configure database
      ...
```

---

## Roles

### 🟡 Q10. What is an Ansible Role and how do you structure one?

```
roles/nginx/
├── defaults/
│   └── main.yaml          # Default variables (lowest priority)
├── vars/
│   └── main.yaml          # Role variables (high priority)
├── tasks/
│   ├── main.yaml          # Entry point
│   ├── install.yaml       # Sub-tasks included from main
│   ├── configure.yaml
│   └── ssl.yaml
├── handlers/
│   └── main.yaml          # Handlers for this role
├── templates/
│   ├── nginx.conf.j2
│   └── vhost.conf.j2
├── files/
│   └── mime.types         # Static files
├── meta/
│   └── main.yaml          # Dependencies, metadata
└── README.md
```

```yaml
# roles/nginx/tasks/main.yaml
---
- name: Include OS-specific variables
  include_vars: "{{ ansible_os_family }}.yaml"
  tags: always

- name: Install nginx
  include_tasks: install.yaml
  tags: [install, packages]

- name: Configure nginx
  include_tasks: configure.yaml
  tags: [configure]

- name: Configure SSL
  include_tasks: ssl.yaml
  when: nginx_ssl_enabled | default(false)
  tags: [ssl]

# roles/nginx/defaults/main.yaml
nginx_version: latest
nginx_user: www-data
nginx_worker_processes: auto
nginx_worker_connections: 1024
nginx_keepalive_timeout: 65
nginx_ssl_enabled: false
nginx_gzip_enabled: true
nginx_log_dir: /var/log/nginx
nginx_vhosts: []

# roles/nginx/meta/main.yaml
galaxy_info:
  author: Platform Team
  description: Install and configure Nginx
  license: MIT
  min_ansible_version: "2.14"
  platforms:
    - name: Ubuntu
      versions: [jammy, focal]
    - name: EL
      versions: [8, 9]
  galaxy_tags: [nginx, web, proxy]

dependencies:
  - role: common                   # Run common role first
  - role: ssl_certs
    when: nginx_ssl_enabled
```

---

## Tags

### 🟡 Q11. How do tags work in Ansible?

```yaml
tasks:
  - name: Install nginx
    apt:
      name: nginx
    tags:
      - install
      - packages
      - nginx

  - name: Configure nginx
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    tags: configure

  - name: This always runs
    debug:
      msg: "Always"
    tags: always            # Special tag — always runs unless explicitly skipped

  - name: This never runs
    debug:
      msg: "Never"
    tags: never             # Special tag — only runs if explicitly called
```

```bash
# Run only tagged tasks
ansible-playbook site.yaml --tags "install,packages"
ansible-playbook site.yaml --tags configure

# Skip specific tags
ansible-playbook site.yaml --skip-tags "ssl,monitoring"

# List all tags in playbook
ansible-playbook site.yaml --list-tags

# List tasks that would run
ansible-playbook site.yaml --tags nginx --list-tasks

# Run tasks tagged 'always' only
ansible-playbook site.yaml --tags always
```

---

## Ansible Vault

### 🟡 Q12. How do you use Ansible Vault?

```bash
# Create encrypted file
ansible-vault create vars/secrets.yaml
ansible-vault create --vault-id production@prompt vars/prod-secrets.yaml

# Encrypt existing file
ansible-vault encrypt vars/secrets.yaml

# Edit encrypted file
ansible-vault edit vars/secrets.yaml

# View without decrypting in place
ansible-vault view vars/secrets.yaml

# Decrypt file
ansible-vault decrypt vars/secrets.yaml

# Rekey (change password)
ansible-vault rekey vars/secrets.yaml

# Encrypt just a string (embed in YAML)
ansible-vault encrypt_string 'mypassword' --name 'db_password'
# Output:
# db_password: !vault |
#   $ANSIBLE_VAULT;1.1;AES256
#   ...

# Use in playbook
ansible-playbook site.yaml --ask-vault-pass
ansible-playbook site.yaml --vault-password-file ~/.vault_pass
ansible-playbook site.yaml --vault-id production@~/.vault_pass_prod

# vault-id for multiple vaults
ansible-playbook site.yaml \
  --vault-id dev@~/.vault_dev \
  --vault-id prod@~/.vault_prod

# In ansible.cfg:
[defaults]
vault_password_file = ~/.ansible_vault_pass
```

---

## Galaxy & Collections

### 🟡 Q13. How do you use Ansible Galaxy and Collections?

```yaml
# requirements.yaml
---
roles:
  - name: geerlingguy.nginx
    version: 3.2.0
  - name: geerlingguy.docker
    version: 7.1.0
  - src: https://github.com/myorg/ansible-role-myapp.git
    name: myapp
    version: main

collections:
  - name: amazon.aws
    version: ">=7.0.0"
  - name: community.general
    version: ">=8.0.0"
  - name: kubernetes.core
    version: ">=3.0.0"
  - name: community.docker
    version: ">=3.0.0"
```

```bash
# Install all requirements
ansible-galaxy install -r requirements.yaml
ansible-galaxy collection install -r requirements.yaml

# Or both at once
ansible-galaxy install -r requirements.yaml
ansible-galaxy collection install -r requirements.yaml --force

# Install to specific path
ansible-galaxy install -r requirements.yaml -p ./roles/

# List installed
ansible-galaxy role list
ansible-galaxy collection list

# Create a new role scaffold
ansible-galaxy role init my_role --init-path roles/

# Create a new collection scaffold
ansible-galaxy collection init myorg.mycollection

# Search
ansible-galaxy search nginx --author geerlingguy
```

---

## Strategy Plugins

### 🟡 Q14. What are Ansible strategy plugins?

```yaml
# Strategy controls how tasks run across hosts

# linear (default) — each task completes on ALL hosts before next task starts
- hosts: webservers
  strategy: linear
  tasks:
    - name: Task 1   # Runs on all hosts
    - name: Task 2   # Runs after Task 1 on ALL hosts

# free — each host runs through tasks as fast as possible (independent)
- hosts: webservers
  strategy: free     # Hosts don't wait for each other
  tasks:
    - name: Task 1
    - name: Task 2

# debug — interactive debugging
- hosts: webservers
  strategy: debug    # Pauses on errors for interactive inspection

# serial — rolling update (process N hosts at a time)
- hosts: webservers
  serial: 2          # 2 hosts at a time
  # or:
  serial:
    - 1              # First: 1 host (canary)
    - 30%            # Then: 30% of remaining
    - 100%           # Then: all remaining

# mitogen (third-party — massive speedup via Python execution)
# Install: pip install mitogen
# ansible.cfg:
# [defaults]
# strategy_plugins = /usr/lib/python3/dist-packages/ansible_mitogen/plugins/strategy
# strategy = mitogen_linear
```

---

## Testing with Molecule

### 🔴 Q15. How do you test Ansible roles with Molecule?

```bash
# Install
pip install molecule molecule-docker

# Initialize molecule in existing role
cd roles/nginx
molecule init scenario --driver-name docker

# Run full test suite
molecule test

# Individual steps
molecule create          # Create test instances
molecule converge        # Run playbook
molecule verify          # Run tests
molecule idempotence     # Run playbook twice, check no changes on 2nd run
molecule lint            # Lint playbook + role
molecule destroy         # Destroy instances
molecule login           # SSH into test instance
```

```yaml
# molecule/default/molecule.yml
---
dependency:
  name: galaxy
  options:
    requirements-file: requirements.yaml

driver:
  name: docker

platforms:
  - name: instance-ubuntu
    image: geerlingguy/docker-ubuntu2204-ansible:latest
    pre_build_image: true
    privileged: true
    cgroupns_mode: host
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    command: /lib/systemd/systemd

  - name: instance-rocky
    image: geerlingguy/docker-rockylinux9-ansible:latest
    pre_build_image: true
    privileged: true
    command: /lib/systemd/systemd

provisioner:
  name: ansible
  playbooks:
    converge: converge.yml
    verify: verify.yml
  inventory:
    group_vars:
      all:
        nginx_ssl_enabled: false
        nginx_vhosts:
          - server_name: test.example.com
            document_root: /var/www/html

verifier:
  name: ansible

lint: |
  set -e
  yamllint .
  ansible-lint
```

```yaml
# molecule/default/converge.yml
---
- name: Converge
  hosts: all
  become: true
  tasks:
    - name: Include nginx role
      include_role:
        name: nginx

# molecule/default/verify.yml
---
- name: Verify
  hosts: all
  gather_facts: false
  tasks:
    - name: Check nginx is installed
      package_facts:
        manager: auto

    - name: Assert nginx installed
      assert:
        that:
          - "'nginx' in ansible_facts.packages"

    - name: Check nginx service
      service_facts:

    - name: Assert nginx running
      assert:
        that:
          - "'nginx' in services"
          - "services['nginx']['state'] == 'running'"

    - name: Check port 80 is listening
      wait_for:
        port: 80
        timeout: 5

    - name: Check nginx responds
      uri:
        url: http://localhost:80/
        status_code: [200, 301, 302]
```

---

## Tower / AWX

### 🔴 Q16. What is Ansible Tower/AWX?

```
Tower = commercial product (Red Hat Ansible Automation Platform)
AWX  = open-source upstream (what Tower is built on)

Key features:
- Web UI for playbook execution
- Role-based access control (RBAC)
- Job scheduling
- Inventory management (static + dynamic)
- Credential management (encrypted vault)
- Workflow editor (chain jobs with conditions)
- Notifications (Slack, email, PagerDuty)
- REST API for integration
- Audit trails
- Multi-tenancy (Organizations)
```

```bash
# AWX installation (Kubernetes)
# Install AWX Operator
kubectl apply -f https://raw.githubusercontent.com/ansible/awx-operator/main/deploy/awx-operator.yaml

# Create AWX instance
cat > awx-instance.yaml << 'EOF'
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
  namespace: awx
spec:
  service_type: NodePort
  ingress_type: none
  image_pull_policy: IfNotPresent
  postgres_storage_class: fast-ssd
  postgres_storage_requirements:
    requests:
      storage: 10Gi
  projects_persistence: true
  projects_storage_class: fast-ssd
  projects_storage_size: 8Gi
EOF
kubectl apply -f awx-instance.yaml -n awx

# Get admin password
kubectl get secret awx-admin-password -o jsonpath="{.data.password}" | base64 --decode

# AWX API usage
# Create inventory
curl -X POST http://awx:8080/api/v2/inventories/ \
  -H "Authorization: Bearer $AWX_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"Production","organization":1}'

# Launch a job
curl -X POST http://awx:8080/api/v2/job_templates/5/launch/ \
  -H "Authorization: Bearer $AWX_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"extra_vars":{"env":"production","version":"1.2.3"}}'
```

---

## Performance Tuning

### 🔴 Q17. How do you tune Ansible for performance?

```ini
# ansible.cfg — performance optimizations

[defaults]
forks = 50                          # Parallel hosts (default 5)
host_key_checking = False
gathering = smart                   # Cache facts between plays
fact_caching = redis                # Or: jsonfile, memory
fact_caching_connection = localhost:6379
fact_caching_timeout = 3600         # 1 hour
pipelining = True                   # Reduce SSH operations (MAJOR speedup)
interpreter_python = auto_silent

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no
pipelining = True
control_path_dir = /tmp/.ansible-cp
control_path = %(directory)s/%%h-%%r
```

```yaml
# Playbook-level optimizations

# 1. gather_facts: false when facts not needed
- hosts: webservers
  gather_facts: false
  tasks:
    - name: Simple command (no facts needed)
      command: echo hello

# 2. Gather only needed facts
- hosts: webservers
  gather_facts: false
  tasks:
    - name: Get only network facts
      setup:
        gather_subset:
          - network
          - '!hardware'             # Exclude hardware (slow)

# 3. Async tasks — run long tasks in background
- name: Run long database migration
  command: ./migrate.sh
  async: 3600                       # Max runtime (seconds)
  poll: 0                           # Don't wait — fire and forget

- name: Check migration status
  async_status:
    jid: "{{ migration.ansible_job_id }}"
  register: job_result
  until: job_result.finished
  retries: 60
  delay: 30

# 4. Delegate_to — run task on different host
- name: Add to load balancer
  command: lb-add {{ inventory_hostname }}
  delegate_to: loadbalancer.example.com

# 5. Run_once — only run on one host
- name: Create database schema
  command: ./create-schema.sh
  run_once: true
  delegate_to: db-primary

# 6. Using mitogen strategy (3-10x faster)
# Install: pip install mitogen
# In ansible.cfg:
[defaults]
strategy = mitogen_linear
```

---

## Master Cheatsheet

### Core Commands
```bash
ansible all -m ping                           # Test connectivity
ansible all -m setup                          # Gather facts
ansible webservers -m apt -a "name=nginx state=present" -b
ansible-playbook site.yaml                    # Run playbook
ansible-playbook site.yaml -i production/     # Specific inventory
ansible-playbook site.yaml --limit webservers # Limit to group
ansible-playbook site.yaml --tags deploy      # Run tagged tasks
ansible-playbook site.yaml --check            # Dry run
ansible-playbook site.yaml --diff             # Show diffs
ansible-playbook site.yaml -e "version=1.2"   # Extra vars
ansible-playbook site.yaml -v/-vv/-vvv        # Verbosity
ansible-vault encrypt vars/secrets.yaml       # Encrypt file
ansible-vault edit vars/secrets.yaml          # Edit encrypted
ansible-galaxy install -r requirements.yaml   # Install roles
ansible-galaxy collection install amazon.aws  # Install collection
molecule test                                 # Test role
```

### Common Patterns
```yaml
# Register + use result
- command: id deploy_user
  register: result
  ignore_errors: true
- debug: var=result.stdout
  when: result.rc == 0

# Idempotent file creation
- file: path=/dir state=directory mode=0755

# Template with validate
- template: src=a.j2 dest=/etc/a.conf validate="cmd -t %s"
  notify: Restart service

# Loop with label
- user: name={{ item.name }} groups={{ item.groups }}
  loop: "{{ users }}"
  loop_control: { label: "{{ item.name }}" }

# Until loop
- uri: url=http://localhost/health status_code=200
  register: r
  until: r.status == 200
  retries: 10
  delay: 5
```

### Question Coverage Index
| Q# | Topic | Difficulty |
|---|---|---|
| Q1 | Ansible basics & ad-hoc | 🟢 |
| Q2 | Inventory: static & dynamic | 🟢 |
| Q3 | Playbook structure | 🟡 |
| Q4 | Essential modules | 🟡 |
| Q5 | Variables, facts & precedence | 🟡 |
| Q6 | Jinja2 templates | 🟡 |
| Q7 | Conditionals & loops | 🟡 |
| Q8 | Handlers | 🟡 |
| Q9 | block/rescue/always error handling | 🟡 |
| Q10 | Roles structure | 🟡 |
| Q11 | Tags | 🟡 |
| Q12 | Ansible Vault | 🟡 |
| Q13 | Galaxy & Collections | 🟡 |
| Q14 | Strategy plugins | 🟡 |
| Q15 | Molecule testing | 🔴 |
| Q16 | Tower / AWX | 🔴 |
| Q17 | Performance tuning | 🔴 |
