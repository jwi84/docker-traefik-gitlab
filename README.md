# docker-traefik-gitlab
GitLab &amp; GitLab Registry & GitLab Runner Behind Traefik in Docker

# Prerequisites
- Ubuntu 20.04 (Recommended)
- Docker
- Docker-Compose
- Traefik 2+
- FQDN (gitlab.example.com) (registry.gitlab.example.com)
- Change ssh port 22 to 2222
- Enable port 22 entrypoint for Traefik

# Installation

```
docker network create traefik-public
```

```
version: '3.3'
services:
  gitlab:
    image: 'gitlab/gitlab-ce:latest'
    container_name: gitlab
    restart: always
    labels:
      - traefik.enable=true
      - traefik.docker.network=traefik-public
      - traefik.constraint-label=traefik-public
      - traefik.http.routers.gitlab-http.rule=Host(`gitlab.example.com`)
      - traefik.http.routers.gitlab-http.entrypoints=http
      - traefik.http.routers.gitlab-http.middlewares=https-redirect
      - traefik.http.routers.gitlab-https.rule=Host(`gitlab.example.com`)
      - traefik.http.routers.gitlab-https.entrypoints=https
      - traefik.http.routers.gitlab-https.tls=true
      - traefik.http.routers.gitlab-https.tls.certresolver=le
      - traefik.http.services.gitlab.loadbalancer.server.port=80
      - traefik.tcp.routers.gitlab-ssh.rule=HostSNI(`*`)
      - traefik.tcp.routers.gitlab-ssh.entrypoints=ssh
      - traefik.tcp.routers.gitlab-ssh.service=gitlab-ssh-svc
      - traefik.tcp.services.gitlab-ssh-svc.loadbalancer.server.port=22
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        # Server Configuration
        external_url 'https://gitlab.example.com'
        nginx['listen_https'] = false
        nginx['listen_port'] = 80
        nginx['proxy_set_headers'] = {
          "X-Forwarded-Proto" => "https",
          "X-Forwarded-SSL" => "on"
        }
        # Database Configuration
        gitlab_rails['db_adapter'] = "postgresql"
        gitlab_rails['db_database'] = "gitlab"
        gitlab_rails['db_username'] = "postgres"
        gitlab_rails['db_password'] = "password"
        gitlab_rails['db_host'] = "gitlab-postgres"
        
        # Gitlab Registry
        registry['enable'] = false
        gitlab_rails['registry_enabled'] = true
        gitlab_rails['registry_host'] = "registry.gitlab.example.com"                      
        gitlab_rails['registry_api_url'] = "https://registry.gitlab.example.com"    
        gitlab_rails['registry_issuer'] = "gitlab-issuer"
 
        # GitLab SSH
        gitlab_rails['gitlab_shell_ssh_port'] = 22
    volumes:
      - './config:/etc/gitlab'
      - './logs:/var/log/gitlab'
      - './data:/var/opt/gitlab'
    networks:
      - traefik-public
    depends_on:
      - gitlab-postgres
 
  registry:
    restart: unless-stopped
    image: registry:2.7
    container_name: gitlab_registry
    volumes:
     - ./data:/registry
     - ./certs:/certs
    labels:
      - traefik.enable=true
      - traefik.docker.network=traefik-public
      - traefik.constraint-label=traefik-public
      - traefik.http.routers.registry-http.rule=Host(`registry.gitlab.example.com`)
      - traefik.http.routers.registry-http.entrypoints=http
      - traefik.http.routers.registry-http.middlewares=https-redirect
      - traefik.http.routers.registry-https.rule=Host(`registry.gitlab.example.com`)
      - traefik.http.routers.registry-https.entrypoints=https
      - traefik.http.routers.registry-https.tls=true
      - traefik.http.routers.registry-https.tls.certresolver=le
      - traefik.http.services.registry.loadbalancer.server.port=5000
    networks:
      - traefik-public
    environment:
      REGISTRY_LOG_LEVEL: debug
      REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /registry
      REGISTRY_AUTH_TOKEN_REALM: https://gitlab.example.com/jwt/auth
      REGISTRY_AUTH_TOKEN_SERVICE: container_registry
      REGISTRY_AUTH_TOKEN_ISSUER: gitlab-issuer
      REGISTRY_AUTH_TOKEN_ROOTCERTBUNDLE: /certs/gitlab-registry.crt
      REGISTRY_STORAGE_DELETE_ENABLED: 'true' 
 
  gitlab-postgres:
    image: postgres:13
    restart: always
    environment:
      POSTGRES_PASSWORD: "password"
      POSTGRES_USER: postgres
      POSTGRES_DB: gitlab
    volumes:
      - './database:/var/lib/postgresql/data'
    networks:
      - traefik-public
 
networks:
  traefik-public:
    external: true
```

Deploy:

```
docker-compose up -d
```

# Configuring GitLab Registry

GitLab itself needs some time for the bootstrap process. We can check the status with ```docker-compose logs -f```

Don’t worry if the registry container is hanging in a restart loop; we’ll get to that.

For GitLab and our Docker Image Registry to communicate with each other, we need a shared certificate. The good thing is, GitLab creates a key for us at bootstrap, only the registry doesn’t know about it yet.

What we have to do is copy the certificate key created by GitLab into the volume of the registry and create a certificate out of it.

```
cd certs
```
```
docker cp gitlab:/var/opt/gitlab/gitlab-rails/etc/gitlab-registry.key .
openssl req  -key gitlab-registry.key -new -subj "/CN=gitlab-issuer" -x509 -days 365 -out gitlab-registry.crt
```
After a minute the docker image registry should now be available, and we can use it with GitLab.

# GitLab Runner

Login into your root account and navigate to /admin/runners for example https://gitlab.example.com/admin/runners and copy down your registration token

Create a runners directory

```
sudo mkdir runners && cd runners
```

Create docker-compose.yml

```
version: '3.3'
services:
  gitlab_runner:
    container_name: gitlab_runner
    image: 'gitlab/gitlab-runner:latest'
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./config:/etc/gitlab-runner
    networks:
      - traefik-public

networks:
  traefik-public:
    external: true
```

Deploy runner with

```
docker-compose up -d
```

# Runner Configuration

After you finish registration, the resulting configuration is written to your chosen configuration volume (for example, /srv/gitlab-runner/config) and is loaded by the runner using that configuration volume.

To register a runner using a Docker container:

```
docker exec -it gitlab_runner gitlab-runner register
```

You will be prompted with a series of questions.

After registration, navigate back to your admin area to find a registered runner:

# Privileged Container

Remember to set privileged = true at ```config/config.toml```


