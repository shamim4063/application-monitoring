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
5. Run ```docker-compose down``` then ``` docker-compose up -d```. Go to browser enter  http://<your-server-host>:3000 will render Grafana. To connect Grafan with prometheus, go to Connection > Data Source and click Add New Datasource. Enter http://prometheus:9090 as Prometheus server URL as Grafana and Prometheus are in same network http://<prometheus-container-name>:9090 should work. Now Click Save & Test  

![image](https://github.com/user-attachments/assets/6ccc1598-38ad-4ba2-8ead-8cc451d39501)

6. Now navigate to Dashboard and Click New > Import. Enter a Grafana Dashboard Id or Url for Node Exporter and select Prometheus as Data Source. In my case I've used 1860 Dashboard id or find from here https://grafana.com/grafana/dashboards/. Dashboard will look like this:

![image](https://github.com/user-attachments/assets/d53aba89-b765-4d78-893e-3b2a102e4d94)


Now you can add cadvisor to scrap your docker containers, mongodb-exporter, sqlserver-exporter to monitor mongoDB and SQL Server or any other database, RabbitMQ, elastic-search etc, 

## API Scraping
To collect metrics of Web APIs you can expose web metrics from you Nodejs, DotNet or from any other framework to monitor APIs performence. In my case I enabled it in DotNet webApi following way:
1. I added a custom middleware first like following:
 ```
namespace WebApi.Middlewares
{
    public class CustomMetricsMiddleware
    {
        private readonly RequestDelegate _next;

        // Updated Histogram with 'controller' and 'action' labels added
        private static readonly Histogram _requestDuration = Metrics
            .CreateHistogram("http_request_duration_seconds",
                             "Duration of HTTP requests in seconds",
                             new HistogramConfiguration
                             {
                                 Buckets = Histogram.ExponentialBuckets(0.1, 2, 10),
                                 LabelNames = new[] { "method", "route", "status_code", "controller", "action" }
                             });

        public CustomMetricsMiddleware(RequestDelegate next)
        {
            _next = next;
        }

        public async Task InvokeAsync(HttpContext context)
        {
            var timer = Stopwatch.StartNew();

            await _next(context);  // Continue processing the request

            timer.Stop();

            var statusCode = context.Response.StatusCode.ToString();
            var method = context.Request.Method;

            // Capture route pattern instead of raw path
            var routeData = context.GetRouteData();
            var route = routeData.Values["page"]?.ToString() ??
                        routeData.Values["controller"]?.ToString() ??
                        routeData.Values["action"]?.ToString() ??
                        context.Request.Path.Value ?? "unknown";

            // Add labels for controller and action
            var controller = routeData.Values["controller"]?.ToString() ?? "unknown";
            var action = routeData.Values["action"]?.ToString() ?? "unknown";

            _requestDuration
                .WithLabels(method, route, statusCode, controller, action)
                .Observe(timer.Elapsed.TotalSeconds);
        }
    }
}
 ```
2. In Startup.cs add following lines

 ```
        // Enable Routing
        app.UseRouting();

        // Add Prometheus HTTP Metrics Middleware (Before Custom Middleware)
        app.UseHttpMetrics();

        // Add Custom Metrics Middleware for Enhanced Metrics
        app.UseMiddleware<CustomMetricsMiddleware>();                    
        app.UseEndpoints(endpoints =>
        {
            endpoints.MapMetrics();
            endpoints.MapControllers();
        });
```

 3. In nodejs you can do in following way
```
 const express = require('express');
 const client = require('prom-client');
 const app = express();
 const register = client.register;
 
 // Default system metrics (CPU, memory, etc.)
 client.collectDefaultMetrics();
 
 // Custom metric example
 const httpRequestCounter = new client.Counter({
   name: 'http_requests_total',
   help: 'Total number of HTTP requests',
   labelNames: ['method', 'route', 'status']
 });
 
 // Middleware to increment counter
 app.use((req, res, next) => {
   res.on('finish', () => {
     httpRequestCounter.labels(req.method, req.path, res.statusCode).inc();
   });
   next();
 });
 
 // Metrics endpoint
 app.get('/metrics', async (req, res) => {
   res.set('Content-Type', register.contentType);
   res.end(await register.metrics());
 });
 
 // Your actual API routes
 app.get('/', (req, res) => {
   res.send('Hello from Node.js app!');
 });
 
 const PORT = process.env.PORT || 3000;
 app.listen(PORT, () => console.log(`App listening on port ${PORT}`));
```

3. Update prometheus.yml for API scraping with:

```
- job_name: 'dotnet-api' # any name 
    static_configs:
      - targets: ['<web-api-container>:<port>'] # Here I am using container's name as host as my web application and prometheus is also in same network. Make sure the IP addess if these are not in same network 
```

## Alertmanager configuration
1. To enabale alert you need to update docker-compose.yml file by adding another service for prometheus alert.
 ```
  alertmanager:
   image: prom/alertmanager
   container_name: alertmanager
   restart: always
   ports:
    - "9093:9093"
   volumes:
    - ./alertmanager.yml://etc/alertmanager/alertmanager.yml
   networks:
    - monitoring
```
2. Create alertmanager.yml to  Routes Alerts to the Right Channels, Groups and Manages Alerts Logically, Adds Control & Customization

```
global:
  resolve_timeout: 5m

route:
  receiver: 'slack-alert'
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 3h

receivers:
  - name: 'slack-alert'
    slack_configs:
      - send_resolved: true
        channel: '#<channel-name>'
        username: 'SystemAlertManager'
        api_url: '<Slack API Url>'
        title: "{{ range .Alerts }}{{ .Annotations.summary }}\n{{ end }}"
        text: "{{ range .Alerts }}{{ .Annotations.description }}\n{{ end }}"
```

3. To Define Alert Rules create alert.rules.yml. We will define conditions under which alerts should be triggered, Set thresholds and durations for when those conditions become actionable.

```
groups:
- name: system_alerts
  rules:
  - alert: HighCPUUsage
    expr: (100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)) > 80
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "High CPU Usage"
      description: "CPU usage is above 80% for more than 2 minutes."

  - alert: HighMemoryUsage
    expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "High Memory Usage"
      description: "Memory usage is above 90% for more than 2 minutes."

  - alert: LowDiskSpace
    expr: (node_filesystem_avail_bytes{device="/dev/root"} / node_filesystem_size_bytes{device="/dev/root"}) * 100 < 10 # I define this expression based on my file system(AWS ubuntu). Make sure your directory structure
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Low Disk Space"
      description: "Less than 10% disk space available."

```

## Finally 
To see my full configuration see the source folder. Update the host, port, rules etc based on your case and just run `docker-compose up -d`. Then add different dashboards in Grafana for cadvisor(Dashboard Id: 193) , SQL Server(Dashboard ID: 13919)





