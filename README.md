# Ghost with Let's Encrypt Using Docker Compose

## New notes

Original notes are in the following section. This section is after my own tweaks.

Main tweaks include using docker hardened images. you would need to get a dhi account to download and use the images. Then, these images may have stricter behaviour than the original images.

The hardened mysql (dhi.io/mysql:8.4) and traefik (dhi.io/traefik:3) images, defined in .env require that the mounted volumes have the correct permissions.
The user and groups defined for these images are available on dhi e.g. [here](https://hub.docker.com/hardened-images/catalog/dhi/traefik/images)

### mysql
When you try to run `docker compose up -d` you might come across ghost container having access errors to the db. This is because the mysql image requires that the mounted volume has the correct permissions. 
- You _might_ need to install mysql on your host machine, I had it so I'm assuming this.
- You would need to add the correct ownership to the `/var/lib/mysql` directory; typically this is already the `mysql` user, and the group id is `133`.
- You would need to add the `group_add` to the mysql service in the `docker-compose.yml` file, with the correct group_id. Check `getent group mysql` for your `mysql` user's group id, mine was 133.

- You would need to create the database and the user with the password manually. 
```
$ docker exec -it ghost-traefik-letsencrypt-docker-compose-mysql-1  mysql -u root -p
$ mysql > CREATE DATABASE ghostdb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
$ mysql > CREATE USER 'ghost'@'%' IDENTIFIED BY 'your_password';
$ mysql > GRANT ALL PRIVILEGES ON ghostdb.* TO 'ghost'@'%';
$ mysql > FLUSH PRIVILEGES;
$ exit;
```
- even with the above considerations, you might have some issues. I came across ghost failing with some migration error. You might need to manually unlock the migration, and then restart ghost.
```
$ mysql > UPDATE migrations_lock SET locked=0 WHERE lock_key='km01'; 
```

### traefik
The traefik image requires that the volume to be mounted, `/etc/traefik/acme/acme.json` to EXIST on your host, and have the correct permissions.
- note that now the mounted volume on the traefik service volume is not a named volume, but the host directory.
- You would need to create the `acme.json` file with the correct permissions. 
```
$ mkdir -p /etc/traefik/acme
$ touch /etc/traefik/acme/acme.json
$ chmod 600 /etc/traefik/acme/acme.json
$ chown 65532:65532 /etc/traefik/acme/acme.json
```
- note that 65532 is for "nobody" or `nonroot`. You can find the group id by running `getent group docker` on your host, and then you'd add the group id to the `group_add` in the traefik service in the `docker-compose.yml` file.
- pretty sure the ping entrypoint has to be `traefik` instead of `ping`.
- healthcheck has to be changed to use traefik instead of ping too.

## old notes
❗ Change variables in the `.env` to meet your requirements.

💡 Note that the `.env` file should be in the same directory as `ghost-traefik-letsencrypt-docker-compose.yml`.

Create networks for your services before deploying the configuration using the commands:

`docker network create traefik-network`

`docker network create ghost-network`

Deploy Ghost using Docker Compose:

`docker compose -f ghost-traefik-letsencrypt-docker-compose.yml -p ghost up -d`

## Backups

The `backups` container in the configuration is responsible for the following:

1. **Database Backup**: Creates compressed backups of the MariaDB database using pg_dump.
Customizable backup path, filename pattern, and schedule through variables like `MARIADB_BACKUPS_PATH`, `MARIADB_BACKUP_NAME`, and `BACKUP_INTERVAL`.

2. **Application Data Backup**: Compresses and stores backups of the application data on the same schedule. Controlled via variables such as `DATA_BACKUPS_PATH`, `DATA_BACKUP_NAME`, and `BACKUP_INTERVAL`.

3. **Backup Pruning**: Periodically removes backups exceeding a specified age to manage storage. Customizable pruning schedule and age threshold with `MARIADB_BACKUP_PRUNE_DAYS` and `DATA_BACKUP_PRUNE_DAYS`.

By utilizing this container, consistent and automated backups of the essential components of your instance are ensured. Moreover, efficient management of backup storage and tailored backup routines can be achieved through easy and flexible configuration using environment variables.

## ghost-restore-database.sh Description

This script facilitates the restoration of a database backup:

1. **Identify Containers**: It first identifies the service and backups containers by name, finding the appropriate container IDs.

2. **List Backups**: Displays all available database backups located at the specified backup path.

3. **Select Backup**: Prompts the user to copy and paste the desired backup name from the list to restore the database.

4. **Stop Service**: Temporarily stops the service to ensure data consistency during restoration.

5. **Restore Database**: Executes a sequence of commands to drop the current database, create a new one, and restore it from the selected compressed backup file.

6. **Start Service**: Restarts the service after the restoration is completed.

To make the `ghost-restore-database.shh` script executable, run the following command:

`chmod +x ghost-restore-database.sh`

Usage of this script ensures a controlled and guided process to restore the database from an existing backup.

## ghost-restore-application-data.sh Description

This script is designed to restore the application data:

1. **Identify Containers**: Similarly to the database restore script, it identifies the service and backups containers by name.

2. **List Application Data Backups**: Displays all available application data backups at the specified backup path.

3. **Select Backup**: Asks the user to copy and paste the desired backup name for application data restoration.

4. **Stop Service**: Stops the service to prevent any conflicts during the restore process.

5. **Restore Application Data**: Removes the current application data and then extracts the selected backup to the appropriate application data path.

6. **Start Service**: Restarts the service after the application data has been successfully restored.

To make the `ghost-restore-application-data.sh` script executable, run the following command:

`chmod +x ghost-restore-application-data.sh`

By utilizing this script, you can efficiently restore application data from an existing backup while ensuring proper coordination with the running service.


## Credits go to [Vladimir Mikhalev](https://github.com/heyvaldemar), I forked his repo and modded it to my needs.

🎬 [YouTube](https://www.youtube.com/channel/UCf85kQ0u1sYTTTyKVpxrlyQ?sub_confirmation=1)\
🐦 [X / Twitter](https://twitter.com/heyvaldemar)\
🎨 [Instagram](https://www.instagram.com/heyvaldemar/)\
🐘 [Mastodon](https://mastodon.social/@heyvaldemar)\
🧵 [Threads](https://www.threads.net/@heyvaldemar)\
🎸 [Facebook](https://www.facebook.com/heyvaldemarFB/)\
🧊 [Bluesky](https://bsky.app/profile/heyvaldemar.bsky.social)\
🎥 [TikTok](https://www.tiktok.com/@heyvaldemar)\
💻 [LinkedIn](https://www.linkedin.com/in/heyvaldemar/)\
📣 [daily.dev Squad](https://app.daily.dev/squads/devopscompass)\
🧩 [LeetCode](https://leetcode.com/u/heyvaldemar/)\
🐈 [GitHub](https://github.com/heyvaldemar)

