# How to bind Public IPv4 of an Ubuntu server to a domain

**Note:** The related ports of ubuntu server must be open for outbound access. (e.g.: If you're using a EC2 instance, you should edit your security group to manage access that comes from outbound/inbound.)

## Install nginx with above commands

[`Up to date source`](https://nginx.org/en/linux_packages.html#Ubuntu)

Install the prerequisites:

```sh
$ sudo apt install curl gnupg2 ca-certificates lsb-release ubuntu-keyring
```

Import an official nginx signing key so apt could verify the packages authenticity. Fetch the key:

```sh
$ curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
    | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null
```
 
Verify that the downloaded file contains the proper key:

```sh
$ gpg --dry-run --quiet --no-keyring --import --import-options import-show /usr/share/keyrings/nginx-archive-keyring.gpg
```

The output should contain the full fingerprint 573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62 as follows:

```sh
pub   rsa2048 2011-08-19 [SC] [expires: 2024-06-14]
      573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62
uid                      nginx signing key <signing-key@nginx.com>
```

If the fingerprint is different, remove the file.
To set up the apt repository for stable nginx packages, run the following command:

```sh
$ echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
http://nginx.org/packages/ubuntu `lsb_release -cs` nginx" \
    | sudo tee /etc/apt/sources.list.d/nginx.list
```

If you would like to use mainline nginx packages, run the following command instead:

```sh
$ echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
http://nginx.org/packages/mainline/ubuntu `lsb_release -cs` nginx" \
    | sudo tee /etc/apt/sources.list.d/nginx.list
```
 
Set up repository pinning to prefer our packages over distribution-provided ones:

```sh
$ echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" \
    | sudo tee /etc/apt/preferences.d/99nginx
```
 
To install nginx, run the following commands:

```sh
$ sudo apt update
$ sudo apt install nginx
```

<hr>

## Nginx Configuration

Will be added to `/etc/nginx/nginx.conf` after the line of `include /etc/nginx/conf.d/*.conf;`:

```conf
...
include /etc/nginx/sites-enabled/*;
...
```

Create a file with full domain (e.g. api.example.com) in `/etc/nginx/sites-available/`, paste this and save it.

```conf
server {
    server_name api.example.com;

 location / {
  proxy_pass http://localhost:3333; # your api url that's running on server
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "Upgrade";
  proxy_set_header Host $host;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_connect_timeout 150;
  proxy_send_timeout 100;
  proxy_read_timeout 100;
  proxy_buffers 4 32k;
  client_max_body_size 8m;
  client_body_buffer_size 128k;
    }
}
```

Then run below command to create a symlink to point `sites-available/api.example.com` file from `/sites-enabled` folder:

```sh
$ ln -s /etc/nginx/sites-available/api.example.com /etc/nginx/sites-enabled/api.example.com
```

And check configs with below command:

```sh
$ sudo nginx -t
```

<hr>

## Install Certbot

Add the Certbot PPA to list of repositories and install Certbot with the commands below:

```sh
$ sudo apt-get install software-properties-common
$ sudo add-apt-repository universe
$ sudo add-apt-repository ppa:certbot/certbot
$ sudo apt-get update
$ sudo apt-get install certbot python3-certbot-nginx
```

Then if nginx configuration check is success, run the below command for the domain you specified to create free ssl certificate:

```sh
$ sudo certbot --nginx
```

## Arrangements on Domain DNS

Create a record:

```
Type: A
IPv4 Address: your-server-public-ip
TTL: 14400 (default)
Name: api.example.com
```
