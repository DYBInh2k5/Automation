# Ansible - Infrastructure Automation Platform

## Tổng quan về Ansible

Ansible là nền tảng automation mã nguồn mở cho infrastructure management, configuration management, và application deployment. Ansible sử dụng agentless architecture và YAML-based playbooks để định nghĩa automation tasks.

## Cài đặt và Cấu hình

### Installation
```bash
# Ubuntu/Debian
sudo apt update
sudo apt install ansible

# CentOS/RHEL
sudo yum install epel-release
sudo yum install ansible

# macOS
brew install ansible

# Python pip
pip install ansible

# Verify installation
ansible --version
```

### Inventory Configuration
```ini
# inventory/hosts
[webservers]
web1.example.com ansible_host=192.168.1.10
web2.example.com ansible_host=192.168.1.11
web3.example.com ansible_host=192.168.1.12

[databases]
db1.example.com ansible_host=192.168.1.20
db2.example.com ansible_host=192.168.1.21

[loadbalancers]
lb1.example.com ansible_host=192.168.1.30

[production:children]
webservers
databases
loadbalancers

[webservers:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/id_rsa
http_port=80
https_port=443

[databases:vars]
ansible_user=centos
db_port=3306
```

### Dynamic Inventory
```python
#!/usr/bin/env python3
# inventory/dynamic_inventory.py

import json
import boto3

def get_aws_inventory():
    ec2 = boto3.client('ec2', region_name='us-west-2')
    
    inventory = {
        '_meta': {
            'hostvars': {}
        }
    }
    
    response = ec2.describe_instances()
    
    for reservation in response['Reservations']:
        for instance in reservation['Instances']:
            if instance['State']['Name'] == 'running':
                instance_id = instance['InstanceId']
                private_ip = instance.get('PrivateIpAddress', '')
                public_ip = instance.get('PublicIpAddress', '')
                
                # Get tags
                tags = {tag['Key']: tag['Value'] for tag in instance.get('Tags', [])}
                
                # Group by environment
                env = tags.get('Environment', 'unknown')
                if env not in inventory:
                    inventory[env] = {'hosts': []}
                
                inventory[env]['hosts'].append(instance_id)
                
                # Host variables
                inventory['_meta']['hostvars'][instance_id] = {
                    'ansible_host': public_ip or private_ip,
                    'private_ip': private_ip,
                    'public_ip': public_ip,
                    'instance_type': instance['InstanceType'],
                    'tags': tags
                }
    
    return inventory

if __name__ == '__main__':
    print(json.dumps(get_aws_inventory(), indent=2))
```

## Playbook Development

### Basic Playbook Structure
```yaml
# playbooks/webserver.yml
---
- name: Configure Web Servers
  hosts: webservers
  become: yes
  vars:
    nginx_version: "1.20"
    app_user: "webapp"
    app_directory: "/var/www/myapp"
    
  pre_tasks:
    - name: Update package cache
      apt:
        update_cache: yes
        cache_valid_time: 3600
      when: ansible_os_family == "Debian"
      
  tasks:
    - name: Install required packages
      package:
        name:
          - nginx
          - python3-pip
          - git
          - ufw
        state: present
        
    - name: Create application user
      user:
        name: "{{ app_user }}"
        system: yes
        shell: /bin/bash
        home: "{{ app_directory }}"
        create_home: yes
        
    - name: Configure nginx
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/sites-available/default
        backup: yes
      notify:
        - restart nginx
        
    - name: Enable nginx site
      file:
        src: /etc/nginx/sites-available/default
        dest: /etc/nginx/sites-enabled/default
        state: link
      notify:
        - restart nginx
        
    - name: Configure firewall
      ufw:
        rule: allow
        port: "{{ item }}"
        proto: tcp
      loop:
        - "22"
        - "80"
        - "443"
        
    - name: Enable firewall
      ufw:
        state: enabled
        policy: deny
        direction: incoming
        
  handlers:
    - name: restart nginx
      service:
        name: nginx
        state: restarted
        enabled: yes
        
  post_tasks:
    - name: Verify nginx is running
      uri:
        url: "http://{{ ansible_default_ipv4.address }}"
        method: GET
        status_code: 200
      delegate_to: localhost
```

