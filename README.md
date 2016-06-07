# Scaling your Drupal (Or any PHP) application on the Google Compute Cloud with Docker.

When running a Drupal platform hosting is essential for the succes of your website. Being able to scale with spikes, increasing traffic and storage can take alot of time and is often expensive. The Google Compute Cloud (http://cloud.google.com) can help out while still being in control as a developer. In this article we will focus on deploying a scalable and reliable Drupal platform.

To start out we need to signup for the Google Cloud and create a project.
- Visit https://console.cloud.google.com and create your project. If you want to make use of the App Engine like the Task Queue be sure to check whether the App Engine location is in the desired region under 'Advanced Options'.
- Install the gcloud console tool for your system (https://cloud.google.com/sdk/downloads).
- Run `gcloud init` to setup your access to the Google Cloud project.

If you already have a Google Cloud account and project initialize it with the commands:
```
gcloud projects list
gcloud config set project [PROJECT_ID]
```

The Google Cloud allows you to choose the zones you want your application to run. It is important to think about where you want your platform to run. In this article we will focus on a single region. The available zones and regions can be found using the command `gcloud compute zones list`. More information about these zones can be found here https://cloud.google.com/compute/docs/regions-zones/regions-zones. After determining the zone make sure this is used as default:
```
gcloud config set compute/zone [ZONE]
```

## Setting up your MySQL database using Google CloudSQL (2nd generation).
1. First you should check which machine type you want to use. The second generation SQL uses the compute engine for deploying the database. You can get a list of supported machine types using `gcloud sql tiers list` or find more information at https://cloud.google.com/sql/docs/admin-api/v1beta4/tiers#second and https://cloud.google.com/sql/pricing#v2-instance-pricing.
In this article we start with the default db-n1-standard-1 which is upgradable to a more performant machine later. If you are just running a small blog or mostly static website you can even choose to use the db-f1-micro machine type. 

2. To be able to access the database from your network determine your IP address.

Create your SQL Instance:
```
gcloud sql instances create [CLOUDSQL_INSTANCE_NAME[ --tier=[TIER] --activation-policy=ALWAYS --authorized-networks=[IP] --region [REGION] --gce-zone [ZONE] --backup --backup-time [HH:MM] --enable-bin-log --database-flags innodb_file_per_table=1
```

Set the root password:
```
gcloud sql instances set-root-password [CLOUDSQL_INSTANCE_NAME] --password [PASSWORD]
```

If availability is important you might want to make use of the High Availability configuration which will deploy a master in the preferred zone and a failover in a different zone closeby. When a zonal outage occurs and your master fails over to your failover replica, any existing connections to the instance are closed. However, your application can reconnect using the same connection string or IP address; you do not need to update your application after a failover. More information can be found here https://cloud.google.com/sql/docs/high-availability. A replica can be enabled when backups are active, binlog is active and at least one backup has been created. Create the failover the next day:
```
gcloud sql instances create [CLOUDSQL_INSTANCE_NAME]-repli --master-instance-name [CLOUDSQL_INSTANCE_NAME]
```

Find the IP address of your running db instance:
```
gcloud sql instances list
```

Test whether you can connect to your MySQL:
```
mysql -h [CLOUDSQL_INSTANCE_IP] -u root -p
```

After successfully connecting to your MySQL create a user and database to allow the CloudSQL Proxy to access your database.
```
mysql> CREATE USER [CLOUDSQL_USER]@'cloudsqlproxy~%';
mysql> CREATE DATABASE [CLOUDSQL_DB];
mysql> GRANT ALL PRIVILEGES ON [CLOUDSQL_DB].* TO '[CLOUDSQL_USER]'@'cloudsqlproxy~%';
```
