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
- Download node exporter `wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz`
- Extract `sudo tar -xvzf node_exporter-1.7.0.linux-amd64.tar.gz` and `cd node_exporter-1.7.0.linux-amd64`
- Run `./node_exporter` or run on the background `nohup ./node_exporter > node_exporter.txt 2>&1 &`
- Node exporer by default run on `localhost:9100`
- To make as a service, do the following things
- Create user group `sudo groupadd --system node-exporter`
- Create user for the exporter `sudo useradd --system --no-create-home --shell /bin/false -g node-exporter node-exporter`
- Move exporter binary `sudo mv node_exporter /usr/local/bin/`
- Change ownership `sudo chown node-exporter:node-exporter /usr/local/bin/node_exporter`
- Create systemd unit config `sudo nano /usr/lib/systemd/system/node-exporter.service`
  ```ini
  [Unit]
  Description=Node Exporter
  Wants=network-online.target
  After=network-online.target
  StartLimitIntervalSec=500
  StartLimitBurst=5

  [Service]
  User=node-exporter
  Group=node-exporter
  Type=simple
  Restart=on-failure
  RestartSec=5s
  ExecStart=/usr/local/bin/node_exporter \
    --collector.logind \
    --collector.systemd \
    --collector.processes

  [Install]
  WantedBy=multi-user.target
  ```
- Set file permission `sudo chmod 664 /usr/lib/systemd/system/node-exporter.service`
- Enable to start on boot `sudo systemctl enable node-exporter.service`
- Reload daemon `sudo systemctl daemon-reload`
- Start the service `sudo systemctl start node-exporter`
- On prometheus server edit `prometheus.yml`
  ```ini
  scrape_configs:
  - job_name: "node_exporter"
    static_configs:
      - targets: ["server1:9100", "server2:9100"]
  ```

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
  3. Create exporter config file `sudo nano /etc/.mysqld_exporter.cnf`
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

### Nginx exporter
- Create nginx status config `nano /etc/nginx/sites-available/status.conf`
  ```nginx
  server {
        listen 80;
        server_name <ip-machine>;
        location = /basic-status {
                allow 127.0.0.1;
                allow <ip-allowed-to-access>;
                deny all;
                stub_status;
        }
  }
  ```
- Enable status site `sudo ln -s /etc/nginx/sites-available/status.conf /etc/nginx/sites-enabled/status.conf`
- Reload configuration `sudo service nginx reload`
- Make sure the status page is available from the other machine `curl http://<ip-nginx>/basic-status`
- Download nginx-prometheus-exporter `wget https://github.com/nginxinc/nginx-prometheus-exporter/releases/download/v1.1.0/nginx-prometheus-exporter_1.1.0_linux_amd64.tar.gz`
- Extract `sudo tar -xvzf nginx-prometheus-exporter_1.1.0_linux_amd64.tar.gz`
- Create user for the exporter `sudo useradd --system --no-create-home --shell /bin/false nginx-exporter`
- Move `sudo mv nginx-prometheus-exporter /usr/local/bin/`
- Set ownership `sudo chown nginx-exporter:nginx-exporter /usr/local/bin/nginx-prometheus-exporter`
- Create systemd unit config `sudo nano /etc/systemd/system/nginx-exporter.service`
  ```ini 
  [Unit]
  Description=Nginx Exporter
  Wants=network-online.target
  After=network-online.target

  StartLimitIntervalSec=0

  [Service]
  User=nginx-exporter
  Group=nginx-exporter
  Type=simple
  Restart=on-failure
  RestartSec=5s
  ExecStart=/usr/local/bin/nginx-prometheus-exporter \
    --nginx.scrape-uri=http://<Nginx_IP>/basic-status

  [Install]
  WantedBy=multi-user.target
  ```
- Enable service `sudo systemctl enable nginx-exporter`
- Start service `sudo systemctl start nginx-exporter`
- Configure prometheus `sudo nano /etc/prometheus/prometheus.yml`
  ```yml
  - job_name: "nginx"
    static_configs:
      - targets: ["<Machine_IP>:9113"]
        labels:
          alias: "Nginx"
  ```
- Reload prometheus `sudo systemctl restart prometheus`

### PHP-FPM exporter
- PHP-FPM exporter version: https://github.com/Lusitaniae/phpfpm_exporter/tree/master
- Edit fpm config, enable status `sudo nano /etc/php/8.1/fpm/pool.d/www.conf`
  ```ini
  pm.status_path = /status
  ping.path = /ping
  ```
