jupyterhub @ ocf
================

This repository contains the [JupyterHub][jupyterhub] configuration for its
[Kubernetes][k8s] deployment at the [Open Computing Facility][ocf] at UC
Berkeley.

JupyterHub spawns multiple [Jupyter Notebook][jupyternb] servers for multiple
users. It can be used for distributing code to students or research groups,
and allows users to execute and modify this code through a web browser,
without any setup.

This is based off of the [Zero to JupyterHub with Kubernetes][zero-to-jh-k8s]
distribution, which uses the [Helm][helm] package manager to install
a paramterized JupyterHub from a `config.yaml`.

However, at the OCF we've modified this setup to avoid using Helm, to better
suit our infrastructure and security needs.

## Architecture

JupyterHub runs three major processes:

- A **Hub** (Tornado process), which authenticates users and spawns Jupyter
  Notebook servers
- An **http proxy** (node-http-proxy), which directs users to the Hub at
  first, then forwards them to their Jupyter Notebook after login
- Multiple single-user **Jupyter Notebook** servers

## Configuration

JupyterHub loads its configuration from `jupyterhub_config.py`.

The full JupyterHub configuration reference can be found [here][jupyterhub-config],
with a sample [config.yaml][jupyterhub-config-yaml].

### Storage

The notebook storage space for each user is set by

```
c.Spawner.notebook_dir = "~/notebooks"
```
This can use shell expansion, so for instance this will expand to
each user's home directory.

[(source)][jupyterhub-spawner-dir]

The Hub also needs access to a persistent SQL database. Since we use MariaDB
at the OCF, we should set:

```py
c.JupyterHub.db_url = "mysql+pymysql://<db-username>:<db-password>@<db-hostname>:<db-port>/<db-name>"
```

[(source)][zero-to-jh-sql]

### Security

See JupyterHub's [Security Overview][jupyterhub-sec] and
[Security Settings][jupyterhub-sec-settings] for more info.

#### Cookie Secret

The cookie secret needs to be set as some random 32 bytes,
either through a cookie secret file:

```sh
openssl rand -hex 32 > /srv/jupyterhub/jupyterhub_cookie_secret
```

which then has the corresponding config:

```py
c.JupyterHub.cookie_secret_file = '/srv/jupyterhub/jupyterhub_cookie_secret'
```

or as an environment variable:

```sh
export JPY_COOKIE_SECRET=$(openssl rand -hex 32)
```

If the cookie secret changes, then all users will be logged out,
and all notebook servers need to be restarted.

#### Proxy auth token

There's also an auth token the Hub uses to authenticate with the proxy:
```py
c.JupyterHub.proxy_auth_token = '0deadbeef...'
```
```sh
export CONFIGPROXY_AUTH_TOKEN=$(openssl rand -hex 32)
```
If not set, the Hub will generate a random key on startup; which means
the *proxy must be restarted after the Hub is restarted*. By default,
this happens automatically as the proxy is a subprocess of the hub.

### Kubespawner

The Hub runs [Kubespawner][kubespawner], which spawns the Jupyter Notebook
servers as pods on a Kubernetes cluster.

The full Kubespawner settings can be found [here][kubespawner-docs]. However,
it's pretty extensive, so from Zero to Jupyterhub's
[jupyterhub_config.py][zero-to-jh-config], we've isolated the following
KubeSpawner parameters relevant to our setup:

```
c.KubeSpawner.namespace = os.environ.get('POD_NAMESPACE', 'default')
c.KubeSpawner.image_pull_policy
c.KubeSpawner.events_enabled = events
c.KubeSpawner.service_account = serviceAccountName
c.KubeSpawner.storage_extra_labels = storage.extraLabels
c.KubeSpawner.mem_limit = memory.limit
c.KubeSpawner.mem_guarantee = memory.guarantee
c.KubeSpawner.cpu_limit = cpu.limit
c.KubeSpawner.cpu_guarantee = cpu.guarantee
c.KubeSpawner.extra_resource_limits = extraResource.limits
c.KubeSpawner.extra_resource_guarantees = extraResource.guarantees
c.KubeSpawner.environment = extraEnv

c.KubeSpawner.image = singleuser.image.name
```

[helm]: https://helm.sh
[jupyterhub]: https://jupyterhub.readthedocs.io/en/stable/
[jupyterhub-config]: https://zero-to-jupyterhub.readthedocs.io/en/latest/reference.html
[jupyterhub-config-yaml]: https://github.com/jupyterhub/zero-to-jupyterhub-k8s/blob/master/jupyterhub/values.yaml
[jupyterhub-sec]: https://jupyterhub.readthedocs.io/en/stable/reference/websecurity.html
[jupyterhub-sec-settings]: https://jupyterhub.readthedocs.io/en/stable/getting-started/security-basics.html
[jupyterhub-spawner-dir]: https://jupyterhub.readthedocs.io/en/stable/getting-started/spawners-basics.html
[jupyternb]: https://jupyter-notebook.readthedocs.io/en/latest/

[k8s]: https://kubernetes.io
[kubespawner]: https://github.com/jupyterhub/kubespawner
[kubespawner-docs]: https://jupyterhub-kubespawner.readthedocs.io/en/latest/spawner.html
[ocf]: https://www.ocf.berkeley.edu/
[zero-to-jh-k8s]: https://zero-to-jupyterhub.readthedocs.io/en/latest/index.html
[zero-to-jh-config]: https://github.com/jupyterhub/zero-to-jupyterhub-k8s/blob/master/images/hub/jupyterhub_config.py#L91
[zero-to-jh-sql]: https://zero-to-jupyterhub.readthedocs.io/en/latest/reference.html?highlight=mysql#hub-db-type