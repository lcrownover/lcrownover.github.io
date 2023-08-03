---
title: Ansible - Basics
categories: [ansible]
tags: [ansible]
---

# Ansible - Basics

This is a series where I talk about how I like to organize my Ansible code.

Ansible - Setup \[Coming Soon\]

[Ansible - Basics]({% post_url 2023-08-02-ansible-basics %}) <- (you are here)

[Ansible - Roles]({% post_url 2023-08-03-ansible-roles %})

Ansible - Variables \[Coming Soon\]

Ansible - Templating \[Coming Soon\]

Ansible - Pull \[Coming Soon\]

## Basics

#### ansible.cfg

```
[defaults]
forks = 400
timeout = 5
stdout_callback = yaml

; inventory
inventory = ./inventory.yml
```

Now that ansible is configured to be able to run, let's configure a web server:

#### Install httpd and mod_ssl

shell:

```bash
useradd apache
yum install -y httpd mod_ssl
```

ansible:

```yaml
- name: create apache group
  ansible.builtin.group:
    name: apache

- name: create apache user
  ansible.builtin.user:
    name: apache
    shell: /sbin/nologin
    group: apache
    system: true

- name: install httpd and mod_ssl
  ansible.builtin.package:
    state: present
    name:
      - httpd
      - mod_ssl
```


#### Start and enable the httpd service

shell:

```bash
systemctl start httpd
systemctl enable httpd
```

#### Create the notification handler for restarting httpd

```yaml
handlers:
  - name: restart_httpd
    ansible.builtin.service:
      name: httpd
      state: restarted
      enabled: true
```

#### Ensure the ssl.conf with paths to cert and key, linking to the service

The first method is the easiest, just replace the line in file.

shell: 

```bash
sed -i 's$SSLCertificateFile /etc/pki/tls/certs/localhost.crt$SSLCertificateFile /etc/pki/tls/certs/is-racs-pack.uoregon.edu.crt$g'
sed -i 's$SSLCertificateKeyFile /etc/pki/tls/private/localhost.key$SSLCertificateKeyFile /etc/pki/tls/private/is-racs-pack.uoregon.edu.key$g'
```

ansible:

```yaml
- name: ssl certificate path
  ansible.builtin.lineinfile:
    dest: /etc/httpd/conf.d/ssl.conf
    regexp: '^SSLCertificateFile /etc/pki/tls/certs/localhost.crt$'
    line: 'SSLCertificateFile /etc/pki/tls/certs/is-racs-pack.uoregon.edu.crt'
  notify: restart_httpd

- name: ssl key path
  ansible.builtin.lineinfile:
    dest: /etc/httpd/conf.d/ssl.conf
    regexp: '^SSLCertificateKeyFile /etc/pki/tls/private/localhost.key$'
    line: 'SSLCertificateKeyFile /etc/pki/tls/private/is-racs-pack.uoregon.edu.key'
  notify: restart_httpd
```

If you can also replace the entire file. This is usually the better option, 
but it depends on how often the maintainer of the package changes the config file.
If you ensure the whole file, you'll want to audit the differences whenever there's a major update
to the package being configured.

```yaml
- name: ensure ssl.conf
  ansible.builtin.copy:
    src: ssl.conf
    dest: /etc/httpd/conf.d/ssl.conf
```

#### Ensure the cert and key, linking to service

```yaml
- name: ssl certificate file
  ansible.builtin.copy:
    src: is-racs-pack.uoregon.edu.crt
    dest: /etc/pki/tls/certs/is-racs-pack.uoregon.edu.crt
  notify: restart_httpd

- name: ssl key file
  ansible.builtin.copy:
    src: is-racs-pack.uoregon.edu.key
    dest: /etc/pki/tls/private/is-racs-pack.uoregon.edu.key
    mode: '0600'
  notify: restart_httpd
```

Use `ansible-vault` to encrypt the key file.

TODO

#### Ensure some content, linking to service

```yaml
- name: index.html
  ansible.builtin.template:
    src: index.html.j2
    dest: /var/www/html/index.html
    owner: apache
    group: apache
  notify: restart_httpd
```

```html
<html>
<h1>Welcome!</h1>
<p>This page is hosted from {{inventory_hostname}}</p>
</html>
```

## Up Next: Roles!
