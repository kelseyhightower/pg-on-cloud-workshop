# Postgres On Cloud Workshop

This is a simple tutorial on running Postgres on GCP. It's a collection of notes from now, and I might improve it in the future.

## Create Two GCP Projects

```
gcloud projects create --name development
```

> Output
```
No project id provided.

Use [development-312912] as project id (Y/n)?  y

Create in progress for [https://cloudresourcemanager.googleapis.com/v1/projects/development-312912].
Waiting for [operations/cp.9195103853985747792] to finish...done.                                                  
Enabling service [cloudapis.googleapis.com] on project [development-312912]...
Operation "operations/acf.p2-1060547558691-278170c5-cb09-4cd7-84cd-5d8fd5cb82b3" finished successfully.
```

```
gcloud projects create --name production-databases
```

```
No project id provided.

Use [production-databases-312912] as project id (Y/n)?  y

Create in progress for [https://cloudresourcemanager.googleapis.com/v1/projects/production-databases-312912].
Waiting for [operations/cp.5591427643466269802] to finish...done.                                                  
Enabling service [cloudapis.googleapis.com] on project [production-databases-312912]...
Operation "operations/acf.p2-417948797520-673bfc9c-f47c-4dc8-ad32-6443e6fa439c" finished successfully.
```

Get the `development` and `production-databases` project IDs:


```
DEVELOPMENT_PROJECT_ID=$(gcloud projects list \
  --filter="name:development" \
  --format "value(projectId)")
```

```
PRODUCTION_DATABASES_PROJECT_ID=$(gcloud projects list \
  --filter="name:production-databases" \
  --format "value(projectId)")
```


## Create Cloud SQL Database

```
gcloud services enable sqladmin.googleapis.com \
  --project ${PRODUCTION_DATABASES_PROJECT_ID}
```

Create the `production-db` database in the `production-databases` project:

```
gcloud sql instances create production-db \
 --database-version POSTGRES_13 \
 --cpu 2 \
 --memory 3840MB \
 --region us-west1 \
 --project ${PRODUCTION_DATABASES_PROJECT_ID}
```

> Output
```
Creating Cloud SQL instance...done.                                                                                
Created [https://sqladmin.googleapis.com/sql/v1beta4/projects/production-databases-312912/instances/production-db].
NAME           DATABASE_VERSION  LOCATION    TIER              PRIMARY_ADDRESS  PRIVATE_ADDRESS  STATUS
production-db  POSTGRES_13       us-west1-c  db-custom-2-3840  XX.XX.XX.X       -                RUNNABLE
```

Set the postgres user password:

```
gcloud sql users set-password postgres \
  --instance production-db \
  --project ${PRODUCTION_DATABASES_PROJECT_ID} \
  --prompt-for-password
```
> Output
```
Instance Password: 
Updating Cloud SQL user...done.  
```

```
sudo apt install postgresql-client
```

Create a client certificate using the ssl client-certs create command:

```
gcloud sql ssl client-certs create client client-key.pem \
  --instance=production-db \
  --project ${PRODUCTION_DATABASES_PROJECT_ID}
```

Retrieve the public key for the certificate you just created and copy it into the client-cert.pem file with the ssl client-certs describe command:

```
gcloud sql ssl client-certs describe client \
  --instance=production-db \
  --project ${PRODUCTION_DATABASES_PROJECT_ID} \
  --format="value(cert)" > client-cert.pem
```

Copy the server certificate into the server-ca.pem file using the instances describe command:

```
gcloud beta sql ssl server-ca-certs list \
  --format="value(cert)" \
  --instance=production-db \
  --project ${PRODUCTION_DATABASES_PROJECT_ID} > server-ca.pem
```

## Connect to your instance using SSL

```
HOST_ADDR=$(gcloud sql instances describe production-db \
  --project ${PRODUCTION_DATABASES_PROJECT_ID} \
  --format="value(ipAddresses[0].ipAddress)")
```

```
psql "sslmode=verify-ca sslrootcert=server-ca.pem \
      sslcert=client-cert.pem sslkey=client-key.pem \
      hostaddr=${HOST_ADDR} \
      user=postgres dbname=postgres"
```

> Let me know if you get stuck and it does not connect.

## Create a new database

List the current set of databases using the `databases list` command:

```
gcloud sql databases list \
  --instance=production-db \
  --project ${PRODUCTION_DATABASES_PROJECT_ID}
```

> Output
```
NAME      CHARSET  COLLATION
postgres  UTF8     en_US.UTF8
```

List the current set of databases using `psql`:

```
psql "sslmode=verify-ca sslrootcert=server-ca.pem \
      sslcert=client-cert.pem sslkey=client-key.pem \
      hostaddr=${HOST_ADDR} \
      user=postgres dbname=postgres" -c '\l'
```
> Output