### Advanced Playbook with Roles
```yaml
# playbooks/site.yml
---
- name: Deploy Full Stack Application
  hosts: all
  gather_facts: yes
  
  pre_tasks:
    - name: Check connectivity
      ping:
      
- name: Configure Database Servers
  hosts: databases
  become: yes
  roles:
    - role: mysql
      vars:
        mysql_root_password: "{{ vault_mysql_root_password }}"
        mysql_databases:
          - name: myapp_production
            encoding: utf8mb4
            collation: utf8mb4_unicode_ci
        mysql_users:
          - name: myapp_user
            password: "{{ vault_mysql_app_password }}"
            priv: "myapp_production.*:ALL"
            host: "%"
            
- name: Configure Web Servers
  hosts: webservers
  become: yes
  serial: "30%"  # Rolling deployment
  max_fail_percentage: 10
  
  roles:
    - role: common
    - role: nginx
      vars:
        nginx_sites:
          - server_name: myapp.com
            root: /var/www/myapp
            index: index.php
            
    - role: php
      vars:
        php_version: "8.1"
        php_modules:
          - php8.1-mysql
          - php8.1-curl
          - php8.1-json
          
    - role: application
      vars:
        app_repository: "https://github.com/company/myapp.git"
        app_version: "{{ app_version | default('main') }}"
        
- name: Configure Load Balancers
  hosts: loadbalancers
  become: yes
  roles:
    - role: haproxy
      vars:
        haproxy_backend_servers: "{{ groups['webservers'] }}"
```

## Roles Development

### Role Structure
```
roles/
└── nginx/
    ├── defaults/
    │   └── main.yml
    ├── files/
    │   └── nginx.conf
    ├── handlers/
    │   └── main.yml
    ├── meta/
    │   └── main.yml
    ├── tasks/
    │   └── main.yml
    ├── templates/
    │   └── vhost.conf.j2
    ├── tests/
    │   ├── inventory
    │   └── test.yml
    └── vars/
        └── main.yml
```

### Role Implementation
```yaml
# roles/nginx/defaults/main.yml
---
nginx_user: www-data
nginx_worker_processes: auto
nginx_worker_connections: 1024
nginx_keepalive_timeout: 65
nginx_client_max_body_size: 64m

nginx_sites: []
nginx_remove_default_site: true

# roles/nginx/tasks/main.yml
---
- name: Install nginx
  package:
    name: nginx
    state: present
  notify: restart nginx

- name: Create nginx directories
  file:
    path: "{{ item }}"
    state: directory
    owner: root
    group: root
    mode: '0755'
  loop:
    - /etc/nginx/sites-available
    - /etc/nginx/sites-enabled
    - /var/log/nginx

- name: Configure nginx main config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    backup: yes
    validate: 'nginx -t -c %s'
  notify: restart nginx

- name: Remove default site
  file:
    path: "{{ item }}"
    state: absent
  loop:
    - /etc/nginx/sites-enabled/default
    - /etc/nginx/sites-available/default
  when: nginx_remove_default_site
  notify: restart nginx

- name: Configure virtual hosts
  template:
    src: vhost.conf.j2
    dest: "/etc/nginx/sites-available/{{ item.server_name }}"
    backup: yes
  loop: "{{ nginx_sites }}"
  notify: restart nginx

- name: Enable virtual hosts
  file:
    src: "/etc/nginx/sites-available/{{ item.server_name }}"
    dest: "/etc/nginx/sites-enabled/{{ item.server_name }}"
    state: link
  loop: "{{ nginx_sites }}"
  notify: restart nginx

- name: Start and enable nginx
  service:
    name: nginx
    state: started
    enabled: yes

# roles/nginx/handlers/main.yml
---
- name: restart nginx
  service:
    name: nginx
    state: restarted

- name: reload nginx
  service:
    name: nginx
    state: reloaded

# roles/nginx/templates/nginx.conf.j2
user {{ nginx_user }};
worker_processes {{ nginx_worker_processes }};
pid /run/nginx.pid;

events {
    worker_connections {{ nginx_worker_connections }};
    use epoll;
    multi_accept on;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout {{ nginx_keepalive_timeout }};
    types_hash_max_size 2048;
    client_max_body_size {{ nginx_client_max_body_size }};

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml+rss
        application/atom+xml
        image/svg+xml;

    # Virtual Host Configs
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}

# roles/nginx/templates/vhost.conf.j2
server {
    listen 80;
    server_name {{ item.server_name }};
    root {{ item.root }};
    index {{ item.index | default('index.html') }};

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;

    # Logging
    access_log /var/log/nginx/{{ item.server_name }}_access.log;
    error_log /var/log/nginx/{{ item.server_name }}_error.log;

    location / {
        try_files $uri $uri/ =404;
    }

    {% if item.php | default(false) %}
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
    }
    {% endif %}

    # Deny access to hidden files
    location ~ /\. {
        deny all;
    }
}
```

