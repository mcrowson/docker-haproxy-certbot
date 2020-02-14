# HAProxy with Certbot, on Docker

Docker container running Alpine Linux with HAProxy and Certbot. HAProxy is setup to use a 0 downtime reload method that queses requests when the HAProxy service is bounced as new certificates are added or existing certificates refreshed.  
See soft versions in the Dockerfile.

Based on [halsbox/docker-haproxy-certbot](https://github.com/halsbox/docker-haproxy-certbot), itself based on https://bitbucket.org/blanksy-docker/haproxy, itself based on [nmarus/docker-haproxy-certbot](https://github.com/nmarus/docker-haproxy-certbot).  
Possibly more info on origin repos.

## Usage

First some terminology...  
HAProxy is a reverse proxy load balancer among other things.  
Let's Encrypt is a service that allows the creation and renewal of SSL certificates at no cost through an API and with automatic authentication.  
Certbot is a Linux CLI tool for interfacing with the Let's Encrypt API.

Certbot contains it's own http/https server and handles the authorization process from Let's Encrypt. This container is setup using HAProxy to redirect the Let's Encrypt callbacks (authentication) to the Certbot http server while all other requests are directed to the backend server(s).  
This configuration of HAProxy is also setup to do all the SSL termination so that your backend server(s) do not require a SSL configuration or certificates to be installed.

In order to use this in your environment, you must point all your SSL enabled domains to the IP Address of this container. This means updating the A records for these domains with your DNS provider. This includes the website name and all alternate names (i.e. example.com and www.example.com).  
After this is setup, an inbound request for your website(s) is initially received by HAProxy. If the request is part of the Let's Encrypt authentication process, it will redirect that traffic to the local instance of Certbot which is running on internal container ports 8080 and 8443. Otherwise it will pass through the request to a backend server (or servers) as defined in the haproxy.cfg file. The details of HAProxy setup are out of the scope for this README, but some examples are included below to get you started.

### Setup and Create Container

This will create the *haproxy-certbot* image.
```bash
docker build --no-cache -t haproxy-certbot https://github.com/lonequantum/docker-haproxy-certbot.git
```

This will create the *haproxy-certbot* container. Note that only the inbound ports for 80 and 443 are exposed.
```bash
docker run -d \
  --restart=always \
  --name haproxy-certbot \
  --cap-add=NET_ADMIN \
  -p 80:80 \
  -p 443:443 \
  -v /docker/haproxy/config:/config \
  -v /docker/haproxy/letsencrypt:/etc/letsencrypt \
  -v /docker/haproxy/certs.d:/usr/local/etc/haproxy/certs.d \
  haproxy-certbot:latest
```

It is important to note the mapping of the 3 volumes in the above command. This ensures that all persistent variable data is not maintained in the container itself.

The description of the 3 mapped volumes are as follows:

* `/config` - The configuration file location for HAProxy (haproxy.cfg).
* `/etc/letsencrypt` - The directory that Let's Encrypt will store it's configuration, certificates and private keys. **It is of significant importance that you maintain a backup of this folder in the event the data is lost or corrupted.**
* `/usr/local/etc/haproxy/certs.d` - The directory that this container will store the processed certs/keys from Let's Encrypt after they have been converted into a format that HAProxy can use. This is automatically done at each refresh and can also be manually initiated. This volume is not as important as the previous as the certs used by HAProxy can be regenerated again based on the contents of the letsencrypt folder.

### Container Helper Scripts

There are a handful of helper scripts to ease the amount of configuration parameters needed to administer this container.

#### Add a New Cert

This will add a new cert using a Certbot config that is compatible with the HAProxy config template below. After creating the cert, you should run the refresh script referenced below to initialize HAProxy to use it. After adding the cert and running the refresh script, no further action is needed.

```bash
# request certificate from Let's Encrypt
docker exec haproxy-certbot certbot-certonly \
  --domain example.com \
  --domain www.example.com \
  --email nmarus@gmail.com \
  --dry-run

# create/update HAProxy-formatted certs in certs.d and then restart HAProxy
docker exec haproxy-certbot haproxy-refresh
```

*After testing the setup, remove `--dry-run` to generate a live certificate.*

#### Renew a Cert

Renewing happens automatically but should you choose to renew manually, you can do the following.

```bash
docker exec haproxy-certbot certbot-renew --dry-run
```

*After testing the setup, remove `--dry-run` to refresh a live certificate.*

#### Create/Refresh Certs used by HAProxy from Let's Encrypt

This will parse and individually concatenate all the certs found in `/etc/letsencrypt/live` directory into the folder `/usr/local/etc/haproxy/certs.d`. It additionally will restart the HAProxy service so that the new certs are active.

When HAProxy is restarted, the system will queue requests using tc and libnl and minimal to 0 interruption of the HAProxy services is expected.  
See [this blog entry](https://engineeringblog.yelp.com/2015/04/true-zero-downtime-haproxy-reloads.html) for more details.

**Note: This process automatically happens whenever the cron job runs to refresh the certificates that have been registered.**

```bash
docker exec haproxy-certbot haproxy-refresh
```

#### Check an alternate configuration for HAProxy

This will check your changes in haproxy.cfg without using them for now:
```bash
docker exec haproxy-certbot haproxy-check
```

This will check a file named test.cfg (you must put it in the same directory as haproxy.cfg):
```bash
docker exec haproxy-certbot haproxy-check test.cfg
```

### Example haproxy.cfg, Using Cluster Backend

This example intercepts the Let's Encrypt validation and redirects to Certbot.  
Normal traffic is passed to the backend servers.  
If the request arrives as a http request, it is redirected to https. If there is not a certificate installed for the requested website, HAProxy will present a self signed default certificate. This behavior can be modified by adapting the haproxy config file if so desired.

This example also does not do any routing based on the URL. It assumes that all domains pointed to this HAProxy instance exist on the same backend server(s).  
The backend setup in this example consists of 3 web server that HAProxy will load balance against. If there is only a single server, or a different quantity this can be adjusted in the backend configuration block. This specific example would be a configuration that could be used in front of a PaaS cluster such as Flynn.io or Tsuru.io (both of which have their own http router in order to direct the traffic to the required application).

```
global
  maxconn 1028

  log 127.0.0.1 local0
  log 127.0.0.1 local1 notice

  ca-base /etc/ssl/certs
  crt-base /etc/ssl/private

  ssl-default-bind-ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+3DES:!aNULL:!MD5:!DSS
  ssl-default-bind-options no-sslv3


defaults
  log global
  mode http

  option httplog
  option dontlognull

  timeout connect 5000
  timeout client 10000
  timeout server 10000

  default-server init-addr none


#enable resolving throught docker dns and avoid crashing if service is down while proxy is starting
resolvers docker_resolver
  nameserver dns 127.0.0.11:53


frontend http-in
  bind 0.0.0.0:80
  mode http

  reqadd X-Forwarded-Proto:\ http

  acl letsencrypt_http_acl path_beg /.well-known/acme-challenge/
  redirect scheme https if !letsencrypt_http_acl
  use_backend letsencrypt_http if letsencrypt_http_acl

  default_backend my-default-backend

frontend https-in
  bind *:443 ssl crt /usr/local/etc/haproxy/default.pem crt /usr/local/etc/haproxy/certs.d ciphers ECDHE-RSA-AES256-SHA:RC4-SHA:RC4:HIGH:!MD5:!aNULL:!EDH:!AESGCM
  mode http

  reqadd X-Forwarded-Proto:\ https

  use_backend my-default-backend


backend letsencrypt_http
  mode http
  server letsencrypt_http_srv 127.0.0.1:8080


backend my-default-backend
  mode http
  balance roundrobin
  server application_node_1 docker_container_name_1:80 check inter 5s resolvers docker_resolver resolve-prefer ipv4
  server application_node_2 docker_container_name_2:80 check inter 5s resolvers docker_resolver resolve-prefer ipv4
```
