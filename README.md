# ELK Stack Installation Guide for Ubuntu 24.04 LTS

This guide provides step-by-step instructions for installing and configuring the ELK Stack (Elasticsearch, Logstash, Kibana) with Filebeat on Ubuntu 24.04 LTS for centralized log management and analysis.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Step 1: Install Java](#step-1-install-java-for-elastic-stack)
3. [Step 2: Install Elasticsearch](#step-2-install-elasticsearch)
4. [Step 3: Configure Elasticsearch](#step-3-configure-elasticsearch)
5. [Step 4: Install Logstash](#step-4-install-logstash)
6. [Step 5: Install Kibana](#step-5-install-kibana)
7. [Step 6: Configure Kibana](#step-6-configure-kibana)
8. [Step 7: Install and Configure Filebeat](#step-7-install-and-configure-filebeat)
9. [Verification](#verification)

---

## Prerequisites

- Ubuntu 24.04 LTS server
- Root or sudo access
- Minimum 4GB RAM recommended
- Stable internet connection

---

## Step 1: Install Java for Elastic Stack

Elastic Stack components require Java to run. We'll install OpenJDK 17.

### Update System Packages

```bash
sudo apt update
```

### Install Required Dependencies

```bash
sudo apt install apt-transport-https
```

### Install OpenJDK 17

```bash
sudo apt install openjdk-17-jdk -y
```

### Verify Java Installation

```bash
java -version
```

### Set JAVA_HOME Environment Variable

Edit the environment file:

```bash
sudo nano /etc/environment
```

Add the following line at the end:

```
JAVA_HOME="/usr/lib/jvm/java-17-openjdk-amd64"
```

Apply the changes:

```bash
source /etc/environment
```

Verify the configuration:

```bash
echo $JAVA_HOME
```

---

## Step 2: Install Elasticsearch

Elasticsearch is the core search and analytics engine of the ELK Stack.

### Import the GPG Key

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
```

### Add Elasticsearch Repository

```bash
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
```

### Update Package Lists

```bash
sudo apt-get update
```

### Install Elasticsearch

```bash
sudo apt-get install elasticsearch
```

### Start and Enable Elasticsearch

```bash
sudo systemctl start elasticsearch
sudo systemctl enable elasticsearch
```

### Verify Service Status

```bash
sudo systemctl status elasticsearch
```

---

## Step 3: Configure Elasticsearch

### Edit Configuration File

```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

### Modify the Following Settings

- **Network Host**: Uncomment and set to `0.0.0.0` to allow external access
- **Discovery**: Uncomment `discovery.seed_hosts: []`
- **Security**: For basic setup (not recommended for production), disable security features by setting `security.enabled: false`

### Restart Elasticsearch

```bash
sudo systemctl restart elasticsearch
```

### Test the Configuration

```bash
curl -X GET "localhost:9200"
```

You should receive a JSON response with cluster information.

### Access via Browser

Navigate to `http://<your-server-ip>:9200` (default Elasticsearch port)

---

## Step 4: Install Logstash

Logstash processes and forwards log data to Elasticsearch.

### Install Logstash

```bash
sudo apt-get install logstash -y
```

### Start and Enable Logstash

```bash
sudo systemctl start logstash
sudo systemctl enable logstash
```

### Verify Service Status

```bash
sudo systemctl status logstash
```

### Create Logstash Configuration

Create a configuration file for auth logs:

```bash
sudo nano /etc/logstash/conf.d/authlog.conf
```

Add the following configuration:

```
input {
  beats {
    port => 5044
  }
}

filter {
  if [log_type] == "auth" {
    grok {
      match => { "message" => "%{SYSLOGTIMESTAMP:timestamp} %{HOSTNAME:host} %{DATA:program}(?:\[%{POSINT:pid}\])?: %{GREEDYDATA:auth_message}" }
    }
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "auth-logs-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "your_password"
  }
}
```

### Restart Logstash

```bash
sudo systemctl restart logstash
```

---

## Step 5: Install Kibana

Kibana provides a web interface for visualizing Elasticsearch data.

### Install Kibana

```bash
sudo apt-get install kibana
```

### Start and Enable Kibana

```bash
sudo systemctl start kibana
sudo systemctl enable kibana
```

### Verify Service Status

```bash
sudo systemctl status kibana
```

---

## Step 6: Configure Kibana

### Edit Configuration File

```bash
sudo nano /etc/kibana/kibana.yml
```

### Uncomment and Adjust the Following Lines

```yaml
server.port: 5601
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://localhost:9200"]
```

### Restart Kibana

```bash
sudo systemctl restart kibana
```

### Access Kibana Dashboard

Navigate to `http://<your-server-ip>:5601` in your web browser.

You can start by adding integrations or exploring on your own.

---

## Step 7: Install and Configure Filebeat

Filebeat is a lightweight shipper for forwarding and centralizing log data.

### Import GPG Key (if not already done)

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
```

### Add Repository (if not already done)

```bash
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
```

### Update Package Lists

```bash
sudo apt-get update
```

### Install Filebeat

```bash
sudo apt-get install filebeat
```

### Configure Filebeat

Edit the main configuration file:

```bash
sudo nano /etc/filebeat/filebeat.yml
```

Add the following configuration:

```yaml
filebeat.inputs:
- type: filestream
  id: auth-logs-input
  enabled: true
  paths:
    - /var/log/auth.log
    - /var/log/auth.log.*
  fields:
    log_type: auth
  fields_under_root: true

filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: true
  reload.period: 10s

output.logstash:
  hosts: ["16.16.79.236:5044"]

setup.kibana:
  host: "16.16.79.236:5601"

processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
```

### Enable System Module

```bash
sudo filebeat modules enable system
```

This command activates the system module by renaming `/etc/filebeat/modules.d/system.yml.disabled` to `/etc/filebeat/modules.d/system.yml`.

### Configure System Module

Edit the system module configuration:

```bash
sudo nano /etc/filebeat/modules.d/system.yml
```

Configure as follows:

```yaml
- module: system
  
  syslog:
    enabled: true 
  
  auth:
    enabled: true
    var.paths: ["/var/log/auth.log*"]
```

### Test Configuration

Check for syntax errors:

```bash
sudo filebeat test config
```

Expected output: `Config OK ✅`

Test Logstash connectivity:

```bash
sudo filebeat test output
```

Expected output showing successful connection to Logstash.

### Start Filebeat

```bash
sudo systemctl restart filebeat
sudo systemctl status filebeat
```

Expected status: `active (running) ✅`

---

## Verification

### Create Data View in Kibana

1. Navigate to **Stack Management** → **Data Views** → **Create data view**
2. Configure:
   - **Name**: `filebeat-auth-logs`
   - **Index pattern**: `filebeat-*`
   - **Timestamp field**: `@timestamp`
3. Click **Save**

### View Logs in Discover

1. Go to **Analytics** → **Discover**
2. Select data view: `filebeat-auth-logs`
3. Add filter: `log_type: "auth"`
4. View columns:
   - `@timestamp`
   - `message` (contains auth.log content)
   - `host.name`
   - `log.file.path`

### Result

Auth.log contents successfully displayed in Kibana! 🎉

---

## Default Ports

| Service | Port |
|---------|------|
| Elasticsearch | 9200 |
| Kibana | 5601 |
| Logstash | 5044 |

---

## Security Notes

⚠️ **Warning**: The configuration in this guide disables security features for basic setup purposes. This is **NOT recommended for production environments**.

For production deployments:
- Enable Elasticsearch security features
- Configure SSL/TLS encryption
- Implement proper authentication and authorization
- Use strong passwords
- Configure firewall rules

---

## Troubleshooting

### Check Service Logs

```bash
# Elasticsearch logs
sudo journalctl -u elasticsearch -f

# Logstash logs
sudo journalctl -u logstash -f

# Kibana logs
sudo journalctl -u kibana -f

# Filebeat logs
sudo journalctl -u filebeat -f
```

### Common Issues

- **Service not starting**: Check logs for error messages
- **Connection refused**: Verify service is running and firewall rules
- **No data in Kibana**: Verify Filebeat is sending data to Logstash
- **Index not created**: Check Logstash output configuration

---

## License

This guide is provided as-is for educational purposes.

## Support

For official documentation:
- [Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Logstash](https://www.elastic.co/guide/en/logstash/current/index.html)
- [Kibana](https://www.elastic.co/guide/en/kibana/current/index.html)
- [Filebeat](https://www.elastic.co/guide/en/beats/filebeat/current/index.html)