## Advanced Automation Patterns

### Blue-Green Deployment
```yaml
# playbooks/blue-green-deploy.yml
---
- name: Blue-Green Deployment
  hosts: webservers
  become: yes
  vars:
    app_version: "{{ app_version | mandatory }}"
    deployment_color: "{{ deployment_color | default('blue') }}"
    
  tasks:
    - name: Determine current and target environments
      set_fact:
        current_env: "{{ 'green' if deployment_color == 'blue' else 'blue' }}"
        target_env: "{{ deployment_color }}"
        
    - name: Create application directory for target environment
      file:
        path: "/var/www/{{ target_env }}"
        state: directory
        owner: www-data
        group: www-data
        mode: '0755'
        
    - name: Deploy application to target environment
      git:
        repo: "{{ app_repository }}"
        dest: "/var/www/{{ target_env }}"
        version: "{{ app_version }}"
        force: yes
      notify: restart php-fpm
      
    - name: Install application dependencies
      composer:
        command: install
        working_dir: "/var/www/{{ target_env }}"
        no_dev: yes
        optimize_autoloader: yes
        
    - name: Run database migrations
      command: php artisan migrate --force
      args:
        chdir: "/var/www/{{ target_env }}"
      environment:
        APP_ENV: production
        
    - name: Warm up application cache
      command: php artisan cache:clear && php artisan config:cache
      args:
        chdir: "/var/www/{{ target_env }}"
        
    - name: Health check target environment
      uri:
        url: "http://{{ ansible_default_ipv4.address }}:8080"
        method: GET
        status_code: 200
        timeout: 30
      retries: 5
      delay: 10
      
    - name: Switch nginx to target environment
      template:
        src: nginx-app.conf.j2
        dest: /etc/nginx/sites-available/app
        backup: yes
      vars:
        app_root: "/var/www/{{ target_env }}"
      notify: reload nginx
      
    - name: Wait for nginx reload
      meta: flush_handlers
      
    - name: Final health check
      uri:
        url: "http://{{ ansible_default_ipv4.address }}"
        method: GET
        status_code: 200
      delegate_to: localhost
      
    - name: Clean up old environment
      file:
        path: "/var/www/{{ current_env }}"
        state: absent
      when: cleanup_old_env | default(true)
```

