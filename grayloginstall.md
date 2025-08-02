# Taming the Log Tsunami: Your Guide to Graylog on Ubuntu 24.04

Ever felt the frustration of jumping between a dozen different servers, `tail`-ing log files, and trying to piece together what went wrong during an outage? 😩 It's a digital scavenger hunt nobody wants to play. This is where centralized logging comes to the rescue.

**Graylog** is a powerful, open-source platform that collects, indexes, and analyzes log data from virtually any source. Instead of your logs being scattered across your entire infrastructure, they're all sent to one central place. This allows you to:

* **Search everything at once**: Find errors across all your servers in seconds.
* **Create alerts**: Get notified automatically when specific errors occur (e.g., failed logins, application crashes).
* **Build dashboards**: Visualize your data to spot trends and potential problems before they become critical.

In this guide, we'll walk you through setting up a complete Graylog server from scratch on **Ubuntu 24.04**.

***

### → A Note on Server Specs: RAM/CPU

Don't be shy with resources here. A logging server is constantly ingesting, indexing, and serving up data. For a home lab or a small number of busy servers, a virtual machine with **8 vCPUs and 12GB of RAM** is a great starting point. The more logs you throw at it, the more resources it will appreciate!

***

## The A-Team: Installing the Graylog Stack

Graylog doesn't work alone. It relies on a couple of key sidekicks to get the job done. We'll be installing everything on one server for a simple, all-in-one setup.

### → Part 1: MongoDB - The Configuration Brain 🧠

First up, we need a database to store Graylog's configuration. **MongoDB** is a NoSQL database that Graylog uses to keep track of all its settings—user accounts, dashboards, stream rules, inputs, and more.

**Important**: MongoDB does **not** store your log messages. It only stores the application's configuration.

The instructions on the official Graylog page for MongoDB 6.x don't play nicely with Ubuntu 24.04. We'll use the recommended version 7.x instead.

1.  **Install prerequisites and import the official MongoDB GPG key**: This ensures the packages we download are authentic.
    ```bash
    sudo apt-get install gnupg curl
    curl -fsSL [https://www.mongodb.org/static/pgp/server-7.0.asc](https://www.mongodb.org/static/pgp/server-7.0.asc) | \
       sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg \
       --dearmor
    ```

2.  **Add the MongoDB repository**: This tells `apt` where to find the MongoDB packages.
    ```bash
    echo "deb [ signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] [http://repo.mongodb.org/apt/debian](http://repo.mongodb.org/apt/debian) bookworm/mongodb-org/7.0 main" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
    ```

3.  **Update `apt` and install MongoDB**:
    ```bash
    sudo apt-get update
    sudo apt-get install -y mongodb-org
    ```
    If you see a warning about the `ulimit` for open files, you can increase it to the recommended value:
    ```bash
    ulimit -n 64000
    ```
