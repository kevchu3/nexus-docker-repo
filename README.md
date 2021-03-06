# Docker Repository with Nexus and Httpd

I followed these steps to set up both a Docker private (hosted) and proxy repository on Sonatype Nexus.

### 1. Installing Sonatype Nexus

Provision a virtual machine with 4 cores and 8 GB RAM according to minimum [system requirements].  Follow the instructions to [install Nexus], [install Java] and [run Nexus as a service].

### 2. Configure Private Docker Repository

From the Nexus UI, configure a private (hosted) or proxy [Docker repository] on Sonatype Nexus.  For the purposes of this demonstration, I will do both.

Configure Nexus security to enable Docker logins.  From the Nexus UI, navigate to **Security** -> **Realms** -> set **Docker Bearer Token Realm** to **Active**, and **Save**.

### 3. Install and configure httpd as reverse proxy

Download and set up Apache httpd as a reverse proxy.  This will allow a repository connector to be used on a separate port, reasoning below provided by [this article]:

> What is a Repository Connector?
> When you make a request using the Docker client, you provide a hostname and port followed by the Docker image. Docker does not support the use of a context to specify the path to the repository. 
>
> Does not work:
> `docker pull centos7:8081/repository/docker-group/postgres:9.4`
>
> Does work:
> `docker pull centos7:18080/postgres:9.4`
>
> Since we cannot include the repository name in the Docker client request, we use a Repository Connector to assign a port to the Docker repository which can be used in Docker client commands. The Repository Connector is found in the settings for each docker repository.

Follow the steps to [configure httpd as a reverse proxy].

Install needed packages: `yum install httpd mod_ssl mod_headers -y`

Edit the main configuration file: `vi /etc/httpd/conf/httpd.conf`.  For this example, I chose two arbitrary ports that were not in use (5002 and 5004) but you can pick your own.

```
Listen 5002
Listen 5004

# Docker
ProxyRequests Off
ProxyPreserveHost On

# Extend timeout value when pulling large blobs (i.e. Docker images)
RequestReadTimeout header=600 body=600

<VirtualHost *:5002>
  SSLEngine on
  SSLCertificateFile /etc/httpd/nexus.crt
  SSLCertificateKeyFile /etc/httpd/nexus.key

  ServerName nexus.example.com
  ServerAdmin admin.example.com

  AllowEncodedSlashes NoDecode

  ProxyPass / http://nexus.example.com:8081/repository/docker-proxy/ nocanon timeout=600
  ProxyPassReverse / http://nexus.example.com:8081/repository/docker-proxy/
  RequestHeader set X-Forwarded-Proto "https"

  ErrorLog logs/nexus.example.com/nexus-proxy/error.log
  CustomLog logs/nexus.example.com/nexus-proxy/access.log common
</VirtualHost>

<VirtualHost *:5004>
  SSLEngine on
  SSLCertificateFile /etc/httpd/nexus.crt
  SSLCertificateKeyFile /etc/httpd/nexus.key

  ServerName nexus.example.com
  ServerAdmin admin.example.com

  AllowEncodedSlashes NoDecode

  ProxyPass / http://nexus.example.com:8081/repository/docker-hosted/ nocanon timeout=600
  ProxyPassReverse / http://nexus.example.com:8081/repository/docker-hosted/
  RequestHeader set X-Forwarded-Proto "https"

  ErrorLog logs/nexus.example.com/nexus-hosted/error.log
  CustomLog logs/nexus.example.com/nexus-hosted/access.log common
</VirtualHost>
```

Create logs directory:
```
mkdir -p /var/log/httpd/nexus.example.com/nexus-hosted
mkdir -p /var/log/httpd/nexus.example.com/nexus-proxy
```

### 4. Generate and configure SSL certificates

Follow the steps to generate SSL certificates with SAN.
```
cd /etc/httpd
openssl req -addext "subjectAltName = DNS:nexus.example.com" \
            -addext "certificatePolicies = 1.2.3.4" \
            -newkey rsa:4096 -nodes -sha256 -keyout nexus.key -x509 -days 730 -out nexus.crt

... answer some questions to generate your cert ...

cat nexus.crt nexus.key > nexus.pem
```

Restore SELinux context after certificates have been placed:`restorecon -RvF /etc/httpd/`

Edit the SSL configuration file to point to your custom certificates: `vi /etc/httpd/conf.d/ssl.conf`
```
SSLCertificateFile /etc/httpd/nexus.crt
SSLCertificateKeyFile /etc/httpd/nexus.key
```
### 5. Set up whitelisting

For this example, I chose two arbitrary ports that were not in use (5002 and 5004) but you can pick your own.

```
# set SELinux permissions for non-standard ports 5002 and 5004
yum -y install policycoreutils-python-utils
semanage port -a -t http_port_t -p tcp 5002
semanage port -a -t http_port_t -p tcp 5004

# https://access.redhat.com/solutions/2980121
setsebool -P httpd_can_network_connect 1

# whitelist ports 5002 and 5004 in firewalld
firewall-cmd --permanent --add-port=5002/tcp
firewall-cmd --permanent --add-port=5004/tcp
firewall-cmd --reload

# enable and start httpd
systemctl enable httpd
systemctl restart httpd
```

### 6. Configure certificate on client

#### 6a. Configure Docker client

Configure your Docker client (i.e. your bastion host) to [trust self-signed certificate]. Copy nexus certificate to every host and OS certificate trust.  You do not need to restart Docker.

```
mkdir -p /etc/docker/certs.d/nexus.example.com:5002
cp nexus.crt /etc/docker/certs.d/nexus.example.com:5002/ca.crt

mkdir -p /etc/docker/certs.d/nexus.example.com:5004
cp nexus.crt /etc/docker/certs.d/nexus.example.com:5004/ca.crt

cp certs/nexus.crt /etc/pki/ca-trust/source/anchors/nexus.crt
update-ca-trust
```

#### 6b. Configure Podman client

The instructions are mostly the same as above, with the exception of the target directory.

```
mkdir -p /etc/containers/certs.d/nexus.example.com:5002
cp nexus.crt /etc/containers/certs.d/nexus.example.com:5002/ca.crt

mkdir -p /etc/containers/certs.d/nexus.example.com:5004
cp nexus.crt /etc/containers/certs.d/nexus.example.com:5004/ca.crt

cp certs/nexus.crt /etc/pki/ca-trust/source/anchors/nexus.crt
update-ca-trust
```

### 7. Examples of clients interacting with Nexus repository

* Log into Nexus and perform an image pull against proxy repository on port 5002: [see example](./examples/image-pull-through-proxy.txt)

* Mirror OpenShift content from quay.io to hosted repository on port 5004: [see example](./examples/oc-mirror-registry.txt)

[system requirements]: https://help.sonatype.com/repomanager3/system-requirements
[install Nexus]: https://help.sonatype.com/repomanager3/installation
[install Java]: https://help.sonatype.com/repomanager3/installation/java-runtime-environment
[run Nexus as a service]: https://help.sonatype.com/repomanager3/installation/run-as-a-service
[Docker repository]: https://blog.sonatype.com/using-nexus-3-as-your-repository-part-3-docker-images
[this article]: https://support.sonatype.com/hc/en-us/articles/115013153887-Docker-Repository-Configuration-and-Client-Connection
[configure httpd as a reverse proxy]: https://help.sonatype.com/repomanager3/installation/run-behind-a-reverse-proxy
[trust self-signed certificate]: https://docs.docker.com/registry/insecure/#use-self-signed-certificates
