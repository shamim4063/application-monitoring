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