```
                                                List of databases
     Name      |       Owner       | Encoding |  Collate   |   Ctype    |            Access privileges            
---------------+-------------------+----------+------------+------------+-----------------------------------------
 cloudsqladmin | cloudsqladmin     | UTF8     | en_US.UTF8 | en_US.UTF8 | 
 postgres      | cloudsqlsuperuser | UTF8     | en_US.UTF8 | en_US.UTF8 | 
 template0     | cloudsqladmin     | UTF8     | en_US.UTF8 | en_US.UTF8 | =c/cloudsqladmin                       +
               |                   |          |            |            | cloudsqladmin=CTc/cloudsqladmin
 template1     | cloudsqlsuperuser | UTF8     | en_US.UTF8 | en_US.UTF8 | =c/cloudsqlsuperuser                   +
               |                   |          |            |            | cloudsqlsuperuser=CTc/cloudsqlsuperuser
(4 rows)
``` 

Create a new database in the production instance:

```
gcloud sql databases create web \
  --instance=production-db \
  --project ${PRODUCTION_DATABASES_PROJECT_ID}
```
> Output

```
Creating Cloud SQL database...done.                                                                                
Created database [web].
instance: production-db
name: web
project: production-databases-312912
```

List the current set of databases using the `databases list` command:

```
gcloud sql databases list \
  --instance=production-db \
  --project ${PRODUCTION_DATABASES_PROJECT_ID}
```

> Output

```
NAME      CHARSET  COLLATION
postgres  UTF8     en_US.UTF8
web       UTF8     en_US.UTF8
```

List the current set of databases using `psql`:

```
psql "sslmode=verify-ca sslrootcert=server-ca.pem \
      sslcert=client-cert.pem sslkey=client-key.pem \
      hostaddr=${HOST_ADDR} \
      user=postgres dbname=postgres" -c '\l'
```

## Create a new database user

```
psql "sslmode=verify-ca sslrootcert=server-ca.pem \
      sslcert=client-cert.pem sslkey=client-key.pem \
      hostaddr=${HOST_ADDR} \
      user=postgres dbname=postgres"
```

```
postgres=> create database frontend;
```

```
postgres=> create user frontend with encrypted password 'frontend123';
```

```
postgres=> grant all privileges on database frontend to frontend;
```

List the current set of databases using the `databases list` command:

```
gcloud sql databases list \
  --instance=production-db \
  --project ${PRODUCTION_DATABASES_PROJECT_ID}
```

> Output

```
NAME      CHARSET  COLLATION
postgres  UTF8     en_US.UTF8
web       UTF8     en_US.UTF8
frontend  UTF8     en_US.UTF8
```

Create the `frontend` client certificate using the ssl client-certs create command:

```
gcloud sql ssl client-certs create frontend frontend-key.pem \
  --instance=production-db \
  --project ${PRODUCTION_DATABASES_PROJECT_ID}
```

Retrieve the public key for the `frontend` certificate you just created and copy it into the frontend-cert.pem file with the ssl client-certs describe command:

```
gcloud sql ssl client-certs describe frontend \
  --instance=production-db \
  --project ${PRODUCTION_DATABASES_PROJECT_ID} \
  --format="value(cert)" > frontend-cert.pem
```

Connect to the `frontend` database on the `production` instance:

```
psql "sslmode=verify-ca sslrootcert=server-ca.pem \
      sslcert=frontend-cert.pem sslkey=frontend-key.pem \
      hostaddr=${HOST_ADDR} \
      user=frontend dbname=frontend"
```

> Output

```
Password for user frontend: 
psql (11.11 (Debian 11.11-0+deb10u1), server 13.2)
WARNING: psql major version 11, server major version 13.
         Some psql features might not work.
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

frontend=> 
```

## Deploy Postgresql on GCE

### Hints

Create a GCE VM:

```
gcloud compute instances create postgres \
  --image-family ubuntu-2004-lts \
  --image-project ubuntu-os-cloud \
  --machine-type e2-standard-2 \
  --project ${PRODUCTION_DATABASES_PROJECT_ID} \
  --zone us-west1-a
```

> Output

```
API [compute.googleapis.com] not enabled on project [417948797520]. 
Would you like to enable and retry (this will take a few minutes)? 
(y/N)?  y

Enabling service [compute.googleapis.com] on project [417948797520]...
Operation "operations/acf.p2-417948797520-fa6f9a3b-170e-4d97-8324-d8236f94a9b1" finished successfully.
Created [https://www.googleapis.com/compute/v1/projects/production-databases-312912/zones/us-west1-a/instances/postgres].
NAME      ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP      STATUS
postgres  us-west1-a  e2-standard-2               XX.XXX.X.X   XXX.XXX.XXX.XXX  RUNNING
```

SSH into the postgres VM:

```
gcloud compute ssh postgres \
  --zone us-west1-a \
  --project ${PRODUCTION_DATABASES_PROJECT_ID}
```

## Clean Up

```
gcloud sql instances delete production-db \
  --project ${PRODUCTION_DATABASES_PROJECT_ID}
```

## Terraform


