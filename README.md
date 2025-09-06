# Custom Honeypot SIEM on AWS (Elasticsearch + Kibana + Logstash)

This project sets up a honeypot (Cowrie) feeding into an ELK stack on AWS EC2. It collects, parses, and visualizes attacker activity in near real time.

---

## Architecture
- **EC2 #1 (elk-core):** Elasticsearch, Kibana, Logstash
- **EC2 #2 (honeypot):** Cowrie + Filebeat
- Real SSH moved to port **2222**; Cowrie listens on port **22**
- Filebeat ships logs to Logstash → Elasticsearch → Kibana

**Required Ports**
- 9200 (Elasticsearch API, internal only)
- 9300 (Elasticsearch transport, internal only)
- 5601 (Kibana UI)
- 5044 (Logstash Beats input)
- 22 (Cowrie fake SSH, open to world)
- 2222 (real SSH, restricted to your IP)

---

## Setup Steps

### 1. Launch EC2 Instances
- Create two Ubuntu EC2 instances
- Configure security groups as above

*[Screenshot of EC2 instances + security groups]*

### 2. Install Elasticsearch, Kibana, Logstash (elk-core)
   ```bash
   sudo apt update
   sudo apt install -y elasticsearch kibana logstash
   ```

Configure Elasticsearch (`/etc/elasticsearch/elasticsearch.yml`):
```yaml
cluster.name: elk-lab
node.name: elk-core-1
network.host: 0.0.0.0
discovery.type: single-node
```

Configure Kibana (`/etc/kibana/kibana.yml`):
```yaml
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://127.0.0.1:9200"]
```

Create Logstash pipeline (`/etc/logstash/conf.d/10-beats.conf`):
```conf
input { beats { port => 5044 } }
output {
  elasticsearch {
    hosts => ["http://127.0.0.1:9200"]
    index => "honeypot-%{+YYYY.MM.dd}"
  }
}
```

*[Screenshot of Elasticsearch health + Kibana login]*

### 3. Install Cowrie + Filebeat (honeypot)
# Install Cowrie
   ```
   sudo adduser cowrie
   su - cowrie
   git clone https://github.com/cowrie/cowrie.git
   cd cowrie
   python3 -m venv cowrie-env
   source cowrie-env/bin/activate
   pip install -r requirements.txt
   ```
# Move your system SSH daemon to port 2222
   ```bash
   sudo nano /etc/ssh/sshd_config
   ```
Find the line:
   ```
   #Port 22
   ```
and change it to:
   ```
   Port 2222
   ```
then, restart the SSH service:
   ```bash
   sudo systemctl restart ssh
   ```
WARNING! Test that it works on another machine (do this before closing your current session):
   ```bash
   ssh -p 2222 youruser@yourserver
   ```

# Enable Cowrie to bind port 22 with authbind, then set up systemd service.

Install `authbind`:

   **Debian/Ubuntu:**
   ```bash
   sudo apt install authbind
   ```
Allow port 22 for the `cowrie` user:
   ```bash
   sudo touch /etc/authbind/byport/22
   sudo chown cowrie:cowrie /etc/authbind/byport/22
   sudo chmod 500 /etc/authbind/byport/22
   ```
Edit your Cowrie config (`cowrie.cfg`):
   ```ini
   [ssh]
   listen_endpoints = tcp:22:interface=0.0.0.0
   ```

Start Cowrie with `authbind`:
   ```bash
   authbind --deep bin/cowrie start
   ```

Test honeypot (Cowrie):
  ```bash
  ssh someuser@yourserver
  ```

Test real SSH:
  ```bash
  ssh -p 2222 youruser@yourserver
  ```

Install Filebeat + Cowrie module:
   ```bash
   sudo apt install -y filebeat
   sudo filebeat modules enable cowrie
   ```
Configure Filebeat output to Logstash:
   ```yaml
   output.logstash:
     hosts: ["<ELK_INSTANCE_PRIVATE_IP>:5044"]
   ```


*[Screenshot of Cowrie logs + Filebeat config]*

### 4. Create Index Template
Example:
```json
PUT _index_template/honeypot {
  "index_patterns": ["honeypot-*"]
}
```

*[Screenshot of Kibana Index Template view]*

### 5. Verify Data Flow
- Connect to honeypot via SSH on port 22
- Logs should appear in Kibana Discover

*[Screenshot of Discover view with honeypot logs]*

### 6. Build Visualizations
- **Lens bar chart**: top usernames/passwords
- **Line chart**: attacks over time
- **Table**: top source IPs

*[Screenshot of dashboard overview with multiple visualizations]*

---

## Next Steps
- Add multiple honeypots (Cowrie, Dionaea, Conpot, etc.)
- Automate deployment with T-Pot or Ansible
- Add ILM policies for data retention
- Enrich with GeoIP + ASN data

---

## Repo Structure
```
README.md          # Documentation
logstash/          # Pipeline configs
filebeat/          # Example filebeat.yml and module configs
dashboards/        # Kibana dashboard exports (NDJSON)
```

---

## Disclaimer
For educational and research use only. Do not expose production credentials or networks to honeypots.
