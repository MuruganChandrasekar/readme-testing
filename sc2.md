[Setup Instructions](#setup-instructions)

* [1 Cloud-Provider Setup](#cloud-provider-setup)

  +  [1.1 On Digital Ocean](#on-digital-ocean)

     + [Create droplet](#create-droplet)

     + [Configure droplet](#configure-droplet)

* [2 After server is set up](#after-server-is-set-up)

  + [2.1 Create and edit the .env file](#create-and-edit-the.envfile)

  + [2.2 Set up the Node-RED and InfluxDB API logins](#set-up-the-node-red-and-influxdb-api-logins)

* [3 Start the server](#start-the-server)

  + [3.1 Restart servers in the background](#restart-servers-in-the-background)

  + [3.2 Initial testing](#initial-testing)

  + [3.3 Set up first data source](#set-up-first-data-source)

  + [3.4 Test Node-RED](#test-node-red)

  + [3.5 Creating an InfluxDB database](#creating-an-influxdb-database)

  + [3.6 Add Nginx log in for NodeRed or query after the fact](#add-nginx-log-in-for-nodered-or-query-after-the-fact)

  + [3.7 MQTT User Credentials setup](#mqtt-user-credentials-setup)
  

Setup Instructions
==================

**Notes:**

For example, if the dashboard server name: `dashboard.example.com`

Other things are to be named consistently:

-   `/opt/docker/dashboard.example.com` is the directory (on the host system) containing the docker files.

-   `/var/opt/docker/dashboard.example.com` is the directory (on the host system) containing persistent data.

-   Node-RED familiarity is assumed.

Cloud-Provider Setup
-----------------------

As an initial step, a cloud provider is required and Docker and Docker-Compose must be installed which is provider dependent.

### On Digital Ocean
--------------------

#### Create droplet

1.  Log in to [Digital Ocean](https://cloud.digitalocean.com/)

2.  Create a new project (if needed) to hold the new droplet.

3.  Discover \> Marketplace, search for `Docker`

4.  This page will be redirected:
    <https://cloud.digitalocean.com/marketplace/5ba19751fc53b8179c7a0071?i=ec3581>

5.  Press "Create"

6.  Select the standard 8G GB Starter that is selected.

7.  Choose a datacenter; *New York is selected in the example created for this document.*

8.  Additional options: none.

9.  Add the SSH keys.

10. Choose a host name, *e.g. `passivehouse-ecovillage`.*

11. Select the project.

12. Press "Create"

#### Configure droplet

1.  Note the IP address from above.

2.  `ssh root\@{ipaddress}`

3.  Remove the motd (message of the day).

4.  Add user:

```bash
    adduser username
    adduser username admin
    adduser username docker
    adduser username plugdev
    adduser username staff
```

5.  Disable root login via SSH or via password

6.  Optional: enable `username` to sudo without password.
```bash
    sudo VISUAL=vi visudo
```
 + Add the following line at the bottom:
```bash
    username ALL=(ALL) NOPASSWD: ALL
```
7.  Test that you can become `username`:
```console
    # sudo -i username
    username\@host-name:\~\$
```
8.  Drop back to root, and then copy the authorized_keys file to   `~username`:
```bash
    mkdir -m 700 \~username/.ssh
    cp -p .ssh/authorized_keys \~username/.ssh
    chown -R username.username \~username/.ssh/authorized_keys
```
9.  Confirm if the user can SSH in.

10.  Optional: set up byobu by default:
```bash
    byobu
    byobu-enable
```
1`.  Set the host name.
```bash
    vi /etc/hosts
```
   Change the line `127.0.1.1 name name` to `127.0.0.1 myhost.myfq.dn myhost`.

12.  If needed, use `hostnamectl` to set the static hostname to match `myhost`.

13.  set up Git:
```bash
    sudo add-apt-repository ppa:git-core/ppa
    sudo apt update
    sudo apt install git
```
14.  We'll put the docker files at `/opt/docker/docker-iot-dashboard`, setting up as follows:
 ```bash
sudo mkdir /opt/docker
cd /opt/docker
sudo chgrp admin .
sudo chmod g+w .
```

####
-------


After server is set up
----------------------

The following instructions are essentially independent of the cloud provider and the underlying distribution. But this was only tested on Ubuntu and (in 2019) on CentOS.

Clone this repository.
 ```bash
    git clone git\@github.com:mcci-catena/docker-iot-dashboard.git /opt/docker/dashboard.example.com
```

2.  Move to the directory populated in step 1.
 ```bash
    cd /opt/docker/dashboard.example.com
```
3.  Get a fully-qualified domain name (FQDN) for the server, for which the DNS can be controlled. Point it to the server. Make sure it works, using "`dig FQDN`" -- get back an `A` record pointing to your server's IP address.

### Create and edit the .env file

1.  Create a .env file. To get a template:
 ```bash
    sed -ne '/^#+++/,/^#---/p' docker-compose.yml | sed -e '/^#[^ \t]/d' -e '/^# TTN/s/$/=/' > .env
```
2.  Edit the .env file as follows:

    1.  `IOT_DASHBOARD_NGINX_FQDN=myhost.example.com`  this sets the name of the resulting server. It tells Nginx what it's serving out. It must be a fully-qualified domain name (FQDN) that resolves to the IP address of the container host.

    2.  `IOT_DASHBOARD_CERTBOT_FQDN=myhost.example.com`  this should be the same as `IOT_DASHBOARD_NGINX_FQDN`.

    3.  `IOT_DASHBOARD_CERTBOT_EMAIL=someone\@example.com`  this sets the contact email for Let's Encrypt. The script automatically accepts the Let's Encrypt terms of service, and this indicates who is doing the accepting.

    4.  `IOT_DASHBOARD_DATA=/full/path/to/directory/`  the trailing slash is required! This will put all the data file for this instance as subdirectories of the specified path. If this is undefined, `docker-compose` will print error messages and quit.

    5.  `IOT_DASHBOARD_GRAFANA_ADMIN_PASSWORD=SomethingVerySecretIndeed` this needs to be confidential. Indeed this sets the *initial* password for the Grafana admin login.This should be changed via the Grafana UI after booting the server.

    6.  `IOT_DASHBOARD_GRAFANA_SMTP_FROM_ADDRESS`   this sets the Grafana
        originating mail address.

    7.  `IOT_DASHBOARD_GRAFANA_INSTALL_PLUGINS`  this sets a list of Grafana plugins to install.

    8.  `IOT_DASHBOARD_INFLUXDB_INITIAL_DATABASE_NAME=demo` Change "demo" to the desired name of the initial database that will be created in InfluxDB.

    9.  `IOT_DASHBOARD_MAIL_HOST_NAME=myhost.example.com`  this sets the name of your mail server. Used by Postfix.

    10. `IOT_DASHBOARD_MAIL_DOMAIN=example.com`  this sets the domain name of your mail server. Used by Postfix.

    11. `IOT_DASHBOARD_NODERED_INSTALL_PLUGINS=node-red-node-example1
        node-red-node-example2`  this installs one or more Node-RED plug-ins.

    12. `IOT_DASHBOARD_TIMEZONE=Europe/Paris`  If not defined, the default time
        zone will be GMT.

The `.env` file should look like this:

```bash
### env file for configuring dashboard.example.com
IOT_DASHBOARD_NGINX_FQDN=dashboard.example.com
#       The fully-qualified domain name to be served by Nginx.
#
# IOT_DASHBOARD_AWS_ACCESS_KEY_ID=
# The access key for AWS for backups.
#
# IOT_DASHBOARD_AWS_DEFAULT_REGION=
# The AWS default region.
#
# IOT_DASHBOARD_AWS_S3_BUCKET_INFLUXDB=
# The S3 bucket to use for uploading the influxdb backup data.
#
# IOT_DASHBOARD_AWS_SECRET_ACCESS_KEY=
# The AWS API secret key for backing up influxdb data.
#
IOT_DASHBOARD_CERTBOT_EMAIL=somebody@example.com
#       The email address to be used for registering with Let's Encrypt.
#
IOT_DASHBOARD_CERTBOT_FQDN=dashboard.example.com
#       The domain(s) to be used by certbot when registering with Let's Encrypt.
#
IOT_DASHBOARD_DATA=/var/opt/docker/dashboard.example.com/
#       The path to the data directory. This must end with a '/', and must eithe
r
#       be absolute or must begin with './'. (If not, you'll get parse errors.)
#
IOT_DASHBOARD_GRAFANA_ADMIN_PASSWORD=...................
#       The password to be used for the admin user on first login. This is ignored
#       after the Grafana database has been built.
#
IOT_DASHBOARD_GRAFANA_PROJECT_NAME=My Dashboard
#       The project name to be used for the emails from the administrator.
#
# IOT_DASHBOARD_GRAFANA_LOG_MODE=
#       Set the grafana log mode.
#
# IOT_DASHBOARD_GRAFANA_LOG_LEVEL=
#       Set the grafana log level (e.g. debug)
#
IOT_DASHBOARD_GRAFANA_SMTP_ENABLED=true
#       Set to true to enable SMTP.
#
# IOT_DASHBOARD_GRAFANA_SMTP_SKIP_VERIFY=
#       Set to true to disable SSL verification.
#       Defaults to false.
#
# IOT_DASHBOARD_GRAFANA_INSTALL_PLUGINS=
#       A list of grafana plugins to install.
#
IOT_DASHBOARD_GRAFANA_SMTP_FROM_ADDRESS=grafana-admin@dashboard.example.com
# The "from" address for Grafana emails.
#
# IOT_DASHBOARD_GRAFANA_USERS_ALLOW_SIGN_UP=
#       Set to true to allow users to sign-up to get access to the dashboard.
#
IOT_DASHBOARD_INFLUXDB_ADMIN_PASSWORD=jadb4a4WH5za7wvp
#       The password to be used for the admin user by influxdb. Again, this is
#       ignored after the influxdb database has been built.
#
IOT_DASHBOARD_INFLUXDB_INITIAL_DATABASE_NAME=mydatabase
#       The inital database to be created on first launch of influxdb. Ignored
#       after influxdb has been launched.
#
IOT_DASHBOARD_MAIL_DOMAIN=example.com
# the postfix mail domain.
#
IOT_DASHBOARD_MAIL_HOST_NAME=dashboard.example.com
# the external FQDN for the mail host.
#
# IOT_DASHBOARD_MAIL_RELAY_IP=
# the mail relay machine, assuming that the real mailer is upstream from us.
#
IOT_DASHBOARD_NODERED_INSTALL_PLUGINS=node-red-node-example1 nodered-node-example2
#  Additional plugins to be installed for Node-RED.
#
# IOT_DASHBOARD_PORT_HTTP=
#       The port to listen to for HTTP. Primarily for test purposes. Defaults to
#       80.
#
# IOT_DASHBOARD_PORT_HTTPS=
#       The port to listen to for HTTPS. Primarily for test purposes. Defaults to
#       443.
#
# IOT_DASHBOARD_TIMEZONE=
#       The timezone to use. Defaults to GMT.
```

### Set up the Node-RED and InfluxDB API logins

1.  Prepare everything:
```bash
    docker-compose pull
    docker-compose build
```
If there are any errors, they need to be fixed before going on.

2.  Use `docker-compose run nginx /bin/bash` to launch a shell in the Nginx container.

    -   If this fails with the message, `ERROR: Couldn't connect to Docker daemon at http+docker://localunixsocket - is it running?`, then probably the user ID is not in the `docker` group. To fix this, `sudo adduser MYUSER docker`, where "MYUSER" is the login ID. Then (**very important**) log out and log back in.

3.  Change ownership of Nginx's /etc/nginx/authdata to user `www-data`.
```bash
    chown www-data /etc/nginx/authdata
```
4.  Add Nginx's /etc/nginx/authdata/.htpasswd.
```bash
    touch /etc/nginx/authdata/.htpasswd
    chown www-data /etc/nginx/authdata/.htpasswd
```
5.  Add user logins for node-red and influxdb queries Make `USERS` be a list of login IDs.
```bash
    export USERS="tmm amy josh"
    for USER in $USERS; do echo "Set password for "$USER; 
    htpasswd /etc/nginx/authdata/.htpasswd $USER; done
```
6.  Exit Nginx's container with Control+D.

Start the server
----------------

1. Starting things up in "interactive mode" is recommended.
```bash
    docker-compose up
```
This will show the log files. It will also be pretty clear if there are any issues.

One common error (for me, anyway) is entering an illegal initial InfluxDB database name. InfluxDB will spew a number of errors, but eventually it will start up anyway. But then the database needs to be created manually.

### Restart servers in the background

Once the servers are coming up interactively, use \^C to shut them down, and then restart in daemon mode.
```bash
    docker-compose up -d
```
### Initial testing

-   Open Grafana on [https://dashboard.example.com](https://dashboard.example.com/), and log in as admin.

-   Change the admin password.

### Set up first data source

Use the Grafana UI -- either click on "add first data source" or use "Configure\>Add Data Source", and add an InfluxDB data source.

-   Set the URL (under HTTP Settings) to `<http://influxdb:8086>`.

-   Select the database. If InfluxDB is properly initialized in a database, connect to it as a Grafana data source. If not, [create an InfluxDB database](https://github.com/mcci-catena/docker-iot-dashboard/blob/master/SETUP.md#creating-an-influxdb-database).

-   Leave user and password blank.

-   Click "Save & Test".

### Test Node-RED

Open Node-RED on <https://dashboard.example.com/node-red/>, and build a flow that stores data in InfluxDB. **Be sure to add the trailing slash! Otherwise a 404 error pops from Grafana. This will be fixed soon.**

### Creating an InfluxDB database

To create a database, log in to the host machine, and cd to `/opt/docker/dashboard.example.com`. Use the following commands:

```console
$ docker-compose exec influxdb /bin/bash
# influx
Connected to http://localhost:8086 version 1.7.6
InfluxDB shell version: 1.7.6
Enter an InfluxQL query
> show databases
name: databases
name
----
_internal
> create database "my-new-database"
> show databases
name: databases
name
----
_internal
my-new-database
> ^D
# ^D
$
```
### Add Nginx log in for NodeRed or query after the fact

To add a user with Node-RED access or query access, follow this procedure.

1.  Log into the host machine

2.  Change the directory (cd) to `/opt/docker/dashboard.example.com`.

3.  log into the Nginx docker container.
```console
    $ docker-compose exec Nginx /bin/bash
```
4.  In the container, move to the authdata directory.
```console
    # cd /etc/nginx/authdata
```
5.  Add the user.
```console
# htpasswd .htpasswd {newuserid}
New password:
Re-type new password:
Adding password for user {newuserid}
```
6.  Close the connection to nginx (Ctrl+D).

### MQTT User Credentials setup

To access mqtt channel, user needs credentials to access it.

1.  Log into the host machine

2.  Change the directory (cd) to `/opt/docker/dashboard.example.com`.

3.  log into the mqtts docker container.
```bash
    $ docker-compose exec mqtts /bin/bash
```
4.  In the container,
```bash
    # mosquitto_passwd -c /etc/mosquitto/credentials/passwd \<user\>
    Password:
    Reenter password:
```
5.  Close the connection to mqtts (Ctrl+D).
