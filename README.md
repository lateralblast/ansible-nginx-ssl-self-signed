![alt tag](https://raw.githubusercontent.com/lateralblast/ansible-grub-serial/master/images/self-signed.jpg)

Automating Nginx SSL Self Signed Certificate Configuration
==========================================================

Introduction
------------

This is a quick example of how to automate Nginx SSL configuration with self signed certificates

Requirements
------------

Required software:

- Ansible
- Nginx
- OpenSSL

Overview
--------

This example does the following:

- Check packages are installed
- Create key pair
- Create certificate signing request
- Create certificates
- Create Delfie Hellman parameters
- Create Nginx self-signed SSL configuration file
- Create Nginx self-signed SSL parameters file
- Create Nginx SSL self-signed site configuration file

Check packages are installed:

```
- name: Check Base Packages
  apt:
    name:         "{{ item.package }}"
    state:        "{{ item.state }}"
    update_cache: yes
  loop:
    - { package: "nginx",   state: "present" }
    - { package: "openssl", state: "present" }
```

Create key pair:

```
- name: Create Key Pair
  openssl_privatekey:
    path: /etc/ssl/private/nginx-selfsigned.key
    size: 204
```

Create certificate signing request:

```
- name: Create Certificate Signing Request
  openssl_csr:
    common_name:        "Common Name"
    country_name:       "XX"
    email_address:      "email@domainname.com"
    locality_name:      "City"
    organization_name:  "Org Name"
    path:               /etc/ssl/certs/nginx-selfsigned.crt
    subject_alt_name: 
      - "DNS:*.domainname.com"
      - "DNS:domainname.com"
    privatekey_path:    /etc/ssl/private/nginx-selfsigned.key
```

Create certificates:

```
- name: Create Certificate
  openssl_certificate:
    csr_path:         /etc/ssl/certs/nginx-selfsigned.crt
    path:             /etc/ssl/certs/nginx-selfsigned.crt
    provider:         selfsigned
    privatekey_path:  /etc/ssl/private/nginx-selfsigned.ke
```

Create Delfie Hellman parameters:

```
- name: Generate DH Parameters
  openssl_dhparam:
    path: /etc/nginx/dhparam.pem
    size: 4096
```

Create Nginx SSL self-signed configuration file:

```
- name: Check Nginx SSL Self-signed Configuration
  blockinfile:
    path:   /etc/nginx/snippets/self-signed.conf
    create: yes
    block: |
      ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
      ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
```

Create Nginx parameters file:

```
- name: Check Nginx SSL Parameters File
  blockinfile:
    path:   /etc/nginx/snippets/ssl-params.conf
    create: yes
    block: |
      ssl_protocols TLSv1.2;
      ssl_prefer_server_ciphers on;
      ssl_dhparam /etc/nginx/dhparam.pem;
      ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384;
      ssl_ecdh_curve secp384r1; # Requires nginx >= 1.1.0
      ssl_session_timeout  10m;
      ssl_session_cache shared:SSL:10m;
      ssl_session_tickets off; # Requires nginx >= 1.5.9
      ssl_stapling on; # Requires nginx >= 1.3.7
      ssl_stapling_verify on; # Requires nginx => 1.3.7
      resolver {{ web_dns }} valid=300s;
      resolver_timeout 5s;
      # Disable strict transport security for now. You can uncomment the following
      # line if you understand the implications.
      # add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";
      add_header X-Frame-Options DENY;
      add_header X-Content-Type-Options nosniff;
      add_header X-XSS-Protection "1; mode=block";
```

Create Nginx SSL self-signed site configuration file:

```
- name: Check Nginx Site File
  blockinfile:
    path:   /etc/nginx/sites-available/self-signed
    create: yes
    block: |
      server {
        listen 443 ssl;
        listen [::]:443 ssl;
        include snippets/self-signed.conf;
        include snippets/ssl-params.conf;

        server_name _;

        root /var/www/html;
        index index.html index.htm index.nginx-debian.html;

        location / {
          try_files $uri $uri/ =404;
        }
  register: restart_nginx

- name: Check Nginx Sites File Symlink 
  file:
    src:    /etc/nginx/sites-available/self-signed
    dest:   /etc/nginx/sites-enabled/self-signed
    state:  link
    owner:  root
    group:  root
```

Restart Nginx:

```
- name: Restart Nginx Service
  service:
    name:    "nginx"
    enabled: yes
    state:   restarte
  when: restart_nginx.changed == true
```