### Rolling Updates with Health Checks
```yaml
# playbooks/rolling-update.yml
---
- name: Rolling Update with Health Checks
  hosts: webservers
  become: yes
  serial: 1
  max_fail_percentage: 0
  
  pre_tasks:
    - name: Check if server is healthy before update
      uri:
        url: "http://{{ ansible_default_ipv4.address }}/health"
        method: GET
        status_code: 200
      delegate_to: localhost
      
  tasks:
    - name: Remove server from load balancer
      uri:
        url: "http://{{ load_balancer_ip }}/api/servers/{{ inventory_hostname }}/disable"
        method: POST
        headers:
          Authorization: "Bearer {{ lb_api_token }}"
      delegate_to: localhost
      
    - name: Wait for connections to drain
      wait_for:
        timeout: 30
        
    - name: Update application
      git:
        repo: "{{ app_repository }}"
        dest: /var/www/myapp
        version: "{{ app_version }}"
        force: yes
      notify:
        - restart php-fpm
        - restart nginx
        
    - name: Install dependencies
      composer:
        command: install
        working_dir: /var/www/myapp
        no_dev: yes
        
    - name: Run migrations
      command: php artisan migrate --force
      args:
        chdir: /var/www/myapp
      run_once: true
      
    - name: Clear cache
      command: php artisan cache:clear
      args:
        chdir: /var/www/myapp
        
    - name: Flush handlers
      meta: flush_handlers
      
    - name: Health check updated server
      uri:
        url: "http://{{ ansible_default_ipv4.address }}/health"
        method: GET
        status_code: 200
        timeout: 30
      retries: 10
      delay: 5
      delegate_to: localhost
      
    - name: Add server back to load balancer
      uri:
        url: "http://{{ load_balancer_ip }}/api/servers/{{ inventory_hostname }}/enable"
        method: POST
        headers:
          Authorization: "Bearer {{ lb_api_token }}"
      delegate_to: localhost
      
  post_tasks:
    - name: Verify server is receiving traffic
      wait_for:
        timeout: 30
        
    - name: Final health check
      uri:
        url: "http://{{ ansible_default_ipv4.address }}/health"
        method: GET
        status_code: 200
      delegate_to: localhost
```

## Ansible Vault

### Encrypting Sensitive Data
```bash
# Create encrypted file
ansible-vault create group_vars/production/vault.yml

# Edit encrypted file
ansible-vault edit group_vars/production/vault.yml

# Encrypt existing file
ansible-vault encrypt group_vars/production/secrets.yml

# Decrypt file
ansible-vault decrypt group_vars/production/secrets.yml

# View encrypted file
ansible-vault view group_vars/production/vault.yml

# Change vault password
ansible-vault rekey group_vars/production/vault.yml
```

### Vault File Structure
```yaml
# group_vars/production/vault.yml
---
vault_mysql_root_password: "super_secure_password_123"
vault_mysql_app_password: "app_password_456"
vault_api_keys:
  stripe: "sk_live_abc123def456"
  sendgrid: "SG.xyz789.abc123"
vault_ssl_private_key: |
  -----BEGIN PRIVATE KEY-----
  MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQC...
  -----END PRIVATE KEY-----

# group_vars/production/vars.yml
---
mysql_root_password: "{{ vault_mysql_root_password }}"
mysql_app_password: "{{ vault_mysql_app_password }}"
api_keys: "{{ vault_api_keys }}"
ssl_private_key: "{{ vault_ssl_private_key }}"
```

### Using Vault in Playbooks
```bash
# Run playbook with vault password prompt
ansible-playbook -i inventory/production playbooks/site.yml --ask-vault-pass

# Run with vault password file
ansible-playbook -i inventory/production playbooks/site.yml --vault-password-file ~/.vault_pass

# Multiple vault passwords
ansible-playbook -i inventory/production playbooks/site.yml \
  --vault-id prod@~/.vault_pass_prod \
  --vault-id dev@~/.vault_pass_dev
```

## Custom Modules

