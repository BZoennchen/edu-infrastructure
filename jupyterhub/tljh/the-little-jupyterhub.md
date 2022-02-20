# The Little JupyterHub (TLJH)

## Authentification 

### Shibboleth

A ``Shibboleth`` authentification with ``TLJH`` has been already estalbished by Rene Neuhaus (neuhaus@hm.edu) in his [thesis](./../../bachelorarbeit_neuhaus/Bachelorarbeit%20Rene%20Neuhaus.pdf).
He used Authentification via [jhub_shibboleth_auth](https://github.com/gesiscss/jhub_shibboleth_auth) which can also be used for a ``Kubernetes`` setup.

### OAuth

``OAhuth`` combined with ``JupyerHub`` is called ``OAuthenticator``.
An installation guide can be found [here](https://github.com/jupyterhub/oauthenticator).

## Customization

``TLJH`` maps its users to Unix users.
They are created automatically but to delete a user completely one has to delete both types of users and may clean up the home directory of the Unix user.

### TLJH Users

Each JupyterHub user gets their own Unix user account created when they first start their server.
The user Unix user name is ``jupyter-<username>`` for a JupyterHub user ``<username>`` and a home directory is created ``/home/jupyter-<username>`` with default permissions ``o-rwx``. 
No password is set for the Unix user by default and all users created by ``TLJH`` are added to the user group ``jupyterhub-users``.

### TLJH Admins

Admins are added to the user group ``jupyterhub-admins`` which is granted **complete root** access to the whole server with the ``sudo`` command on the terminal.
No password required.

## Configure Ressource Limits

Per default there is no **memory limit** or **cpu usage** limit!

Enable it via
````
sudo tljh-config set limits.memory 4G
````

Disable it via
````
sudo tljh-config set limits.memory None
````

To limit the **cpu usage** one can spacify a percentage use, that is, 1 means one cpu and 0.5 one half of a cpu.

Enable it via
````
sudo tljh-config set limits.cpu 2
````

Disable it via
````
sudo tljh-config set limits.cpu None
````

Disable it 

## Ressource Suggestions

For more details [click here](https://tljh.jupyter.org/en/latest/howto/admin/resource-estimation.html)

### Memory

1.5GB RAM is the minimum for the TLJH. Then it is suggested to have: 

``(max concurrent users * max memory per user) + 128 MB``

To estimate the required ressources we could use [jupyter-resource-usage](https://github.com/jupyter-server/jupyter-resource-usage)

>A good rule of thumb is to take the maximum amount of memory you used during your session, and add 20-40% headroom for users to 'play around'. This is the maximum amount of memory that should be given to each user.

### CPU

It is suggested to have 

``(max concurrent users * max CPU usage per user) + 20%``

### Disk Space

``(total users * max disk usage per user) + 2GB``
