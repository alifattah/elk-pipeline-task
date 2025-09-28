# ELK Pipeline Task

This repository provides a ready-to-run ELK (Elasticsearch, Logstash, Kibana) stack for replaying newline-delimited JSON (NDJSON) log files. Docker Compose ties the services together so you can ingest sample data, browse it in Kibana, and optionally archive the raw messages with minimal setup.

## What you get
- **Filebeat** – reads `data/*.json` files from the host, treats each line as JSON, and ships events to Logstash.
- **Logstash** – listens for Beats traffic, lets you add filters, indexes events into Elasticsearch, and writes a gzip archive of the original messages to `archive/`.
- **Elasticsearch** – stores the indexed documents in a single-node cluster with security enabled.
- **Kibana** – lets you explore and visualize the indexed data through the browser.

Configuration for Filebeat and Logstash lives in `config/`. The overall stack is declared in `docker-compose.yml`. Extra background information is available in `docs/wiki.md`.

## Prerequisites
- Docker and Docker Compose installed.
- An `.env` file or exported environment variables that define:
  - `ELASTIC_VERSION` – Elastic Stack version tag (e.g. `8.10.4`).
  - `ELASTIC_PASSWORD` – password for the built-in `elastic` user.
  - `KIBANA_PASSWORD` – password for Kibana's `kibana_system` service user.
- Host directories for the pipeline to read from and write to:
  ```bash
  mkdir -p data archive
  chmod 755 data archive
  ```

## Quick start
1. **Add sample data** – copy one or more NDJSON files into `data/`. Each line must be a valid JSON object.
2. **Launch the stack** – run:
   ```bash
   docker compose up -d
   ```
3. **Wait for services** – give Elasticsearch, Logstash, and Kibana a minute to start. Check their logs if you run into issues:
   ```bash
   docker compose logs -f elasticsearch
   docker compose logs -f logstash
   docker compose logs -f kibana
   ```
4. **Explore the data** – open [http://localhost:5601](http://localhost:5601) in your browser, log in with the `elastic` user, and create an index pattern matching `app-logs-*` to view documents.
5. **Review archives** – gzipped copies of the raw log lines will be written to `archive/` on the host.

## Useful commands
- `docker compose ps` – show container status.
- `docker compose down` – stop the stack (add `-v` to clear the Elasticsearch volume).
- `curl -u elastic:$ELASTIC_PASSWORD http://localhost:9200` – verify Elasticsearch is reachable.

## Customize the pipeline
- Adjust `config/filebeat.yml` to monitor different paths or change the parser.
- Extend the `filter {}` section in `config/logstash.conf` with `grok`, `mutate`, or other plugins to enrich events.
- Tweak the Elasticsearch output index name or add more outputs as needed.

For a deeper dive into the design and advanced tweaks, see the detailed notes in `docs/wiki.md`.
