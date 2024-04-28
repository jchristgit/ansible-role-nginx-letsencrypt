# NGINX & Let's Encrypt Ansible Role

This role allows you to create SSL certificates for NGINX virtual hosts using
`certbot` and its [Webroot
plugin](https://eff-certbot.readthedocs.io/en/stable/using.html#webroot). This
role is only tested on Debian Stable.

## Usage

Include the `nginx-letsencrypt` role in the `dependencies` of the role that
needs it to set up the webroot directory and a NGINX configuration dropin that
you can use in your HTTP vhosts. The added configuration file can be used in
your NGINX HTTP virtual host as follows:

```nginx
server {
    listen          80;
    listen          [::]:80;

    server_name     my-website;

    include         letsencrypt.conf;  # <--

    location / {
        return 301      https://my-website$request_uri;
    }
}
```

Note that the role itself, e.g. the `main` task and not the included task
described below, does not use any configuration parameters except for
`nginx_letsencrypt_webroot_path`.

To fetch certificates, you first set up a HTTP vhost in your nginx
configuration to serve as a webroot directory, reload nginx, include the
`setup-certificate.yml` task, and then set up the HTTPS vhost:

```yaml
- name: set up nginx HTTP vhost
  template:
    src: my-website.http.conf.j2
    dest: /etc/nginx/conf.d/my-website.http.conf
    owner: root
    group: root
    mode: 0444
  notify: reload nginx

- meta: flush_handlers

- name: get certificates
  include_role:
    name: nginx-letsencrypt
    tasks_from: setup-certificate.yml
  vars:
    nginx_letsencrypt_email: mymail@example.com
    nginx_letsencrypt_domains:
      - example.com
    # nginx_letsencrypt_agree_tos: true

- name: set up nginx HTTPS vhost
  template:
    src: my-website.https.conf.j2
    dest: /etc/nginx/conf.d/my-website.https.conf
    owner: root
    group: root
    mode: 0444
  notify: reload nginx
```

The reason for splitting the HTTP and HTTPS vhosts is that if you were to
configure both at the same time and try to reload nginx, it would fail on the
initial deploy due to the missing HTTPS certificate.

## Configuration

### Optional variables

- `nginx_letsencrypt_agree_tos` (bool): Agree to the terms of service of Let's
  Encrypt. This is `false` by default.

- `nginx_letsencrypt_webroot_path` (string): Where to store the validation
  files created by certbot. Defaults to `/var/www/_letsencrypt`.

- `nginx_letsencrypt_domains` (list[string]): Domains for which to fetch
  certificates. The first one is used as the name for certificates. Note that
  subsequent updates of this list to add more domains will not work as expected,
  as the role uses the presence of certificates under the first name to check
  whether it needs to run `certbot`. Defaults to `[{{ ansible_fqdn }}`.

- `nginx_letsencrypt_email` (string): Where to send certificate notices to.
  Defaults to `webmaster@{{ nginx_letsencrypt_domains[0] }}` per [RFC
  2142](https://www.rfc-editor.org/rfc/rfc2142).

- `nginx_letsencrypt_reload_services` (list[string]): Which systemd services to
  reload after certificate updates. Defaults to `[nginx]`.

- `nginx_letsencrypt_reload_services` (list[string]): Which systemd services to
  restart after certificate updates. Defaults to `[]`.

<!-- vim: set textwidth=80 sw=2 ts=2: -->
