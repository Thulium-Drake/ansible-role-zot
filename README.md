# Ansible role: zot

Deploy the [zot](https://zotregistry.dev/) OCI-native container registry as a
Podman container managed by systemd Quadlet.

## Role variables

All variables are optional.

| Variable | Default | Description |
| --- | --- | --- |
| `zot_image` | `ghcr.io/project-zot/zot-linux-amd64:v2.1.22` | Container image and tag. Upstream publishes one image per architecture, so the default is the linux/amd64 image. |
| `zot_pull_policy` | `newer` | Podman pull policy: `always`, `missing`, `never`, or `newer`. |
| `zot_quadlet_options` | `[]` | Additional Quadlet options for the container unit. |
| `zot_listen_uri` | `127.0.0.1:5000` | Host publish address for the registry. |
| `zot_config_dir` | `/etc/zot` | Host directory holding `config.json`. |
| `zot_config` | see `defaults/main.yml` | Additional zot configuration, recursively merged beneath the role's minimal configuration and written to `config.json`. Role-managed keys (`distSpecVersion`, `storage.rootDirectory`, `http.address`, `http.port`, `http.readTimeout`, `http.writeTimeout`, `http.auth.htpasswd.path`, `log.level`) take precedence. See the [zot configuration reference](https://zotregistry.dev/v2.1.22/admin-guide/admin-configuration/). |
| `zot_user_account` | `zot` | Account that owns the configuration directory and runs the container. Use `root` for a system-scoped root deployment, or `null` to use the account Ansible executes as. |
| `zot_log_level` | `info` | Log level written to `log.level` in the generated configuration. Use this instead of `zot_config.log.level`, which the minimal configuration overrides. |
| `zot_http_read_timeout` | `30m` | Maximum duration for reading an entire request, including blob uploads. zot defaults to `60s`, which aborts large pushes. Use `0s` to disable the timeout. |
| `zot_http_write_timeout` | `30m` | Maximum duration for writing a response, including blob downloads. zot defaults to `60s`, which aborts large pulls. Use `0s` to disable the timeout. |
| `zot_apache_vhost` | `false` | Write an Apache httpd vhost that reverse proxies zot over HTTPS. The role does not install Apache; see [Apache httpd reverse proxy](#apache-httpd-reverse-proxy). |
| `zot_apache_server_name` | `{{ ansible_facts['fqdn'] }}` | Hostname the vhost answers on. It must be unique among the vhosts of the httpd instance. |
| `zot_apache_cert` | `/etc/ssl/server.crt` | TLS certificate used by the HTTPS vhost. |
| `zot_apache_key` | `/etc/ssl/server.key` | TLS key used by the HTTPS vhost. |
| `zot_apache_hsts` | `false` | Send the HSTS header from the HTTPS vhost. |
| `zot_apache_backend_url` | `http://{{ zot_listen_uri }}` | URL that Apache proxies requests to. |
| `zot_apache_proxy_timeout` | `1800` | Socket timeout in seconds for the proxied connection to zot. Keep it at least as large as the zot read/write timeouts. |
| `zot_apache_vhost_name` | `zot` | Base name of the vhost file, `<name>.conf`. |
| `zot_htpasswd_users` | `[]` | Local users for htpasswd authentication. Each entry needs a `name` and a `password`; hashes are written to `/etc/zot/htpasswd`, which always exists. Keep passwords in an encrypted source such as Ansible Vault. |
| `zot_sync_credentials` | `[]` | Credentials for upstream registries used by zot's sync extension. Each entry needs a `registry` host, a `username` and a `password`; the list is written as a JSON object to `zot_sync_credentials_file`. Keep passwords in an encrypted source such as Ansible Vault. See [Registry mirroring credentials](#registry-mirroring-credentials). |
| `zot_sync_credentials_file` | `{{ zot_config_dir }}/sync-credentials.json` | Host path of the generated sync credentials file. It must be inside `zot_config_dir`, which is mounted at `/etc/zot` in the container. |
| `zot_debug` | `false` | Disables `no_log` on the container and systemd tasks. Enable it only temporarily while troubleshooting. |

### Configuration handling

The role starts from a minimal zot configuration defined in `vars/main.yml`.
`zot_config` is recursively combined beneath it and written as JSON to
`<zot_config_dir>/config.json`; where both define the same key, the minimal
configuration wins, while non-conflicting user keys are kept. The file is
regenerated and validated on every run, so direct edits to it are overwritten.

### Local users (htpasswd)

Each entry in `zot_htpasswd_users` needs a `name` and a `password`. Passwords
are hashed and the salt is derived from the user name, keeping the file unchanged
across runs unless a user or password changes. zot watches the file and reloads
it without a restart.

```yaml
zot_htpasswd_users:
  - name: 'alice'
    password: "{{ vault_zot_alice_password }}"
```

Note that an empty file leaves no user able to authenticate, so define at least one user
before using the registry.

### Registry mirroring credentials

zot's [sync extension](https://zotregistry.dev/v2.1.23/articles/mirroring/)
reads credentials for upstream registries from the JSON file referenced by
`extensions.sync.credentialsFile`. The role maintains that file at
`zot_sync_credentials_file`, which defaults to
`<zot_config_dir>/sync-credentials.json` on the host and `/etc/zot/sync-credentials.json`
inside the container.

Define `zot_sync_credentials` as a list of upstream registries, each with a
`registry` host, a `username` and a `password`, and reference the container path
from `zot_config`. The role converts the list to the JSON object zot expects, so
the registry name may come from a variable:

```yaml
zot_sync_credentials:
  - registry: 'registry1:5000'
    username: 'user'
    password: "{{ vault_registry1_password }}"
  - registry: "{{ upstream_registry }}"
    username: "{{ upstream_user }}"
    password: "{{ vault_upstream_password }}"

zot_config:
  extensions:
    sync:
      enable: true
      credentialsFile: '/etc/zot/sync-credentials.json'
      registries:
        - urls:
            - 'https://registry1:5000'
          onDemand: false
          pollInterval: '6h'
          tlsVerify: true
          content:
            - prefix: '/repo1/repo'
            - prefix: '/repo2/repo'
              destination: '/repo2'
              stripPrefix: true
        - urls:
            - "https://{{ upstream_registry }}"
          onDemand: false
          pollInterval: '12h'
          tlsVerify: true
          content:
            - prefix: '**'
```

Enable `zot_config.http.compat: ["docker2s2"]` when an upstream registry can
return Docker schema 2 content. See the upstream
[polled mirroring example](https://zotregistry.dev/v2.1.23/articles/mirroring/#example-multiple-repositories-with-polled-mirroring)
for all available sync options.

The file is always written, even when `zot_sync_credentials` is empty, so a
`credentialsFile` referenced from the configuration exists during validation.
Changes to the file restart zot.

### Data and service lifecycle

Registry data is stored in the Podman volume `zot-data`. The role only ensures
the volume exists and never removes it, so repositories survive container
restarts and repeated role runs.

Changes to the container, volume, or configuration notify a handler that
restarts the `zot` systemd unit. The unit is enabled and started on every role
run.

### Non-root deployments

The role creates `zot_user_account` when it does not exist. It creates a regular
account (not a system account) so that `useradd` allocates the subordinate
UID/GID ranges rootless Podman requires. Lingering is enabled for the account
so the user-scoped unit starts at boot.

When using `null` to detect the ansible user account, `become` must be `false`.

### Apache httpd reverse proxy

The role does not install Apache httpd. When `zot_apache_vhost` is enabled it
writes a vhost into the configuration directory of an existing installation and
reloads the service if it is running.

The vhost proxies to `zot_apache_backend_url`, which defaults to the registry's
local publish address, so zot itself can stay bound to `127.0.0.1`. The
registry is always served over HTTPS: the port 80 vhost permanently redirects
to port 443.

The layout follows the
[netboot_server](https://github.com/Thulium-Drake/ansible-role-netboot_server)
role, so both roles can share one httpd instance. Give zot its own
`zot_apache_server_name` thas does not overlap with a different vhost.

Requirements on the existing installation:

- When using OpenID authentication behind the proxy, set
  `zot_config.http.externalUrl` to the public vhost URL so redirects point at
  Apache instead of the container.

#### Large images

zot applies a 60 second `readTimeout` and `writeTimeout` to its HTTP server by
default.  The role raises both zot timeouts to 30 minutes with
`zot_http_read_timeout` and `zot_http_write_timeout`, and gives the Apache
backend connection a matching `ProxyTimeout`, so large blobs can be
transferred.

## Example playbooks

Minimal deployment, bound to `127.0.0.1:5000` on the host:

```yaml
- name: 'Deploy zot'
  hosts: 'all'
  roles:
    - 'zot'
```

Expose the registry on all interfaces and customize the zot configuration:

```yaml
- name: 'Deploy zot registry'
  hosts: 'zot_servers'
  vars:
    zot_listen_uri: '0.0.0.0:5000'
    zot_config:
      storage:
        gc: true
        dedupe: true
  roles:
    - 'zot'
```

Reuse the Apache instance and TLS settings of the netboot_server role:

```yaml
- name: 'Deploy zot behind the netboot server'
  hosts: 'zot_servers'
  vars:
    zot_apache_vhost: true
    zot_apache_server_name: 'zot.deploy.example.nl'
  roles:
    - 'netboot_server'
    - 'zot'
```
