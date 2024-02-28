
## Download prometheus
- Download prometheus `https://prometheus.io/download/`
- Example version `wget https://github.com/prometheus/prometheus/releases/download/v2.49.1/prometheus-2.49.1.linux-amd64.tar.gz`
- Extract `tar -xvzf prometheus-2.49.1.linux-amd64.tar.gz`
- Check version `./prometheus --version`

## Run prometheus
- `./prometheus` run on localhost:9090
- Run on the background `nohup ./prometheus > prometheus.log 2>&1 &`

## Exporter
### Node exporter
- Download & extract node exporter on target machine
- run `./node_exporter` or run on the background `nohup ./node_exporter > node_exporter.txt 2>&1 &`
- Node exporer by default run on `localhost:9100`
- On prometheus server edit `prometheus.yml`
  scrape_configs:
  - job_name: "node_exporter"
    static_configs:
      - targets: ["server1:9100", "server2:9100"]

### MySQL exporter
- Download & extract mysql exporter on target machine
- Prepare MySQL user for exporter
  ```sql
  CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'password' WITH MAX_USER_CONNECTIONS 3;
  GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
  ```
- Register exporter as service
  1. Create group `sudo groupadd --system mysql_exporter`
  2. Create user and join the group `sudo useradd -s /sbin/nologin --system -g mysql_exporter mysql_exporter`
  3. Create exporter config file `sudo vi /etc/.mysqld_exporter.cnf`
     ```ini
     [client] 
     user=exporter 
     password=password 
     ```
  4. Set config permission `sudo chown root:mysql_exporter /etc/.mysqld_exporter.cnf`
  5. Create systemd unit file
     ```ini
     [Unit]
     Description=Prometheus MySQL Exporter
     After=network.target
     User=prometheus
     Group=prometheus
 
     [Service]
     Type=simple
     Restart=always
     ExecStart=/usr/local/bin/mysqld_exporter \
        --config.my-cnf /etc/.mysqld_exporter.cnf \
        --collect.global_status \
        --collect.info_schema.innodb_metrics \
        --collect.auto_increment.columns \
        --collect.info_schema.processlist \
        --collect.binlog_size \
        --collect.info_schema.tablestats \
        --collect.global_variables \
        --collect.info_schema.query_response_time \
        --collect.info_schema.userstats \
        --collect.info_schema.tables \
        --collect.perf_schema.tablelocks \
        --collect.perf_schema.file_events \
        --collect.perf_schema.eventswaits \
        --collect.perf_schema.indexiowaits \
        --collect.perf_schema.tableiowaits \
        --collect.slave_status \
        --web.listen-address=0.0.0.0:9104
    
     [Install]
     WantedBy=multi-user.target
     ```
  6. Reload systemd `sudo systemctl daemon-reload`
  7. Enable service `sudo systemctl enable mysql_exporter`
  8. Start service `sudo systemctl start mysql_exporter`
- Configure prometheus `sudo nano /etc/prometheus/prometheus.yml`
  ```yml
  - job_name: mysql_server1
  static_configs: 
   - targets: ['<Machine_IP>:9104'] 
     labels: 
       alias: db1
  ```
- Reload prometheus `sudo systemctl restart prometheus`


## Setup grafana
- Install the prerequisite `sudo apt-get install -y apt-transport-https software-properties-common wget`
- Import the GPG
  `sudo mkdir -p /etc/apt/keyrings/`
  `wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null`
- To add a repository for stable releases, run the following command:
  `echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list`
- Update the packages `sudo apt-get update`
- Install grafana `sudo apt-get install grafana`
- Enable grafana service `sudo systemctl enable grafana-server`
- Start grafana `sudo systemctl start grafana-server`
- Check grafana status `sudo systemctl status grafana-server`
- Access web admin `http://your-instance-ip:3000` login with username: admin, password: admin
- Change password and continue
- Add prometheus data source, from sidebar go to Configuration - Data Sources - Add data source - choose Prometheus - enter http://localhost:9090 and click Save & Test
- Import dashboard https://grafana.com/grafana/dashboards search for “node exporter” to find available dashboards. Copy the dashboard ID
- Click “Create” in the left sidebar to create a new dashboard. Click "Import" and paste the dashboard ID you copied earlier.
    Load the dashboard and select the Prometheus data source. Click "Import"