4.  **Start and enable the MongoDB service**: This ensures it starts automatically on boot.
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable mongod.service
    sudo systemctl restart mongod.service
    ```

5.  **Check that it's running**:
    ```bash
    sudo systemctl status mongod.service
    ```
    You should see `Active: active (running)`.

6.  **Lock the version**: This is a smart move to prevent automatic updates from `apt` from accidentally breaking your Graylog setup. We'll hold the package at its current version.
    ```bash
    sudo apt-mark hold mongodb-org
    ```

***

### → Part 2: OpenSearch - The Data Powerhouse ⚡

With the configuration database ready, we need a place to store the actual log messages. **OpenSearch** is a high-performance search and analytics engine. It takes the logs Graylog processes and stores them in a way that makes them incredibly fast to search and analyze. When you ask Graylog to "find all errors from web-server-01 in the last hour," it's OpenSearch that does the heavy lifting.

1.  **Install prerequisites and the OpenSearch GPG key**:
    ```bash
    sudo apt-get update && sudo apt-get -y install lsb-release ca-certificates curl gnupg2
    curl -o- [https://artifacts.opensearch.org/publickeys/opensearch.pgp](https://artifacts.opensearch.org/publickeys/opensearch.pgp) | sudo gpg --dearmor --batch --yes -o /usr/share/keyrings/opensearch-keyring
    ```

2.  **Add the OpenSearch repository**:
    ```bash
    echo "deb [signed-by=/usr/share/keyrings/opensearch-keyring] [https://artifacts.opensearch.org/releases/bundle/opensearch/2.x/apt](https://artifacts.opensearch.org/releases/bundle/opensearch/2.x/apt) stable main" | sudo tee /etc/apt/sources.list.d/opensearch-2.x.list
    ```
3.  **Update `apt` and install OpenSearch**:
    ```bash
    sudo apt-get update
    sudo apt-get -y install opensearch
    ```
4.  **Lock the version**: Just like with MongoDB, we'll prevent accidental upgrades.
    ```bash
    sudo apt-mark hold opensearch
    ```

#### OpenSearch Configuration for Graylog

Before we start OpenSearch, we need to tune it for our all-in-one Graylog server.

1.  **Edit the main configuration file**:
    ```bash
    sudo nano /etc/opensearch/opensearch.yml
    ```
    Add or modify the following lines. This tells OpenSearch to run as a single-node cluster (since it's all on one server) and disables the security plugin for simplicity in this initial setup.
    ```yaml
    cluster.name: graylog
    discovery.type: single-node
    action.auto_create_index: false
    plugins.security.disabled: true
    ```
2.  **Adjust Memory Allocation (JVM Options)**: OpenSearch runs on Java and loves memory. A good rule of thumb is to give it 50% of your server's RAM, but no more than 32GB. For our 12GB server, we'll give it 6GB.
    ```bash
    sudo nano /etc/opensearch/jvm.options
    ```
    Change the `-Xms` (initial heap size) and `-Xmx` (maximum heap size) values.
    ```
    # Change these lines:
    -Xms1g
    -Xmx1g
    
    # To this (for a 12GB RAM server):
    -Xms6g
    -Xmx6g
    ```

3.  **Tweak Kernel Settings**: OpenSearch requires a high `max_map_count` value for memory management.
    ```bash
    sudo nano /etc/sysctl.conf
    ```
    Add this line to the end of the file to make the change permanent across reboots:
    ```
    vm.max_map_count=262144
    ```
    Apply the setting immediately without rebooting:
    ```bash
    sudo sysctl -w vm.max_map_count=262144
    ```
4.  **Start and enable OpenSearch**:
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable opensearch.service
    sudo systemctl start opensearch.service
    ```
    Check its status to ensure it's running. Don't worry about the `System::setSecurityManager` warnings; they are expected.

***

### → Part 3: Graylog - The Main Event 🌟

With our database and search engine in place, it's time to install the star of the show! **Graylog Server** is the application that ties everything together. It runs the web interface, processes incoming logs, and communicates with MongoDB and OpenSearch.

1.  **Download and install the Graylog repository package**:
    ```bash
    wget [https://packages.graylog2.org/repo/packages/graylog-6.0-repository_latest.deb](https://packages.graylog2.org/repo/packages/graylog-6.0-repository_latest.deb)
    sudo dpkg -i graylog-6.0-repository_latest.deb
    sudo apt-get update
    sudo apt-get install graylog-server
    ```

2.  **Lock the version**: Consistency is key!
    ```bash
    sudo apt-mark hold graylog-server
    ```

#### Configure Graylog Server

We need to generate a couple of secret keys and tell Graylog how to connect to OpenSearch.

1.  **Generate a `password_secret`**: This is a long, random string used by Graylog to secure sensitive data like user passwords internally.
    ```bash
    < /dev/urandom tr -dc A-Z-a-z-0-9 | head -c96;echo;
    ```
    **Copy the output!** It will look something like this: `CjXG...M0CWP`.

2.  **Generate a `root_password_sha2`**: This will be the hashed version of the password for your `admin` user login.
    ```bash
    echo -n "Enter Password: " && head -1 </dev/stdin | tr -d '\n' | sha256sum | cut -d" " -f1
    ```
    You will be prompted to type your desired admin password. The command will then output a SHA256 hash of it. **Copy this hash!** It will look like: `5fab...289fb`.

3.  **Edit the Graylog configuration file**:
    ```bash
    sudo nano /etc/graylog/server/server.conf
    ```
    Now, find and update these three crucial settings:
    * `password_secret`: Paste the long random string you generated first.
    * `root_password_sha2`: Paste the password hash you generated second.
    * `http_bind_address`: Change this to `0.0.0.0:9000` to make the web interface accessible from other machines on your network.
    * `elasticsearch_hosts`: Tell Graylog where to find OpenSearch. Since it's on the same server, we use `http://127.0.0.1:9200`.

    Your config should look like this:
    ```ini
    password_secret = CjXGDLNLZskEJdPPKnEd1UnvJfsU7iAnePevgw-0Sq0K7ynw1sC3WDF-8L09SHG6lZqCGFweTAhLU1ai2zLFszc9LvbM0CWP
    root_password_sha2 = 5fab5f43d0b2f97526e7daacb1e42ffd178e2067c59a0036d0d984a3bec289fb
    http_bind_address = 0.0.0.0:9000
    elasticsearch_hosts = [http://127.0.0.1:9200](http://127.0.0.1:9200)
    ```
4.  **Start and enable Graylog**:
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable graylog-server.service
    sudo systemctl start graylog-server.service
    ```

***

## Final Health Check ✅

Let's make sure all three services are up and running happily.

```bash
sudo systemctl status mongod
sudo systemctl status opensearch
sudo systemctl status graylog-server