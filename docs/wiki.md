# ELK Pipeline Task Wiki

## Overview
This repository defines a self-contained ELK (Elasticsearch, Logstash, Kibana) stack tailored for ingesting newline-delimited JSON (NDJSON) log files from a host directory, forwarding them through Filebeat and Logstash, and finally indexing the events into Elasticsearch while optionally archiving the raw messages. The stack is orchestrated with Docker Compose and designed for local experimentation or lightweight production usage.

## Architecture
```
[input JSON files] → Filebeat → Logstash → Elasticsearch ← Kibana
                                     ↘
                                      ↳ Compressed file archive
```

### Components
- **Filebeat**: Reads NDJSON log files mounted from the host, parses each line as JSON, and forwards events to Logstash over the Beats protocol. It handles file state tracking and renames processed files.
- **Logstash**: Accepts Beats input on port 5044, optionally enriches events via filters, outputs to Elasticsearch for indexing, and writes a compressed archive of messages back to the host.
- **Elasticsearch**: Stores indexed log documents. Configured in single-node mode with security enabled.
- **Kibana**: Provides a UI for visualizing and querying the indexed data.
- **Volumes**:
  - `esdata`: Docker volume storing Elasticsearch data.
  - `./data`: Host directory mounted into Filebeat for input files (read-only).
  - `./archive`: Host directory mounted into Logstash for compressed log archives (read-write).

## Repository Layout
| Path | Purpose |
| ---- | ------- |
| `docker-compose.yml` | Declares services, environment variables, and volume mappings for the ELK stack. |
| `config/logstash.conf` | Logstash pipeline definition with Beats input, optional filters, Elasticsearch output, and file archive output. |
| `config/filebeat.yml` | Filebeat configuration for reading NDJSON files and sending them to Logstash. |
| `data/` *(expected)* | Place NDJSON log files here; mounted read-only into Filebeat as `/input-data`. |
| `archive/` *(created at runtime)* | Destination for compressed archives produced by Logstash. |
| `docs/wiki.md` | This documentation. |

## Prerequisites
- Docker and Docker Compose installed.
- Environment variables provided (see below).
- Host directories `data/` and `archive/` created with appropriate permissions (Filebeat requires read access; Logstash requires read/write).

## Configuration

### Environment Variables
The Compose file references the following variables:

| Variable | Description | Example |
| -------- | ----------- | ------- |
| `ELASTIC_VERSION` | Elastic stack version tag (e.g., `8.10.4`). All services use the same value. |
| `ELASTIC_PASSWORD` | Password for the built-in `elastic` superuser. Required for Elasticsearch, Logstash, and Kibana to authenticate. |
| `KIBANA_PASSWORD` | Password for the `kibana_system` user that Kibana uses to authenticate with Elasticsearch. |

Define these in a `.env` file alongside `docker-compose.yml`, or export them in the shell before running Compose.

### Filebeat Input
`config/filebeat.yml` configures a `filestream` input that:
- Watches `/input-data/*.json` for NDJSON files.
- Parses each line as JSON (`ndjson` parser) and promotes fields to the top level.
- Renames processed files by appending `.closed` (via `close.on_state_change`).
- Flushes the registry every 30 seconds for quicker testing cycles.

Adjust `paths`, input type, or parser settings to suit other log formats.

### Logstash Pipeline
`config/logstash.conf` contains:
- **Input**: Beats listener on port 5044.
- **Filter**: Placeholder for enrichment logic (e.g., `date` filter to convert timestamps). Add filter plugins inside the `filter {}` block.
- **Outputs**:
  - `elasticsearch`: Sends events to `http://elasticsearch:9200` using the `elastic` user. Indexed into `app-logs-%{+yyyy.MM.dd}` daily indices.
  - `file`: Writes raw messages to `/output-archive/%{+yyyy-MM-dd}.log.gz` with gzip compression. Customizable format via the `codec`.

### Security Considerations
- Elasticsearch security is enabled (`xpack.security.enabled=true`); authenticate using the `elastic` user or create additional users/roles as needed.
- Store credentials in a secrets manager or Compose `.env` file with restricted permissions.
- Restrict exposed ports (`9200`, `5601`, `5044`) to trusted networks when deploying beyond local testing.

