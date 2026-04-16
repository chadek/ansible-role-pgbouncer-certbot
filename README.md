# ansible-role-pgbouncer-certbot

Ansible role to install and configure [PgBouncer](https://www.pgbouncer.org/) with TLS/SSL certificate management. Supports automatic certificate issuance via Certbot (DNS-01 challenge) or deployment of user-provided certificates. Includes optional certificate synchronization across load-balanced peers.

## Requirements

- Debian 12 (bookworm), Debian 13 (trixie), or Ubuntu 24.04 (noble)
- `community.crypto` Ansible collection (required only when using `pgbouncer_share_cert`)

## Role Variables

### PgBouncer configuration

```yaml
pgbouncer_conf:
  databases: {}
  pgbouncer:
    listen_addr: "*"
    listen_port: 5432
    auth_type: scram-sha-256
    auth_file: /etc/pgbouncer/userlist.txt
    admin_users: pgbouncer_admin
    stats_users: pgbouncer_stats
    pool_mode: session
    max_client_conn: 100
    default_pool_size: 20
    server_reset_query: DISCARD ALL
    logfile: /var/log/postgresql/pgbouncer.log
    pidfile: /var/run/postgresql/pgbouncer.pid
```

All keys under `pgbouncer_conf.pgbouncer` map directly to `pgbouncer.ini` parameters.

### User list

```yaml
pgbouncer_users: []
# - name: appuser
#   password: "secret"
```

### TLS

| Variable                          | Default                                 | Description                                 |
| --------------------------------- | --------------------------------------- | ------------------------------------------- |
| `pgbouncer_tls_enabled`           | `false`                                 | Enable TLS                                  |
| `pgbouncer_tls_cert_mode`         | `certbot`                               | Certificate source: `certbot` or `provided` |
| `pgbouncer_tls_dir`               | `/etc/pgbouncer/ssl`                    | Certificate directory                       |
| `pgbouncer_tls_cert_file`         | `{{ pgbouncer_tls_dir }}/fullchain.pem` | Client certificate path                     |
| `pgbouncer_tls_key_file`          | `{{ pgbouncer_tls_dir }}/privkey.pem`   | Client private key path                     |
| `pgbouncer_tls_server_ca_enabled` | `false`                                 | Enable backend server CA verification       |
| `pgbouncer_tls_server_ca_dest`    | `{{ pgbouncer_tls_dir }}/server-ca.pem` | Backend CA path                             |
| `pgbouncer_tls_server_ca_mode`    | `url`                                   | CA source: `url`, `content`, or `file`      |

### Certbot (when `pgbouncer_tls_cert_mode: certbot`)

| Variable                                    | Default                 | Description                                          |
| ------------------------------------------- | ----------------------- | ---------------------------------------------------- |
| `pgbouncer_domain_name`                     | `db.example.com`        | Domain to issue a certificate for                    |
| `pgbouncer_certbot_email`                   | `some.mail@example.com` | Let's Encrypt account email                          |
| `pgbouncer_certbot_dns_provider`            | —                       | DNS plugin name, e.g. `ovh`, `cloudflare`, `route53` |
| `pgbouncer_certbot_dns_credentials`         | —                       | DNS credentials file content (INI format)            |
| `pgbouncer_certbot_dns_propagation_seconds` | `30`                    | DNS propagation wait time                            |

### User-provided certificates (when `pgbouncer_tls_cert_mode: provided`)

```yaml
pgbouncer_tls_provided_cert: /path/to/cert.pem
pgbouncer_tls_provided_key: /path/to/key.pem
pgbouncer_tls_provided_ca: /path/to/ca.pem # optional
pgbouncer_tls_provided_remote_src: false # set true if files already on the target host
```

### Backend server CA

```yaml
# Download from URL (e.g. AWS RDS bundle)
pgbouncer_tls_server_ca_mode: url
pgbouncer_tls_server_ca_url: https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem

# Inline PEM content
pgbouncer_tls_server_ca_mode: content
pgbouncer_tls_server_ca_content: |
  -----BEGIN CERTIFICATE-----
  ...

# Copy from file
pgbouncer_tls_server_ca_mode: file
pgbouncer_tls_server_ca_src: /path/to/ca.pem
pgbouncer_tls_server_ca_remote_src: false
```

### Certificate synchronization (HA / load-balanced)

When defined, the role sets up rsync-over-SSH to replicate certificates from the Certbot master to backup peers after each renewal.

```yaml
pgbouncer_share_cert:
  state: master # or 'backup'
  peers:
    - 192.168.1.10
    - 192.168.1.11
```

- **master**: issues the certificate, syncs `/etc/pgbouncer/ssl/` to all peers via weekly cron and Certbot renewal hook.
- **backup**: receives the sync via rrsync (SSH key restricted to the certificate directory), reloads PgBouncer monthly.

## Dependencies

None.

## Example Playbooks

### Basic setup (no TLS)

```yaml
- hosts: pgbouncer_servers
  become: true
  roles:
    - ansible-role-pgbouncer-certbot
  vars:
    pgbouncer_conf:
      databases:
        mydb: "host=db.internal port=5432 dbname=mydb"
      pgbouncer:
        listen_port: 6432
        pool_mode: transaction
        max_client_conn: 200
    pgbouncer_users:
      - name: appuser
        password: "{{ vault_app_password }}"
```

### Certbot with OVH DNS challenge

```yaml
- hosts: pgbouncer_servers
  become: true
  roles:
    - ansible-role-pgbouncer-certbot
  vars:
    pgbouncer_domain_name: pgbouncer.company.com
    pgbouncer_certbot_email: ops@company.com
    pgbouncer_certbot_dns_provider: ovh
    pgbouncer_certbot_dns_credentials: |
      dns_ovh_endpoint = ovh-eu
      dns_ovh_application_key = xxxxxxxxxxxxxxxx
      dns_ovh_application_secret = xxxxxxxxxxxxxxxx
      dns_ovh_consumer_key = xxxxxxxxxxxxxxxx
    pgbouncer_tls_enabled: true
    pgbouncer_tls_cert_mode: certbot
    pgbouncer_conf:
      databases:
        "*": "host=postgres.internal port=5432"
      pgbouncer:
        listen_addr: "0.0.0.0, ::"
        listen_port: 5432
        client_tls_sslmode: require
        client_tls_cert_file: /etc/pgbouncer/ssl/fullchain.pem
        client_tls_key_file: /etc/pgbouncer/ssl/privkey.pem
    pgbouncer_users:
      - name: appuser
        password: "{{ vault_app_password }}"
```

### User-provided certificates with AWS RDS backend CA

```yaml
- hosts: pgbouncer_servers
  become: true
  roles:
    - ansible-role-pgbouncer-certbot
  vars:
    pgbouncer_tls_enabled: true
    pgbouncer_tls_cert_mode: provided
    pgbouncer_tls_provided_cert: /etc/pgbouncer/certs/server.crt
    pgbouncer_tls_provided_key: /etc/pgbouncer/certs/server.key
    pgbouncer_tls_provided_remote_src: true
    pgbouncer_tls_server_ca_enabled: true
    pgbouncer_tls_server_ca_mode: url
    pgbouncer_tls_server_ca_url: https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
    pgbouncer_conf:
      databases:
        "*": "host=mydb.xxxx.rds.amazonaws.com port=5432"
      pgbouncer:
        listen_addr: "127.0.0.1"
        listen_port: 5432
        server_tls_sslmode: verify-full
        server_tls_ca_file: /etc/pgbouncer/ssl/server-ca.pem
        client_tls_sslmode: require
        client_tls_cert_file: /etc/pgbouncer/ssl/fullchain.pem
        client_tls_key_file: /etc/pgbouncer/ssl/privkey.pem
    pgbouncer_users:
      - name: rds_user
        password: "{{ vault_rds_password }}"
```

### High-availability certificate sync

**Master:**

```yaml
- hosts: pgbouncer_master
  become: true
  roles:
    - ansible-role-pgbouncer-certbot
  vars:
    pgbouncer_domain_name: pgbouncer.company.com
    pgbouncer_certbot_email: ops@company.com
    pgbouncer_certbot_dns_provider: cloudflare
    pgbouncer_certbot_dns_credentials: |
      dns_cloudflare_api_token = xxxxxxxxxxxx
    pgbouncer_tls_enabled: true
    pgbouncer_tls_cert_mode: certbot
    pgbouncer_share_cert:
      state: master
      peers:
        - 192.168.1.10
        - 192.168.1.11
    pgbouncer_conf:
      databases:
        "*": "host=db.internal port=5432"
      pgbouncer:
        listen_addr: "0.0.0.0"
        listen_port: 5432
        client_tls_sslmode: require
        client_tls_cert_file: /etc/pgbouncer/ssl/fullchain.pem
        client_tls_key_file: /etc/pgbouncer/ssl/privkey.pem
    pgbouncer_users:
      - name: appuser
        password: "{{ vault_app_password }}"
```

**Backup nodes:**

```yaml
- hosts: pgbouncer_backup
  become: true
  roles:
    - ansible-role-pgbouncer-certbot
  vars:
    pgbouncer_share_cert:
      state: backup
      peers: []
    pgbouncer_conf:
      # same as master
```

## License

license GPL-3.0-only

## Author

Chadek
