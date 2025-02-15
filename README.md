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

### variables required to change
- `traefik_domain` Main domaine of Traefik, to create a subdomain like xxx.yourdomain.tld (default `yourdomain.tld`)
- `traefik_mgt_users` list of user/password or user/passhash in apache format (more secured) to connect to traefik status webpage at https://traefik.yourdomain.tld (default: test/Test123* and admin/traefik)

### variables useful to change
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
- `traefik_custom_certs`: list of certificats in the following format. Please understand that the path is relative WITHIN the docker container to `/etc/traefik/certs/`. The mount point on the host is `{{docker_traefik_dir}}/{{traefik_sub_part[0]}}`, with the default value typically `/etc/docker/traefik/certs`. One certificat by store is managed currently.
```
- name: xxx
  certFile: "xxx.cer"
  keyFile: "xxx.key"
```
Currently, only one store is usable and is named `default`. We can't use multi store at this note redaction [sources](https://doc.traefik.io/traefik/https/tls/#certificates-stores).

### variables (optionnal)
- `traefik_subdomain` subdomain to use for the Traefik management interface (default `traefik`)
- `traefik_service_name` set a container_name (default `traefik`)
- `traefik_port_https` listen port for https. Enabling traefik to listen to this port and doing port mapping for the host (default `443`) 
- `traefik_port_http` listen port for https. Enabling traefik to listen to this port and doing port mapping for the host (default none)
- `traefik_port_api_unsecured` listen port to access directly to the dashboard of Traefik **for testing only** (default none)
- `traefik_loglevel` set the loglevel like WARNING, INFO,... (default none)
- `traefik_tls_options_default` set the tls default options. [See official documentation](https://doc.traefik.io/traefik/https/tls/#minimum-tls-version) for more details (default `minVersion : VersionTLS12` to allow only min TLS 1.2 clients compatible)
- `traefik_tls_options` set other tls options (dafault none)
- `traefik_trust_https_selfsigned_backend` allow untrusted https backend, like a self-signed service, if you want to centralize the TLS layer in traefik (default `yes`)
- `traefik_tls_sni_strict` Traefik won't allow connections from clients that do not specify a server_name extension or don't match any of the configured certificates [see official documentation(https://doc.traefik.io/traefik/https/tls/#strict-sni-checking)] (default `false`)
- `traefik_sub_part` list of the 3 needed sub-directories, you can rename it but order is important (default `[certs,conf,secrets]`)
- `traefik_middlewares_enabled` list of 3 commons middlewares used that you can use in your container services : redirect all http to https trafic, secure https headers, whitelist all local IP addresses (actif by default, see `defaults/main.yml`)
- `traefik_service_options_extra` you can add extra options under the traefik service
- `traefik_custom_default_cert` you can speficied which of the custom certificat need to be used, same structure as `traefik_custom_certs`. By default the first one is used (`traefik_custom_certs[0]`)

### Docker vars (can be inherit for the docker role)
- `docker_user` your usual user to run a docker container (by default the first user with uid 1000)
- `docker_group` your usual group to run a docker container (default `docker`)
- `docker_conf` the docker configurations's repository (default `/etc/docker`)
- `docker_swarm` activate docker swarm for network and Traefik mode (default no)
- `timezone` timezone for the container (default `Europe/Brussels`)

### Traefik role vars (can be use in roles that depend on traefik_role)
- `traefik_network_name` your traefik network (default `proxy-net`)

### generated vars
- `docker_traefik_dir` is concatenation of `docker_conf`/`traefik_service_name` (default `/etc/docker/traefik`)

Additionnal traefik configuration files
---------------------------------------
You can add custom copnfiguration files into `files/configurations` in yaml files and it will be copy automatically with other files

Dependencies
------------
- belgotux.docker

Example Playbook
----------------

```

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
