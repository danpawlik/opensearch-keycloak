# OpenSearch - Keycloak

Minimal working example of running OpenSearch with Keycloak.

> This setup is not meant to be used in production

> For a version using kubernetes: https://github.com/bob-california/opensearch-kubernetes

## Setup

### Environment variables

- KEYCLOAK_ADMIN_LOGIN: login for keycloak admin user
- KEYCLOAK_ADMIN_PASSWORD: password for keycloak admin user
- KEYCLOAK_URL: keycloak url (used in `setup_keycloak.sh`)
- KEYCLOAK_DASHBOARDS_CLIENT_ID: dashboards client_id in keycloak (used in `setup_keycloak.sh` and opensearch-dashboards.yml)
- KEYCLOAK_DASHBOARDS_CLIENT_SECRET: dashboards client_secret in keycloak (used in `setup_keycloak.sh` and opensearch-dashboards.yml)

### Certificates

Start by generating all the needed certificates:

```bash
chmod +x setup_certs.sh && ./setup_certs.sh
```

### OpenSearch ulimits

Then make sure to raise your host ulimits for OpenSearch:

```bash
sudo sysctl -w vm.max_map_count=262144
```

To make this change persistent, add the line `vm.max_map_count=262144` in the `/etc/sysctl.conf` file. Then execute `sudo sysctl -p`

### Keycloak

Now, you will need to setup keycloak. Start by launching the service: `docker-compose up -d keycloak`. You can now access the admin interface: https://172.17.0.1:8443. The keycloak admin panel is quite complex to take in hands so I created a script to create a client for OpenSearch.

```bash
chmod +x setup_keycloak.sh && ./setup_keycloak.sh
```

> :warning: Wait until keycloak is ready by checking the logs: `docker-compose logs -f`

Now that the client is created, we have one last thing to do, it is to crate a `Role Mapper`.

1. In Keycloak admin panel, click on `Clients` in the sidebar, then click on `opensearch-dashboards` client in the list of clients
2. Click on the `Mappers` tab
3. Click on `Add builtin` in the top right
4. At the bottom of the list select `realm roles` and click on `Add selected`
5. Click on the role you just created, change the `Token Claim Name` to `roles` and make sure that all options are `on`

## OpenSearch

Launch OpenSearch containers: `docker-compose up -d os01 os02 os03`. Then execute the security admin script to setup the security plugin of OpenSearch:

```bash
chmod +x security_admin.sh && ./security_admin.sh
```

## Run

Execute:

```bash
docker-compose up -d
```

Keycloak admin console: https://172.17.0.1:8443

OpenSearch Dashboards: https://172.17.0.1:5601

> Use the values in the `.env` to connect to both.

## Quick setup on Centos 8

```bash
sudo yum install -y yum-utils git jq
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce docker-ce-cli containerd.io
sudo systemctl start docker
git clone https://github.com/bob-california/opensearch-keycloak.git && cd opensearch-keycloak/
sudo curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod 0755 /usr/local/bin/docker-compose
sudo sysctl -w vm.max_map_count=262144

export PATH=/usr/local/bin/:$PATH

LOCAL_IP=$(ip route get 1.1.1.1 | grep -oP 'src \K\S+')
ls | xargs sed -i "s/172.17.0.1/$LOCAL_IP/g"
sed -i "s/172.17.0.1/$LOCAL_IP/g" .env

./setup_certs.sh
chmod 0777 -R certs

# docker login docker.io

/usr/local/bin/docker-compose up -d keycloak postgres
sleep 60
./setup_keycloak.sh

docker-compose up -d os01 os02 os03
sleep 60
./security_admin.sh

# ensure that all is fine
/usr/local/bin/docker-compose up -d
```

If you need to tunel the traffic via ssh:

```bash
sudo ssh -L 9200:localhost:9200 \
         -L 9300:localhost:9300 \
         -L 9600:localhost:9600 \
         -L 9650:localhost:9650 \
         -L 5601:localhost:5601 \
         -L 8080:localhost:8080 \
         -L 8443:localhost:8443 \
         -L 5432:localhost:5432 <YOUR HOST> -l centos
```
