Docker-traefik
=============

Role to create deluge user/group and directory to use in the cluster.
Need to use the same UID and GID on all the system's user.
Based on the image `traefik:latest`.

Can use ACME (prod/stage) and cloudflare DNS chalenges 

Requirements
------------

- Docker and docker-compose installed (with [my role](https://galaxy.ansible.com/belgotux/docker) if needed [github](https://github.com/belgotux/ansible-role-docker))
- Ansible collection community.docker ([documentation](https://docs.ansible.com/ansible/latest/collections/community/docker/docker_compose_module.html)) : `ansible-galaxy collection install community.docker`

Role Variables
--------------
The role can work as it with the [default configuration](defaults/main.yml).

## variables required to change
- `traefik_domain` Main domaine of Traefik, to create a subdomain like xxx.yourdomain.tld (default `yourdomain.tld`)
- `traefik_mgt_users` list of user/password or user/passhash in apache format (more secured) to connect to traefik status webpage at https://traefik.yourdomain.tld (default: test/Test123* and admin/traefik)

## variables useful to change
- `traefik_mem_limit` docker memory limit (default `1g`)
- `traefik_image` image path (default `traefik:latest`)
- `traefik_logs` path to host for traefik log (default none, logs into the container, not permanent data)
- `traefik_mgt_certresolver` use `le` for let's encrypt https | `lestg` for staging let's encrypt | `lecf` for cloudflare dns for let's encrypt (default none and use default auto-sign traefik certificat or `default` store certificats)
- `traefik_cloudflare` allow to use dns verification for letsencrypt with cloudflare if your domain is managed on cloudflare (default `no`)
- `traefik_cloudflare_api_email` cloudflare email
- `traefik_cloudflare_api_token`: cloudflare token
- `traefik_letsencrypt` allow to use acme tlschallenge verification for letsencrypt in your container services (default `no`)
- `traefik_letsencrypt_api_email` your email
- `traefik_letsencrypt_staging`: allow to use STAGING acme tlschallenge verification for letsencrypt in your container services (default `no`)
- `traefik_custom_certs`: list of certificats in the following format. Please understand that the path is relative WITHIN the docker container to `/etc/traefik/certs/`. The mount point on the host is `{{traefik_dir}}/{{traefik_sub_part[0]}}`, with the default value typically `/etc/docker/traefik/certs`. One certificat by store is managed currently.
```
- name: xxx
  certFile: "xxx.cer"
  keyFile: "xxx.key"
```
Currently, only one store is usable and is named `default`. We can't use multi store at this note redaction [sources](https://doc.traefik.io/traefik/https/tls/#certificates-stores).

## variables (optionnal)
- `traefik_subdomain` subdomain to use for the Traefik management interface (default `traefik`)
- `traefik_service_name` set a container_name (default `traefik`)
- `traefik_port_https` listen port for https. Enabling traefik to listen to this port and doing port mapping for the host (default `443`) 
- `traefik_port_http` listen port for https. Enabling traefik to listen to this port and doing port mapping for the host (default none)
- `traefik_port_api_unsecured` listen port to access directly to the dashboard of Traefik **for testing only** (default none)
- `traefik_loglevel` set the loglevel like WARNING, INFO,... (default none)
- `traefik_sub_part` list of the 3 needed sub-directories, you can rename it but order is important (default `[certs,conf,secrets]`)
- `traefik_middlewares_enabled` list of 3 commons middlewares used that you can use in your container services : redirect all http to https trafic, secure https headers, whitelist all local IP addresses (actif by default, see `defaults/main.yml`)
- `traefik_service_options_extra` you can add extra options under the traefik service
- `traefik_network_name` your traefik network (default `proxy-net`)
- `traefik_exposed_by_default` Treafik service discovery take all containers and add it as a service automatically. Otherwize, you need to add explicitly `traefik.enable=true` as label on each container you want to expose in traefik, see [documentation](https://doc.traefik.io/traefik/providers/overview/#restrict-the-scope-of-service-discovery) (default `true`)

### generated vars
- `traefik_dir` is concatenation of `traefik_docker_conf`/`traefik_service_name` (default `/etc/docker/traefik`)

## TLS variables (optionnal)
- `traefik_tls_options_default` set the tls default options. [See official documentation](https://doc.traefik.io/traefik/https/tls/#minimum-tls-version) for more details (default `minVersion : VersionTLS12` to allow only min TLS 1.2 clients compatible)
- `traefik_tls_options` set other tls options (dafault none)
- `traefik_trust_https_selfsigned_backend` allow untrusted https backend, like a self-signed service, if you want to centralize the TLS layer in traefik (default `yes`)
- `traefik_tls_sni_strict` Traefik won't allow connections from clients that do not specify a server_name extension or don't match any of the configured certificates [see official documentation(https://doc.traefik.io/traefik/https/tls/#strict-sni-checking)] (default `false`)
- `traefik_custom_default_cert` you can speficied which of the custom certificat need to be used, same structure as `traefik_custom_certs`. By default the first one is used (`traefik_custom_certs[0]`)

## Docker vars (can be inherit for the docker role)
- `traefik_docker_group` your usual group to run a docker container (default `docker`)
- `traefik_docker_conf` the docker configurations's repository (default `/etc/docker`)
- `traefik_timezone` traefik_timezone for the container (default `Europe/Brussels`)


Additionnal traefik configuration files
---------------------------------------
You can add custom configuration files into `files/configurations` in yaml files and it will be copy automatically with other files

Dependencies
------------
- belgotux.docker

Example Playbook
----------------

```
- hosts: vps
  remote_user: root

  roles:
    - role: docker
    - role: docker_traefik
  variables:
    traefik_domain: vps.yourdomain.tld
    traefik_service_name: proxy
    traefik_mgt_certresolver: "le"
    traefik_network_name: proxy-net
    traefik_subdomain: proxymgt
    traefik_image: traefik:3.3
    traefik_letsencrypt: true
    traefik_letsencrypt_email: you@yourdomain.tld
    traefik_letsencrypt_staging: true
    traefik_middlewares_enabled:
      - secure-headers
    traefik_mem_limit: 100m
    traefik_loglevel: INFO
    traefik_mgt_users:
      - user: you
        passhash: $xxx
```

Example extra configuration file
--------------------------------

### For webserver running on another server
```
http:
  services:
    jeedom:
      loadBalancer:
        servers:
          - url: "http://10.10.10.10:80"
    jeedom-jeedomconnect-wss:
      loadBalancer:
        servers:
          - url: "ws://10.10.10.10:8100"
  routers:
    jeedom:
      entrypoints: websecure
      tls:
        certresolver: le
      middlewares: secure-headers@file
      rule: Host(`jeedom.domain.tld`) || Host(`jeedom.home.domain.tld`)
      service: jeedom@file
    jeedom-wss:
      entrypoints: websecure
      tls:
        certresolver: le
      rule: Host(`jeedom-wss.home.domain.tld`)
      service: jeedom-jeedomconnect-wss@file
```

### For extra middleware
Juste add another configuration file like this for nextcloud : the middleware `nextcloud-redirect` is declared and the middleware `nextcloud` is a chain of 2 middlewares
```
http:
  middlewares:
    nextcloud-redirect:
      redirectregex:
        regex: https://(.*)/.well-known/(card|cal)dav
        replacement: https://${1}/remote.php/dav
    nextcloud:
      chain:
        middlewares:
          - secure-headers
          - nextcloud-redirect
```

License
-------

[GPL v3](https://www.gnu.org/licenses/gpl-3.0.en.html)

Author Information
------------------

Belgotux
[MonLinux](https://www.monlinux.net)
