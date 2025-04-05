# Prometheus Overview
Prometheus collects and stores its metrics as time series data, metrics information is stored with the timestamp at which it was recorded, alongside optional key-value pairs called labels.
## Grafana 
Grafana enables us to query, visualize, alert on, and explore metrics, logs, and traces wherever they’re stored. Grafana data source plugins enable us to query data sources including time series databases like Prometheus and CloudWatch. Grafana provides us with tools to display that data on live dashboards with insightful graphs and visualizations.

## Prometheus Server: 
![architecture](https://github.com/user-attachments/assets/32a22eb8-0402-4b4b-b2a5-94217f152b9e)

The Prometheus server is the core of the system, responsible for collecting, storing, and exposing metrics. It works on a pull model, meaning it periodically pulls (scrapes) metrics from configured targets over HTTP. Internally, the Prometheus server has three main parts:
- Retrieval (Data Collector): This component continuously scrapes metrics from target endpoints using HTTP requests at the defined scrape interval. It is in charge of fetching data from all discovered or configured services.

- Time-Series Database (TSDB): All scraped metric samples are stored locally in Prometheus’s embedded TSDB as time-series data. The TSDB is optimized for efficient storage and querying of timestamped metrics, using data compression and indexing for performance. By default, data is stored on local disk (HDD/SSD) and kept for a configurable retention period (e.g. weeks) before being pruned.

- HTTP Server (API & UI): Prometheus’s built-in HTTP server exposes a RESTful API and a web-based UI. This allows users and tools to query the stored metrics using PromQL (Prometheus’s query language) and to visualize results. The HTTP server provides an expression browser (basic web UI for ad-hoc queries and graphs) and serves as an endpoint for external systems (like Grafana or API clients) to fetch data. In essence, the Prometheus server not only collects and stores metrics but also makes them available for querying and analysis via its HTTP endpoints.

## Installation:
Initially, I will deploy Prometheus using Docker and use Node Exporter to collect and expose key server-level metrics. These include CPU usage, memory consumption, disk space utilization, and file system statistics. In addition, Node Exporter provides valuable insights into system load averages, network I/O, disk I/O, process counts, context switches, and system uptime. This foundational setup will allow me to gain visibility into the health and performance of the infrastructure, helping detect issues early and ensure smooth application operations.

1. create a docker-compose.yml file with followig configuration. Here I am using prometheus offcial docker image prom/prometheus and node-exporter to collect OS and hardware-level metrics like CPU usage, Memory usage, Disk I/O, File system space, Network throughput
Load averages, System uptime, Number of running processes, etc. Make sure both containers are in same network.

 ```
version: '3.8'
services:
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    restart: always
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter
    container_name: node-exporter
    restart: always
    ports:
      - "9100:9100"
    networks:
      - monitoring
```
2. Create file prometheus.yml. This is the main configuration file for how Prometheus operates. We will define here  What to Monitor, How Often to Scrape, Sets Up Monitoring Jobs, Foundation for Alerts etc.

```
# Global configuration
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.

# Scrape configuration.
# Here it's Prometheus itself.
scrape_configs:
  
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
```
3. Run ```docker-compose up -d``` in the directory where you have created the files. After opening the prometheus port(9090) in firewall or adding in the security group enter http://<your-server-host>:9090 in your browser. You will see the prometheus UI like following with promQL field
![image](https://github.com/user-attachments/assets/b03c22f6-6b2a-47a9-a03f-2e59dced9d8b)

To see if prometheus connected to node-exporter and scrapping metrcis, navigate to the Status and Targets. It will be shwon like following:

![image](https://github.com/user-attachments/assets/f5867cb9-894d-4595-a61a-137606e4832c)

4. Install Grafana to to connect to Prometheus for better visualization of the metrics of node exporter. Add following configuration in docker-compose.yml

```
  grafana:
    image: grafana/grafana
    container_name: grafana
    restart: always
    ports:
      - "3000:3000"
    volumes:
      - grafana-storage:/var/lib/grafana
    networks:
      - monitoring
```  