- Enable http to access php-fpm status
  ```nginx
  server {
        listen 80;
        server_name <ip-machine>;

        location ~ ^/(status|ping)$ {
                access_log off;
                allow 127.0.0.1;
                allow <ip-allowed-to-access>;
                deny all;
                include fastcgi_params;
                fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
                fastcgi_index /status;
                fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        }
  }
  ```
- Enable status site `sudo ln -s /etc/nginx/sites-available/status.conf /etc/nginx/sites-enabled/status.conf`
- Reload configuration `sudo service nginx reload`
- Make sure the status page is available from the other machine `curl http://<ip-target>/status` and `curl http://<ip-target>/ping`
- Download exporter `wget https://github.com/Lusitaniae/phpfpm_exporter/releases/download/v0.6.0/phpfpm_exporter-0.6.0.linux-amd64.tar.gz`
- Extract `sudo tar -xvzf phpfpm_exporter-0.6.0.linux-amd64.tar.gz`
- Create user for the exporter `sudo useradd --system --no-create-home --shell /bin/false php-fpm-exporter`
- Move to location`sudo mv phpfpm_exporter /usr/local/bin/`
- Set ownership `sudo chown php-fpm-exporter:php-fpm-exporter /usr/local/bin/phpfpm_exporter`
- Download php opcache exporter `wget https://github.com/Lusitaniae/phpfpm_exporter/blob/master/contrib/php_opcache_exporter.php`
- Move to location `mv php_opcache_exporter.php /etc/php-exporter/`
- Set ownership `sudo chown php-fpm-exporter:php-fpm-exporter /etc/php-exporter -R`
- Create systemd unit config `sudo nano /etc/systemd/system/php-fpm-exporter.service`
  ```ini 
  [Unit]
  Description = PHP-FPM Prometheus Exporter

  [Service]
  SyslogIdentifier = phpfpm_exporter
  ExecStart = /usr/local/bin/phpfpm_exporter \
    --phpfpm.socket-paths="/run/php/php8.1-fpm.sock" \
    --phpfpm.script-collector-paths="/etc/php-exporter/phpfpm_opcache_exporter.php" \
    --phpfpm.status-path="/status" \
    --web.listen-address=":9253"

  [Install]
  WantedBy = multi-user.target
  ```
- Enable service `sudo systemctl enable php-fpm-exporter`
- Start service `sudo systemctl start php-fpm-exporter`
- Configure prometheus `sudo nano /etc/prometheus/prometheus.yml`
  ```yml
  - job_name: "php-fpm"
    static_configs:
      - targets: ["<Machine_IP>:9253"]
        labels:
          alias: "PHP-FPM"
  ```
- Reload prometheus `sudo systemctl restart prometheus`

## Alert Manager
- Download alert manager `wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz`
- Extract `sudo tar -xvzf alertmanager-0.27.0.linux-amd64.tar.gz` and `cd alertmanager-0.27.0.linux-amd64`
- Create user group `sudo groupadd -f alert-manager`
- Create user for the exporter `sudo useradd -g alertmanager --system --no-create-home --shell /bin/false alert-manager`
- Move alertmanager binary `sudo mv alertmanager /usr/local/bin/`
- Move alertmanager tool binary `sudo mv amtool /usr/local/bin/`
- Change ownership `sudo chown alert-manager:alert-manager /usr/local/bin/alertmanager`
- `sudo chown alert-manager:alert-manager /usr/local/bin/amtool`
- Create configuration directory `sudo mkdir -p /etc/alertmanager`
- Move alertmanager config `sudo mv alertmanager.yml /etc/alertmanager/alertmanager.yml`
- Add permission to it `sudo chown alert-manager:alert-manager /etc/alertmanager -R`
- Create storage directory `sudo mkdir /var/lib/alertmanager`
- Add permission to it `sudo chown alert-manager:alert-manager /var/lib/alertmanager`
- Create systemd unit config `sudo nano /usr/lib/systemd/system/alertmanager.service`
  ```ini 
  [Unit]
  Description=AlertManager
  Wants=network-online.target
  After=network-online.target

  [Service]
  User=alert-manager
  Group=alert-manager
  Type=simple
  ExecStart=/usr/bin/alertmanager \
    --config.file /etc/alertmanager/alertmanager.yml \
    --storage.path /var/lib/alertmanager/ \
    --cluster.advertise-address=0.0.0.0:9093

  [Install]
  WantedBy=multi-user.target
  ```
- Add permission `sudo chmod 664 /usr/lib/systemd/system/alertmanager.service`
- Enable to start on boot `sudo systemctl enable alertmanager.service`
- Reloaad daemon `sudo systemctl daemon-reload`
- Start the service `sudo systemctl start alertmanager`


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
- Load the dashboard and select the Prometheus data source. Click "Import"
