---
layout: post
title:  My PostGIS Playground
date:   2017-03-25
tags: [postgis, docker, postgresql]
excerpt: Need an easy, cheap, and powerful way to play with PostGIS?
---

Need an easy, cheap, and powerful way to play with PostGIS? In the past few months I've settled into a cloud-based workflow that allows me both to inexpensively tinker with PostGIS and to run massive queries without tying up my local box. Let me tell you how to set it all up.

But first, why not run it locally?

1. I prefer to have PostgreSQL+PostGIS virtualized or in a container so that I can easily destroy/rebuild/switch versions.
2. VBoxsf makes VirtualBox, and therefore Docker Machine, almost worthless for a database on a Mac (speed and permissions). Last I checked, Docker for Mac doesn't do any better.
3. Running PostGIS in the cloud lets you use the same setup whether you need 2GB of RAM or 200GB.

## Prepping your server image
If you don't already have an account at <a href="https://www.linode.com/" target="_blank">Linode</a>, go make one. In my personal tests, Linode handedly beats out Rackspace and Digital Ocean in performance, while being considerably less expensive. For example, a 4-core, 8GB instance at Digital Ocean is $80/mo, but the same thing at Linode is $40/mo, and the Linode is faster. Others have made <a href="https://forum.linode.com/viewtopic.php?t=11863&p=67121" target="_blank">similar comparisons to AWS</a>.

Now spin up an instance. I generally like 8GB of RAM or more, but if you're just tinkering you can do whatever you want. Deploy the Debian 8 image to your newly-created Linode. Boot your server, wait a couple minutes, then log in via SSH.

My minimum package list is:
* **Docker** for running Postgres,
* **GDAL** for `ogr2ogr` which helps us move data in and out of PostGIS,
* **tmux** for, well, being tmux,
* **awscli** for saving my work in S3 when I destroy the server,
* **pigz** for rapidly compressing my data before uploading to S3.

To install, run the following:

```bash
echo 'Acquire::ForceIPv4 "true";' | sudo tee /etc/apt/apt.conf.d/99force-ipv4;
apt-get update && apt-get install -y apt-transport-https ca-certificates;
apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net --recv-keys 58118E89F3A912897C070ADBF76221572C52609D;
echo "deb https://apt.dockerproject.org/repo debian-jessie main" | sudo tee /etc/apt/sources.list.d/docker.list;
apt-get update && apt-get install -y docker-engine gdal-bin tmux awscli pigz;
apt-get clean;
```

Now we need to create an image. Linode's UI and imaging isn't the best, so follow along closely.

1. If you're not already on the "Dashboard" for your server, click on your server's name to get there.
2. Under "Disks", click on the name of your main Disk, probably "Debian 8 Disk".
3. Hit "Create Image". Name it what you want, and hit "Create Image" (again).
4. **If your image size is limited to 2GB (default), what you've just done will fail.** Open a ticket with Linode asking them to increase your allowed image size. When they get back to you, repeat 1-3. Sorry. :/

That's it! You now have an image you can quickly deploy all ready to run PostGIS (or anything in Docker, for that matter). Next time you create a Linode, deploy your new image instead of Debian 8.

## Starting PostGIS
Let's start Postgres+PostGIS, using the `mdillon/postgis` image:

```bash
docker run -d --name pg -e POSTGRES_PASSWORD=nothing -v ~/96:/var/lib/postgresql/data -p 5432:5432 mdillon/postgis:9.6
```

I like to call the container "pg" (`--name pg`), and put its data in the folder "96" since sometimes I'll switch versions (`-v ~/96:/var...`). Its port is opened on localhost (`-p 5432:5432`), so tools like `ogr2ogr` can act like it's local and I can connect directly via Navicat or similar from my local box.

To get the most out of Postgres, we should tweak its config. Open up `~/96/postgresql.conf` in vim (or your favorite editor), and change the following (I'm assuming 8GB of RAM–adjust accordingly):

* shared_buffers 3GB
* work_mem 64MB
* maintenance_work_mem 256MB
* effective_cache_size 7GB
* max_parallel_workers_per_gather 4
* effective_io_concurrency 100
* random_page_cost 1.1

Once the config is saved, we can simply destroy the container and start it again with the same command:

```bash
docker stop pg
docker rm pg
docker run -d --name pg -e POSTGRES_PASSWORD=nothing -v ~/96:/var/lib/postgresql/data -p 5432:5432 mdillon/postgis:9.6
```

PostgreSQL will be restarted with the new settings.

## Do something fun!
Obviously we want to load some cartography into PostGIS and run queries, so let's download and import last month's (2/17) precipitation data for the whole US:

```bash
wget http://water.weather.gov/precip/p_download_new/2017/03/01/nws_precip_month2date_observed_shape_20170301.tar.gz
tar -xzf nws_precip_month2date_observed_shape_20170301.tar.gz

ogr2ogr -nln precipitation --config PG_USE_COPY YES -f PostgreSQL PG:'dbname=postgres host=127.0.0.1 user=postgres password=nothing' nws_precip_month2date_observed_20170301.shp;
```

For more explanation of `ogr2ogr`, have a look at the <a href="http://www.gdal.org/ogr2ogr.html" target="_blank">docs</a> or Boston GIS's <a href="http://www.bostongis.com/PrinterFriendly.aspx?content_name=ogr_cheatsheet" target="_blank">excellent cheatsheet</a>.

To run queries, start `psql` inside the Docker container:

```bash
docker exec -it pg psql -U postgres
```

Let's find the average rainful within 200 miles of Dallas:
```sql
SELECT AVG(p.globvalue) AS inches
FROM precipitation AS p
WHERE ST_DWithin(
  ST_Transform(ST_GeomFromText('POINT(-96.7970 32.7767)', 4326), 32614),
  ST_Transform(p.wkb_geometry, 32614),
  200 * 1609.34
);

       inches       
--------------------
 2.6440897862232779
(1 row)
```

There ya go!

## Saving your work
This workflow wouldn't be complete if you had to start from scratch every time. I like to upload my database to S3 before deleting the server. Assuming you've already run `aws configure`, do something like this:


```bash
# Stop Postgres so it won't be writing to disk.
docker stop pg
docker rm pg

# Archive the database folder.
# Pipe it to pigz for fast compression.
tar -cf - 96 | pigz - > 96.tar.gz

# Upload, keeping it cheap.
aws s3 cp 96.tar.gz s3://my-bucket --storage-class REDUCED_REDUNDANCY
```

Next time you start up a server, just pull down your archive and pick up where you left off:

```bash
aws s3 cp s3://my-bucket/96.tar.gz .
tar -xzf 96.tar.gz
```
