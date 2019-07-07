---
title: "‘Out of the Box’ HTTPS on GCP with nginx and Let’s Encrypt"
date: 2019-07-07T13:35:12+01:00
draft: false
tags: ["GCP"]
---

Often, while experimenting with Compute Engine on GCP, we need to set up a webserver.

Ideally, we'd want to create a VM which starts "out of the box" with HTTPS
pre-configured, as that's how we'll test most of our applications anyway.
Spending time configuring HTTPS again and again doesn't sound like fun!

The following is a script to set up **nginx** and **Let's Encrypt** automatically. It can
be used as a startup script. It's designed for Centos 7 but the script is easily adapted
to Debian-based or other OSes.

If you use any other webserver, you'll have to do some more work to configure Let's Encrypt --
here's a [Java example](https://danielflower.github.io/2017/04/08/Lets-Encrypt-Certs-with-embedded-Jetty.html).

You can type the startup script inline, or put it in a file accessible to the `gcloud`
command, or store the script in a Cloud Storage bucket.
[This page](https://cloud.google.com/compute/docs/startupscript) has more information,
but the key options are:

> * `--metadata startup-script=CONTENTS`: Supply the startup script contents directly with this key.
> * `--metadata startup-script-url=URL`: Supply a Google Cloud Storage URL to the start script file with this key.
> * `--metadata-from-file startup-script=FILE`: Supply a locally stored start up script file (you can keep small files in Cloud Shell, although obviously this doesn't scale well).

Below, we've used `--metadata-from-file` for  simplicity.

This is `nginx-tls.sh`, save it where your `gcloud` command can find it, e.g.
on the Cloud Shell filesystem.

```bash
sudo yum -y update
sudo yum -y install epel-release certbot-nginx nginx
sudo systemctl start nginx
# you must have a domain name you own and have access to DNS management for that domain
sudo sed -i 's/server_name  _/server_name  SUBDOMAIN.YOURDOMAIN.COM/g' /etc/nginx/nginx.conf
sudo nginx -t
sudo systemctl reload nginx
sudo certbot register --email YOUR@EMAIL.COM --no-eff-email --agree-tos
sudo certbot --nginx -d SUBDOMAIN.YOURDOMAIN.COM --redirect
sudo systemctl enable nginx
# the schedule below will run every month -- change as needed
(sudo crontab -l 2>/dev/null; echo "14 3 5 */1 * /usr/bin/certbot renew --quiet") | sudo crontab -
```

...and create your Compute Engine VM thus:

```bash
gcloud compute --project=YOUR_PROJECT instances create \
    YOUR_MACHINE_NAME \
    --zone=YOUR_REGION --machine-type=YOUR_MACHINE_TYPE \
    --subnet=YOUR_VPC_NAME \
    --address=RESERVED_IP_ADDRESS \
    --network-tier=PREMIUM --maintenance-policy=MIGRATE \
    --service-account=... \
    --scopes=... \
    --tags=webserver --image=centos-7-v20190619 --image-project=centos-cloud \
    --boot-disk-size=30GB --boot-disk-type=pd-standard \
    --boot-disk-device-name=YOUR_MACHINE_NAME \
    --metadata-from-file startup-script=./nginx-tls.sh
```

Note that:

* The IP address specified in `--address=` **must** be a reserved IP address.
* The firewall rules for this machine must be configured to allow ingress
  for `tcp:80,443`. In this example, this rule is applied on the `webserver`
  target tag.
