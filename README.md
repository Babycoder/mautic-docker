# Mautic Migration and Dockerization Documentation

<img src="https://opensourcegeeks.net/content/images/2022/11/How-To-Install-Mautic-With-Docker---Opensource-Geeks.png" width="500" height="250" />

## Overview

This document outlines the step-by-step process to export the database from the live Mautic instance, set up a Dockerized version of Mautic, and migrate the database for testing and future deployment.

## Objectives

- Export the database from the live Mautic instance.
- Set up a Dockerized Mautic environment locally.
- Import the exported database into the new Dockerized instance.
- Test and validate the migration.

---

## 1. Export Database from Live Instance

1. **Dump the Database and transfer it to Local Machine:**
   - Run the following command to export the database:
     ```bash
     mysqldump -u <DB_USER> -p'<DB_PASSWORD>' --host=<DB_HOST> <DB_NAME> > mautic_backup.sql
     ```
2. **Export the necessary media files**


---

## 2. Set Up Dockerized Mautic

### Steps:

1. **Clone Mautic Docker Repository:**

   ```bash
   git clone git@github.com:Babycoder/mautic-docker.git
   cd docker-mautic
   ```

2. **Modify ****************************************************************************************************************************`docker-compose.yml`****************************************************************************************************************************:**

   - This file defines how Mautic and its dependencies run as containers.
   - **Breakdown of Services:**
     - `db`: Runs MySQL 8.0, storing the Mautic database.
     - `mautic_web`: Runs the latest Mautic web application, linked to the database.
     - `mautic_cron`: Handles Mautic's scheduled tasks.
     - `mautic_worker`: Runs background tasks.
   - **Environment Variables:**
     - `${MYSQL_ROOT_PASSWORD}`, `${MYSQL_DATABASE}`, `${MYSQL_USER}`, `${MYSQL_PASSWORD}`: Database credentials.
   - **Volumes:**
     - Volumes ensure that Mautic's configuration, logs, cache, media files, and cron jobs persist across container restarts.
     - Mounted directories include:
       - `./mautic/config:/var/www/html/config` → Stores configuration files (local.php).
       - `./mautic/logs:/var/www/html/var/logs` → Stores logs for debugging.
       - `./mautic/cache:/var/www/html/var/cache` → Stores cached files to improve performance.
       - `./mautic/media/files:/var/www/html/docroot/media/files` → Stores uploaded media files.
       - `./mautic/media/images:/var/www/html/docroot/media/images` → Stores image assets.
       - `./cron:/opt/mautic/cron` → Handles scheduled tasks for automation.
   - **Health Checks & Dependencies:**
     - The services depend on each other (`depends_on` ensures the database is ready before launching Mautic).

3. **Start the Docker Containers:**

   ```bash
   docker-compose up -d
   ```

4. **Check Running Containers:**

   ```bash
   docker ps
   ```

---

## 3. Import Database into Dockerized Mautic

### Steps:

1. **Copy the Database and Access the Mautic Database Container:**

   ```bash
   docker cp /path/to/local/mautic_backup.sql <MAUTIC_DB_CONTAINER>:/path/in/container
   docker exec -it <MAUTIC_DB_CONTAINER> bash
   ```

2. **Mautic 5 uses Symfony and Doctrine updates, which may have changed the way JSON fields are handled. If Mautic previously used json_array and the new version expects json, this fix ensures compatibility.**

   ```bash
   sed -i 's/\bjson_array\b/json/g' mautic_backup.sql
   ```

3. **Import the Backup:**

   ```bash
   mysql -u <DB_USER> -p <DB_NAME> < /path/to/mautic_backup.sql
   ```
4. **Import media files:**
```bash
docker cp /path/to/local/files <MAUTIC_DB_CONTAINER>:/var/www/html/docroot/media/files
docker cp /path/to/local/images <MAUTIC_DB_CONTAINER>:/var/www/html/docroot/media/images

```
   

---



## 5. Running migration


1.  **Run the migration command and !! check the logs**
 ```
 docker compose exec <MAUTIC_WEB_CONTAINER> php ../bin/console doctrine:migrations:migrate --no-interaction --dry-run
 ```
2. **clear the cache** 
 ```
 docker compose exec <MAUTIC_WEB_CONTAINER> php ../bin/console cache:clear
 ```


---

## 4. Testing & Validation

### Steps:

1. **Verify Database Integrity:**

   - Run SQL queries to check if all tables and data exist.

2. **Check Logs:**

   ```bash
   mautic/logs
   ```

3. **Access Mautic in the Browser:**

   - Navigate to `http://localhost:8001`
   - Login with admin credentials.
   - Ensure the data is correctly migrated.

---

## 5. Notes

- Set proper permission < MAUTIC_WEB_CONTAINER >
```
chown -R www-data:www-data /var/www/html/
find /var/www/html/ -type d -exec chmod 755 {} \;
find /var/www/html/ -type f -exec chmod 644 {} \;
```
- Ensure all integrations and configurations are restored.