### Python Module Development
```python
#!/usr/bin/python
# library/custom_service.py

from ansible.module_utils.basic import AnsibleModule
import requests
import json

def main():
    module_args = dict(
        name=dict(type='str', required=True),
        state=dict(type='str', required=True, choices=['present', 'absent']),
        api_url=dict(type='str', required=True),
        api_token=dict(type='str', required=True, no_log=True),
        config=dict(type='dict', required=False, default={})
    )

    result = dict(
        changed=False,
        message='',
        service={}
    )

    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True
    )

    name = module.params['name']
    state = module.params['state']
    api_url = module.params['api_url']
    api_token = module.params['api_token']
    config = module.params['config']

    headers = {
        'Authorization': f'Bearer {api_token}',
        'Content-Type': 'application/json'
    }

    try:
        # Check if service exists
        response = requests.get(f'{api_url}/services/{name}', headers=headers)
        service_exists = response.status_code == 200
        
        if service_exists:
            current_service = response.json()
        else:
            current_service = None

        if state == 'present':
            if not service_exists:
                # Create service
                if not module.check_mode:
                    data = {'name': name, 'config': config}
                    response = requests.post(f'{api_url}/services', 
                                           headers=headers, 
                                           json=data)
                    response.raise_for_status()
                    result['service'] = response.json()
                
                result['changed'] = True
                result['message'] = f'Service {name} created'
                
            else:
                # Update service if config changed
                if current_service.get('config') != config:
                    if not module.check_mode:
                        data = {'config': config}
                        response = requests.put(f'{api_url}/services/{name}',
                                              headers=headers,
                                              json=data)
                        response.raise_for_status()
                        result['service'] = response.json()
                    
                    result['changed'] = True
                    result['message'] = f'Service {name} updated'
                else:
                    result['service'] = current_service
                    result['message'] = f'Service {name} already exists with correct config'

        elif state == 'absent':
            if service_exists:
                if not module.check_mode:
                    response = requests.delete(f'{api_url}/services/{name}', headers=headers)
                    response.raise_for_status()
                
                result['changed'] = True
                result['message'] = f'Service {name} deleted'
            else:
                result['message'] = f'Service {name} does not exist'

    except requests.exceptions.RequestException as e:
        module.fail_json(msg=f'API request failed: {str(e)}', **result)
    except Exception as e:
        module.fail_json(msg=f'Unexpected error: {str(e)}', **result)

    module.exit_json(**result)

if __name__ == '__main__':
    main()
```

### Using Custom Module
```yaml
# playbooks/custom-service.yml
---
- name: Manage Custom Services
  hosts: localhost
  tasks:
    - name: Create service
      custom_service:
        name: "my-app-service"
        state: present
        api_url: "https://api.company.com"
        api_token: "{{ api_token }}"
        config:
          replicas: 3
          image: "myapp:v1.2.3"
          environment:
            - name: "DATABASE_URL"
              value: "{{ database_url }}"
      register: service_result
      
    - name: Display service info
      debug:
        var: service_result.service
```

## Testing và Validation

### Molecule Testing
```bash
# Install molecule
pip install molecule[docker]

# Initialize molecule in role
cd roles/nginx
molecule init scenario --driver-name docker

# Run tests
molecule test

# Test specific scenario
molecule test -s default
```

### Molecule Configuration
```yaml
# roles/nginx/molecule/default/molecule.yml
---
dependency:
  name: galaxy
driver:
  name: docker
platforms:
  - name: nginx-ubuntu
    image: ubuntu:20.04
    pre_build_image: true
    command: /lib/systemd/systemd
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    privileged: true
    
  - name: nginx-centos
    image: centos:8
    pre_build_image: true
    command: /usr/sbin/init
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    privileged: true

provisioner:
  name: ansible
  config_options:
    defaults:
      interpreter_python: auto_silent
      callback_whitelist: profile_tasks, timer, yaml
    ssh_connection:
      pipelining: false

verifier:
  name: ansible

# roles/nginx/molecule/default/converge.yml
---
- name: Converge
  hosts: all
  become: true
  tasks:
    - name: "Include nginx role"
      include_role:
        name: "nginx"
      vars:
        nginx_sites:
          - server_name: test.example.com
            root: /var/www/test
            index: index.html

# roles/nginx/molecule/default/verify.yml
---
- name: Verify
  hosts: all
  tasks:
    - name: Check nginx is running
      service:
        name: nginx
        state: started
      register: nginx_service
      
    - name: Verify nginx service is running
      assert:
        that:
          - nginx_service.status.ActiveState == "active"
        fail_msg: "Nginx service is not running"
        
    - name: Check nginx configuration
      command: nginx -t
      register: nginx_config_test
      
    - name: Verify nginx configuration is valid
      assert:
        that:
          - nginx_config_test.rc == 0
        fail_msg: "Nginx configuration is invalid"
        
    - name: Check if site is accessible
      uri:
        url: http://localhost
        method: GET
        status_code: 200
      register: site_check
      
    - name: Verify site is accessible
      assert:
        that:
          - site_check.status == 200
        fail_msg: "Site is not accessible"
```

Ansible cung cấp một nền tảng automation mạnh mẽ và linh hoạt cho infrastructure management với khả năng mở rộng cao thông qua roles, modules và plugins.