# JupyterHub and Kubernetes

## Installation

First configure the ``config.yaml`` file the installation is based on.
Then install ``JupyterHub``via a ``helm-chart``.

### 1. Select or compose a Docker image

The installation is based on a Docker image.
By default this image is the prefdefined [base-notebook]().
Other already defined images can be selected from [here](https://jupyter-docker-stacks.readthedocs.io/en/latest/using/selecting.html#jupyter-base-notebook).

To select an image we modify the config file ``config.yaml``.
In this example we choose the ``jupyter/datascience-notebook`` image.

````yaml
singleuser:
  image:
    # You should replace the "latest" tag with a fixed version from:
    # https://hub.docker.com/r/jupyter/datascience-notebook/tags/
    # Inspect the Dockerfile at:
    # https://github.com/jupyter/docker-stacks/tree/HEAD/datascience-notebook/Dockerfile
    name: jupyter/datascience-notebook
    tag: latest
````

We could also build our own image for our needs and publish it.
The following is an example of a custom image that extends the ``jupyter/tensorflow-notebook`` Docker image and adds JupyterLab extension for Git [jupyterlab-git](https://github.com/jupyterlab/jupyterlab-git) and [nbgitpuller](https://github.com/jupyterhub/nbgitpuller).

````docker
FROM jupyter/tensorflow-notebook
USER root

RUN apt update &&
apt install --yes git

USER $NB_USER

RUN pip install nbgitpuller &&
jupyter serverextension enable --py nbgitpuller --sys-prefix &&
jupyter labextension install @jupyterlab/git &&
pip install jupyterlab-git &&
jupyter serverextension enable --py jupyterlab_git

ENV NBGITPULLER_APP lab

WORKDIR /home/jovyan/work
CMD [“start.sh”, “jupyter”, “lab”,"–notebook-dir=/home/jovyan/work"]
````

Or we can [use multiple profiles to let users select their environment](https://zero-to-jupyterhub.readthedocs.io/en/latest/jupyterhub/customizing/user-environment.html#multiple-profiles).

+ IPython (with )
+ IJava
+ IHaskell
+ Sagemath
+ Julia
+ Machine-Learning ([GPU-Jupyter](https://github.com/iot-salzburg/gpu-jupyter/))

In the following you can see an example of four profiles that lets users select the environment they wish to use.

````yaml
singleuser:
  # Defines the default image
  image:
    name: jupyter/minimal-notebook
    tag: 2343e33dec46
  profileList:
    - display_name: "Minimal environment"
      description: "To avoid too much bells and whistles: Python."
      default: true
    - display_name: "Datascience environment"
      description: "If you want the additional bells and whistles: Python, R, and Julia."
      kubespawner_override:
        image: jupyter/datascience-notebook:2343e33dec46
    - display_name: "Spark environment"
      description: "The Jupyter Stacks spark image!"
      kubespawner_override:
        image: jupyter/all-spark-notebook:2343e33dec46
    - display_name: "Learning Data Science"
      description: "Datascience Environment with Sample Notebooks"
      kubespawner_override:
        image: jupyter/datascience-notebook:2343e33dec46
        lifecycle_hooks:
          postStart:
            exec:
              command:
                - "sh"
                - "-c"
                - >
                  gitpuller https://github.com/data-8/materials-fa17 master materials-fa;
````

By this config, the ``user`` can choose between the following servers:

![Auswahl](./k8s-multi-profiles.png)

**Note:** The user starts a pod using one of the profiles. It seems that users can only choose between profiles if there is no server running for them. They can shut down their sever at ``hub/home`` and then restart their server.


### 2. Install JupyterHub based on the image

````
helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/
helm repo update

helm upgrade --cleanup-on-fail \
  --install <helm-release-name> jupyterhub/jupyterhub \
  --namespace <k8s-namespace> \
  --create-namespace \
  --version=<chart-version> \
  --values config.yaml
````

The ``proxy-public`` can be identified by

````
kubectl get service --namespace <k8s-namespace>
````

**Note:** it seems to be the case that we can change the image at any time, i.e., after the installtion already finisehd and we already working with the hub.

## Authentification 

### Shibboleth

Authentification via [jhub_shibboleth_auth](https://github.com/gesiscss/jhub_shibboleth_auth).
Extend the [zero-to-jupyterhub-k8s](https://github.com/jupyterhub/zero-to-jupyterhub-k8s/blob/main/images/hub/Dockerfile) Dockerfile

````docker
# https://github.com/jupyterhub/jupyterhub/blob/0.9.6/Dockerfile
FROM jupyterhub/jupyterhub:0.9.6

RUN pip install --no-cache-dir jhub_shibboleth_auth==1.5.0
````

to install [jhub_shibboleth_auth](https://github.com/gesiscss/jhub_shibboleth_auth).
Activate the authentification by updating the ``config.yaml`` file with

````yaml
hub:
  image:
    name: gesiscss/k8s-hub
    tag: 0.8.2-v1.5.0
  extraConfig: |
    c.Authenticator.shibboleth_logout_url = 'your Shibboleht logout url'
auth:
  type: custom
  custom:
    className: "jhub_shibboleth_auth.shibboleth_auth.ShibbolethAuthenticator"
  state:
    enabled: true
````

### Security

There are two ways to specify your manual certificate, directly in the ``config.yaml`` or by creating a [Kubernetes **secret**](https://kubernetes.io/docs/concepts/configuration/secret/).

Security considerations can be found [here](https://zero-to-jupyterhub.readthedocs.io/en/1.0.0-beta.1/administrator/security.html).

#### Specify certificate in config.yaml

1. Add your domain name & HTTPS certificate info to your ``config.yaml``
    ````yaml
    proxy:
    https:
      enabled: true
      type: manual
      manual:
        key: |
          -----BEGIN RSA PRIVATE KEY-----
          ...
          -----END RSA PRIVATE KEY-----
        cert: |
          -----BEGIN CERTIFICATE-----
          ...
          -----END CERTIFICATE-----
    ````
2. Apply the config changes by running helm upgrade
3. Wait for about a minute, now your hub should be HTTPS enabled!

#### Specify certificate through Secret resource

1. Create a ``secret`` resource with type ``kubernetes.io/tls`` containing your certificate.
   ````kubectl create secret tls example-tls --key="tls.key" --cert="tls.crt"````
2. Add your domain and the name of your ``secret`` to your ``config.yaml``.
    ````yaml
    proxy:
    https:
      enabled: true
      hosts:
        - <your-domain-name>
      type: secret
      secret:
        name: example-tls
    ````
3. Apply the config changes by running helm upgrade
4. Wait for about a minute, now your hub should be HTTPS enabled!

## Customization

To apply any modification of Kubernetes via the ``config.yaml`` one has to:

1. make the necessary changes inside the ``config.yaml``
2. Run ``helm upgrade``: 
    ````
    helm upgrade --cleanup-on-fail \
    <helm-release-name> jupyterhub/jupyterhub \
    --namespace <k8s-namespace> \
    --version=<chart-version> \
    --values config.yaml
    ````
    in my current case:
    ````
    helm upgrade --cleanup-on-fail \
    jupyterhub jupyterhub/jupyterhub \
    --namespace jhub \
    --version=0.11.1 \
    --values config.yaml
    ````
3. Verify that the *hub* and *proxy* pods entered the ``Running`` state after the upgrade completed:
    ````
    kubectl get pod --namespace jhub
    ````

For some reason I currently have to execute 

````
helm rollback jupyterhub --namespace jhub
````

before updating.

### Memory and CPU guarantees / limits

````yaml
singleuser:
  memory:
    limit: 1G
    guarantee: 1G
  cpu:
    limit: .5
    guarantee: .5
````

GPU allocation is also possible and for memory intensive work, it is beneficial to increase the shared memory (SHM) allocation these pods.

### Storage

JupyterHub uses Kubernetes to manage user storage, i.e. ``PersistentVolumeClaim`` (``PVC``) and ``PersistentVolume`` (``PV``).
Each user gets their own ``PersistentVolumeClaim`` object, representing the data attached to their account.
We can configure the type of storage i.e. the [StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/) object.

````yaml
singleuser:
  storage:
    dynamic:
      storageClass: <storageclass-name>
````

We can also define the size of storage requested by JupyterHub in the ``PVC`` via

````yaml
singleuser:
  storage:
    capacity: 2Gi
````

the default is ``10Gi`` (``10 * 1024 * 1024 * 1024``) bytes.

Futhermore, we can turn off any persistent storage with

````yaml
singleuser:
  storage:
    type: none
````

which might be usful for the **Vorkurs**.

### repo2docker

We can use [repo2docker](https://github.com/jupyterhub/repo2docker) to convert a Git-Repository into a Docker image which we can then share within the JupyterHub deployment (as a base).