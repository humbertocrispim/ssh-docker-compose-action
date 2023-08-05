## Docker Compose Deployment (SSH)
Esta ação do GitHub proporciona a capacidade de executar o `docker-compose` em um servidor remoto por meio de uma conexão SSH. O processo envolve compactar o espaço de trabalho em um arquivo, transferi-lo via SSH para o servidor remoto e, posteriormente, executar o comando `docker-compose up -d`.

O diferencial desta ação em relação a outras similares é a sua independência de imagens Docker desconhecidas. A ação é construída a partir de um Dockerfile que utiliza a base `alpine:3.18`.

## Inputs

* `ssh_private_key` - Chave SSH privada para autenticação no sistema remoto. Aconselha-se manter essa chave segura nas secrets do GitHub.
* `ssh_host` - Nome do Host SSH.
* `ssh_port` - Porta remota, padrão é 22.
* `ssh_user` - Nome de usuário remoto com permissões para acessar o Docker.
* `docker_compose_prefix` - Prefixo do projeto passado para o `docker-compose`. Cada contêiner Docker será nomeado com esse prefixo.
* `docker_compose_filename` - Caminho do arquivo docker-compose no repositório.
* `use_stack` - Utilizar 'docker stack' em vez de 'docker-compose'.
* `pull` - Atualizar imagens ao realizar o pull, padrão é `false`.

# Exemplo de uso

Suponhamos que possuímos um repositório contendo apenas um arquivo `docker-compose`, e temos um servidor remoto baseado em Ubuntu com Docker e Docker Compose instalados.

Siga os passos abaixo:

1. Gere um par de chaves (não utilize uma senha):

```
ssh-keygen -t ed25519 -f ~/.ssh/deploy_key
```

2. Crie um usuário no servidor remoto que será responsável por implantar os containers. Não defina uma senha para este usuário:

```
ssh example.com
$ sudo useradd -m -b /var/lib -G docker docker-deploy
```

3. Permita o login neste usuário usando a chave gerada na etapa um:

```
scp deploy_key.pub example.com:~
ssh example.com
$ sudo mkdir /var/lib/docker-deploy/.ssh
$ sudo chown docker-deploy:docker-deploy /var/lib/docker-deploy/.ssh
$ sudo install -o docker-deploy -g docker-deploy -m 0600 deploy_key.pub /var/lib/docker-deploy/.ssh/authorized_keys
$ sudo chmod 0500 /var/lib/docker-deploy/.ssh
$ rm deploy_key.pub
```

4. Teste o acesso ao servidor:

```
ssh -i deploy_key docker-deploy@example.com
```

5. Adicione a chave privada e o nome de usuário nas secrets do repositório. Suponha que os nomes das secrets sejam `EXAMPLE_COM_SSH_PRIVATE_KEY` e `EXAMPLE_COM_SSH_USER`.

6. Remova sua cópia local da chave SSH:

```
rm deploy_key
```

7. Configure o workflow do GitHub Actions (por exemplo, `.github/workflows/main.yml`):

```yaml
name: Deploy

on:
  push:
    branches: [ master ]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v2

    - uses: humbertocrispim/github-action-ssh-docker-compose@v0.2.0-beta
      name: Implantação Remota com Docker-Compose
      with:
        ssh_host: example.com
        ssh_private_key: ${{ secrets.EXAMPLE_COM_SSH_PRIVATE_KEY }}
        ssh_user: ${{ secrets.EXAMPLE_COM_SSH_USER }}
        docker_compose_prefix: example_com
```

8. Tudo está pronto!

# Swarm & Stack

Se você deseja utilizar recursos avançados como secrets, é necessário configurar um cluster Docker Swarm e utilizar o comando 'docker stack' em vez do simples 'docker-compose'. Para isso, defina o parâmetro `use_stack` como `true`:

```yaml
name: Deploy
on:
  push:
    branches: [ master ]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
    - actions/chockout@v2

    - uses: humbertocrispim/github-action-ssh-docker-compose@v0.2.0-beta
      name: Implantação Remota com Docker-Stack
      with:
        ssh_host: example.com
        ssh_private_key: ${{ secrets.EXAMPLE_COM_SSH_PRIVATE_KEY }}
        ssh_user: ${{ secrets.EXAMPLE_COM_SSH_USER }}
        docker_compose_prefix: example.com
        use_stack: 'true'
        pull: true # Atualizar imagens ao fazer o pull
```

Isso deve proporcionar uma implantação eficiente e controlada de contêineres Docker em um ambiente remoto através do GitHub Actions.