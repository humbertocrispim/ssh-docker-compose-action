# Run Docker Compose on a Remote Server through SSH with GitHub Actions

This GitHub action allows you to run `docker compose` on a remote server through an SSH connection. The process involves compressing the workspace into a file, transferring it via SSH to the remote server, and then running the `docker compose up -d` command.

This action stands out because it doesn't require the use of unknown Docker images. Instead, the action is built from a Dockerfile that uses the `alpine` base.

## Inputs

- `ssh_private_key` - Private SSH key for authentication on the remote system. It is recommended to keep this key secure in GitHub secrets.
- `ssh_host` - SSH Host Name.
- `ssh_port` - Remote port, default is 22.
- `ssh_user` - Remote username with permissions to access Docker.
- `docker_compose_prefix` - Project prefix passed to `docker-compose`. Each Docker container will be named with this prefix.
- `docker_compose_filename` - Path to the docker-compose file in the repository.
- `use_stack` - Use 'docker stack' instead of 'docker-compose'.
- `pull` - Update images when performing a pull, default is `false`.

# Usage Example

Let's assume we have a repository containing only a `docker-compose` file, and we have a remote server based on Ubuntu with Docker and Docker Compose installed.

Follow these steps:

1. Generate a key pair (do not use a password):

```
ssh-keygen -t ed25519 -f ~/.ssh/deploy_key

```

1. Create a user on the remote server that will be responsible for deploying the containers. Do not set a password for this user:

```
ssh example.com
$ sudo useradd -m -b /var/lib -G docker docker-deploy

```

1. Allow login to this user using the key generated in step one:

```
scp deploy_key.pub example.com:~
ssh example.com
$ sudo mkdir /var/lib/docker-deploy/.ssh
$ sudo chown docker-deploy:docker-deploy /var/lib/docker-deploy/.ssh
$ sudo install -o docker-deploy -g docker-deploy -m 0600 deploy_key.pub /var/lib/docker-deploy/.ssh/authorized_keys
$ sudo chmod 0500 /var/lib/docker-deploy/.ssh
$ rm deploy_key.pub

```

1. Test access to the server:

```
ssh -i deploy_key docker-deploy@example.com

```

1. Add the private key and username to the repository secrets. Suppose the names of the secrets are `EXAMPLE_COM_SSH_PRIVATE_KEY` and `EXAMPLE_COM_SSH_USER`.
2. Remove your local copy of the SSH key:

```
rm deploy_key

```

1. Configure the GitHub Actions workflow (for example, `.github/workflows/main.yml`):

```
name: Deploy

on:
  push:
    branches: [ master ]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v2

    - uses: humbertocrispim/ssh-docker-compose-action@v1.0.0
      name: Remote Deployment with Docker-Compose
      with:
        ssh_host: example.com
        ssh_private_key: ${{ secrets.EXAMPLE_COM_SSH_PRIVATE_KEY }}
        ssh_user: ${{ secrets.EXAMPLE_COM_SSH_USER }}
        docker_compose_prefix: example_com
        pull: true # Update images when pulling

```

1. Everything is set!

# Swarm & Stack

If you want to use advanced features such as secrets, you need to set up a Docker Swarm cluster and use the 'docker stack' command instead of the simple 'docker-compose'. To do this, set the `use_stack` parameter to `true`:

```
name: Deploy
on:
  push:
    branches: [ master ]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
    - actions/chockout@v2

    - uses: humbertocrispim/ssh-docker-compose-action@v1.0.0
      name: Remote Deployment with Docker-Stack
      with:
        ssh_host: example.com
        ssh_private_key: ${{ secrets.EXAMPLE_COM_SSH_PRIVATE_KEY }}
        ssh_user: ${{ secrets.EXAMPLE_COM_SSH_USER }}
        docker_compose_prefix: example.com
        use_stack: 'true'
```