## Usage

1. **Prepare directories:**
   ```bash
   mkdir -p data archive
   chmod 755 data archive
   ```
2. **Create `.env` file:**
   ```bash
   cat <<EOF > .env
   ELASTIC_VERSION=8.10.4
   ELASTIC_PASSWORD=YourElasticPassword
   KIBANA_PASSWORD=YourKibanaPassword
   EOF
   ```
3. **Add NDJSON logs:** Place files like `data/app-log-1.json` containing one JSON object per line.
4. **Start the stack:**
   ```bash
   docker compose up -d
   ```
5. **Verify services:**
   - Elasticsearch: `curl -u elastic:$ELASTIC_PASSWORD http://localhost:9200`
   - Kibana UI: `http://localhost:5601`
   - Logstash Beats port: `telnet localhost 5044`
6. **Check archives:** Compressed log files should appear in `archive/` with daily filenames.

Use `docker compose logs <service>` to inspect service logs.

### Stopping and Cleaning Up
```bash
docker compose down           # stop services
docker compose down -v        # additionally remove volumes (including Elasticsearch data)
```

## Indexing Workflow
1. **Filebeat** picks up new `.json` files, parses each line, and forwards events.
2. **Logstash** receives events, optionally enriches them, writes to Elasticsearch, and archives raw messages.
3. **Elasticsearch** indexes documents into time-based indices.
4. **Kibana** accesses the indices for visualization and search.

## Customization Guide
- **Different Input Formats:** Modify `config/filebeat.yml` to use other input types (`log`, `tcp`, etc.) or alternate parsers (`json`, `multiline`).
- **Advanced Filtering:** Extend `filter {}` in Logstash with `grok`, `mutate`, `geoip`, etc., to normalize and enrich data.
- **Index Naming:** Adjust the `index` setting in the Elasticsearch output for alternate naming conventions or data streams.
- **Archiving Strategy:** Change the `file` output path, codec, or compression behavior to align with retention requirements.
- **Resource Tuning:** Override service-level JVM options, memory limits, or CPU quotas in `docker-compose.yml` as needed.

## Monitoring and Maintenance
- Monitor container health via `docker compose ps` and `docker compose logs`.
- Use Kibana Stack Monitoring (requires proper licensing) to observe Elasticsearch and Logstash performance.
- Rotate archives and manage disk usage in `archive/`.
- Backup the `esdata` volume regularly for disaster recovery.

## Troubleshooting
| Symptom | Possible Cause | Resolution |
| ------- | -------------- | ---------- |
| Filebeat logs permission errors | Host `data/` directory unreadable by container | Ensure directory and files are world-readable or adjust user mapping. |
| Logstash fails to connect to Elasticsearch | Wrong `ELASTIC_PASSWORD` or Elasticsearch unavailable | Verify credentials and service status. |
| Kibana shows authentication prompts | `KIBANA_PASSWORD` mismatch | Reset `kibana_system` password in Elasticsearch and update `.env`. |
| No indices appear in Elasticsearch | Filebeat not sending events or Logstash pipeline misconfigured | Check Filebeat logs, confirm files match `paths`, and inspect Logstash logs for pipeline errors. |
| Archive files empty | `message` field missing or codec misconfigured | Ensure events contain `message` field or modify the codec format. |

## Extending the Stack
- Add **Ingest Pipelines** in Elasticsearch for server-side enrichment.
- Integrate **Metricbeat** or **Heartbeat** for infrastructure monitoring.
- Deploy the stack with **Docker Swarm** or **Kubernetes** for scalability.
- Secure the stack with **TLS certificates** and **reverse proxies** (e.g., Traefik, Nginx).

## References
- [Elastic Stack Documentation](https://www.elastic.co/guide/index.html)
- [Filebeat Modules & Inputs](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-inputs.html)
- [Logstash Configuration](https://www.elastic.co/guide/en/logstash/current/configuration.html)
- [Docker Compose Reference](https://docs.docker.com/compose/)

