# Belvedere Glacier Website Infrastructure and Usage Guide

## Overview

This document describes the current working setup of the Belvedere Glacier website infrastructure, including the production and development environments, the server architecture, the deployment workflow, and the use of Hetzner Object Storage for Potree point clouds.

The setup is designed to keep production stable while allowing safe testing in a separate development environment. The website is served through Caddy as a reverse proxy, the application runs in Docker containers, and large Potree point cloud assets are stored in a dedicated Hetzner Object Storage bucket and accessed over S3-compatible HTTP requests.[cite:783][cite:483][cite:374]

## Server Architecture

### High-level components

The current server runs the following main services:

- **Caddy**: public reverse proxy and TLS termination for all exposed subdomains.[cite:483]
- **Production website container**: serves `https://thebelvedereglacier.it`.
- **Development website container**: serves `https://dev.thebelvedereglacier.it`.
- **SFTPGo**: handles file transfer and web access on `https://files.thebelvedereglacier.it`.
- **PostgreSQL/PostGIS**: provides the database backend for the application.
- **Hetzner Object Storage**: stores large static Potree point cloud assets in an S3-compatible bucket.[cite:374][cite:178]

### Network layout

Two external Docker networks are used:

- `public-net`: shared between Caddy and the web-facing services so Caddy can reverse-proxy requests to the containers.
- `infra-net`: shared between the application containers and backend services such as PostgreSQL and SFTPGo.

This separation keeps public routing and internal service communication logically distinct while still allowing the required containers to communicate.

### Public endpoints

The currently active endpoints are:

- `https://thebelvedereglacier.it` → production website
- `https://dev.thebelvedereglacier.it` → development website
- `https://files.thebelvedereglacier.it` → SFTPGo web client, with redirect from `/` to `/web/client`

Caddy automatically manages TLS certificates for these domains and proxies traffic to the corresponding containers.[cite:483]

## Docker Environments

### Production environment

The production environment uses `docker-compose.yml` and runs the main container:

- container name: `belvedereapp`
- internal port: `80`
- networks: `public-net`, `infra-net`

The production stack is intended to be stable. It should only receive tested changes merged into the `main` branch.

#### Main production commands

Start or refresh production:

```bash
docker compose up -d --build
```

Stop production:

```bash
docker compose down
```

Inspect logs:

```bash
docker logs belvedereapp --tail 100
```

### Development environment

The development environment uses `docker-compose.dev.yml` and runs a separate container:

- project name: `thebelvedereglacier-dev`
- container name: `belvedereapp-dev`
- image: `thebelvedereglacier-webserver-dev:latest`
- internal port: `80`
- networks: `public-net`, `infra-net`

It is isolated from production and is intended for testing changes before release.

#### Recommended `docker-compose.dev.yml`

```yaml
name: thebelvedereglacier-dev

services:
  webserver:
    container_name: belvedereapp-dev
    build:
      context: .
      dockerfile: Dockerfile
    image: thebelvedereglacier-webserver-dev:latest
    volumes:
      - ./app:/var/www/html
    environment:
      - DBHOST=${DBHOST}
      - DBNAME=${DBNAME}
      - DBUSERNAME=${DBUSERNAME}
      - DBPORT=${DBPORT}
      - DBSSLMODE=${DBSSLMODE}
      - DBPASSWORD=${DBPASSWORD}
    networks:
      - public-net
      - infra-net

networks:
  public-net:
    external: true
  infra-net:
    external: true
```

#### Main development commands

Start or refresh development:

```bash
docker compose -f docker-compose.dev.yml --env-file .env up -d --build
```

Stop development:

```bash
docker compose -f docker-compose.dev.yml down
```

Inspect logs:

```bash
docker logs belvedereapp-dev --tail 100
```

Check both containers:

```bash
docker ps | grep belvedere
```

## Reverse Proxy Configuration

Caddy routes each hostname to the correct internal container. A working `Caddyfile` layout is:

```caddy
# Production website
thebelvedereglacier.it, www.thebelvedereglacier.it {
    reverse_proxy belvedereapp:80
}

# Development website
dev.thebelvedereglacier.it {
    reverse_proxy belvedereapp-dev:80
}

# SFTPGo web client
files.thebelvedereglacier.it {
    redir / /web/client 302
    reverse_proxy sftpgo:8080
}
```

Reload after changes:

```bash
docker exec caddy caddy reload --config /etc/caddy/Caddyfile
```

Caddy handles HTTPS certificates automatically once the DNS records point to the server and port 80/443 are reachable from the internet.[cite:483]

## Development and Release Workflow

### Suggested branch strategy

Use two long-lived branches:

- `dev` → automatically deploys to `dev.thebelvedereglacier.it`
- `main` → automatically deploys to `thebelvedereglacier.it`

Typical workflow:

1. Make changes locally.
2. Commit and push to `dev`.
3. Test on `https://dev.thebelvedereglacier.it`.
4. Merge `dev` into `main` only after validation.
5. Push `main` to release to production.

This aligns with normal Docker Compose and environment separation practices.[cite:778][cite:783]

### GitHub Actions

Recommended setup:

- `deploy-dev.yml` runs only on pushes to `dev`
- `deploy-prod.yml` runs only on pushes to `main`

This ensures development and production deployments remain separate.

## S3 / Hetzner Object Storage Layout

### Bucket purpose

A dedicated bucket named `belvedere-website` stores website-specific static assets, especially Potree point clouds.

### Current object structure

The point clouds are stored with this prefix structure:

```text
belvedere-website/
└── potree/
    └── pointclouds/
        ├── background/
        ├── 1977/
        ├── 1991/
        ├── 2001/
        ├── 2009/
        ├── 2015/
        ├── 2016/
        ├── 2017/
        ├── 2018/
        ├── 2019/
        ├── 2020/
        ├── 2021/
        ├── 2022/
        └── 2023/
```

Each dataset folder contains files such as:

- `metadata.json`
- `hierarchy.bin`
- `octree.bin`
- `log.txt`

In object storage these are not real folders, but object key prefixes.[cite:178]

## Accessing the Bucket

### Configure AWS CLI for Hetzner Object Storage

Hetzner Object Storage is S3-compatible and can be accessed with standard S3 tools such as AWS CLI.[cite:374]

Example configuration values:

```bash
aws configure
```

Prompts:

```text
AWS Access Key ID: <your hetzner access key>
AWS Secret Access Key: <your hetzner secret key>
Default region name: eu-central
Default output format: json
```

The actual endpoint is specified per command using `--endpoint-url`, for example:

```bash
--endpoint-url https://nbg1.your-objectstorage.com
```

### Upload point clouds

Upload the local Potree assets while preserving the desired prefix structure:

```bash
aws s3 sync ./assets/pointclouds/ s3://belvedere-website/potree/pointclouds/ \
  --endpoint-url https://nbg1.your-objectstorage.com
```

### List uploaded objects

```bash
aws s3 ls s3://belvedere-website/potree/pointclouds/ \
  --recursive \
  --endpoint-url https://nbg1.your-objectstorage.com
```

### Download data locally

Download a single dataset:

```bash
aws s3 sync s3://belvedere-website/potree/pointclouds/2023 ./tmp/2023 \
  --endpoint-url https://nbg1.your-objectstorage.com
```

Download everything:

```bash
aws s3 sync s3://belvedere-website/potree/pointclouds/ ./tmp/pointclouds \
  --endpoint-url https://nbg1.your-objectstorage.com
```

### Update or replace a dataset

After modifying a point cloud locally, re-sync just that folder:

```bash
aws s3 sync ./assets/pointclouds/background/ s3://belvedere-website/potree/pointclouds/background/ \
  --delete \
  --endpoint-url https://nbg1.your-objectstorage.com
```

The `--delete` flag removes remote files that no longer exist locally, keeping the dataset in sync.

## Using the S3 Assets in Potree

### Viewer configuration

The website can load Potree clouds directly from Object Storage by using the bucket URL prefix instead of the local `assets/pointclouds` directory.

Example pattern in `viewer.js`:

```js
const S3_BASE = "https://belvedere-website.nbg1.your-objectstorage.com/potree/pointclouds";

const pointCloudURLs = [
  { url: `${S3_BASE}/1977/metadata.json`, name: "1977" },
  { url: `${S3_BASE}/1991/metadata.json`, name: "1991" },
  { url: `${S3_BASE}/2001/metadata.json`, name: "2001" },
  { url: `${S3_BASE}/2009/metadata.json`, name: "2009" },
  { url: `${S3_BASE}/2015/metadata.json`, name: "2015" },
  { url: `${S3_BASE}/2016/metadata.json`, name: "2016" },
  { url: `${S3_BASE}/2017/metadata.json`, name: "2017" },
  { url: `${S3_BASE}/2018/metadata.json`, name: "2018" },
  { url: `${S3_BASE}/2019/metadata.json`, name: "2019" },
  { url: `${S3_BASE}/2020/metadata.json`, name: "2020" },
  { url: `${S3_BASE}/2021/metadata.json`, name: "2021" },
  { url: `${S3_BASE}/2022/metadata.json`, name: "2022" },
  { url: `${S3_BASE}/2023/metadata.json`, name: "2023", visible: true },
];

loadPointCloud(`${S3_BASE}/background/metadata.json`, "Background", true);
```

Potree loads `metadata.json` first, then fetches `hierarchy.bin`, `octree.bin`, and related binary chunks from the same dataset directory. This is why the directory structure must remain intact.[cite:178]

### Browser access requirements

If the website fetches point clouds directly from the bucket, the bucket must allow **CORS** for the website origins. Hetzner documents CORS support for Object Storage buckets.[cite:723]

Example `cors.json`:

```json
{
  "CORSRules": [
    {
      "AllowedOrigins": [
        "https://dev.thebelvedereglacier.it",
        "https://thebelvedereglacier.it"
      ],
      "AllowedMethods": ["GET", "HEAD"],
      "AllowedHeaders": ["*"],
      "ExposeHeaders": ["ETag", "Accept-Ranges", "Content-Length", "Content-Range"],
      "MaxAgeSeconds": 3600
    }
  ]
}
```

Apply it with:

```bash
aws s3api put-bucket-cors \
  --bucket belvedere-website \
  --cors-configuration file://cors.json \
  --endpoint-url https://nbg1.your-objectstorage.com
```

### Verify direct bucket access

Check whether a Potree metadata file is reachable:

```bash
curl -I \
  -H "Origin: https://dev.thebelvedereglacier.it" \
  https://belvedere-website.nbg1.your-objectstorage.com/potree/pointclouds/2023/metadata.json
```

Expected behavior:

- HTTP success response
- `Access-Control-Allow-Origin` header present

Check range request behavior for binary files:

```bash
curl -I \
  -H "Range: bytes=0-1023" \
  https://belvedere-website.nbg1.your-objectstorage.com/potree/pointclouds/2023/octree.bin
```

This helps validate that Potree can request binary data correctly.

## Access from Other Online Services

Any online service that can read S3-compatible objects over HTTPS can use the bucket if:

- it has credentials for bucket access through the S3 API, or
- the needed objects are readable over HTTP, or
- your application acts as a proxy.

Typical access patterns:

- **AWS CLI / rclone / scripts**: use S3 credentials and the Hetzner endpoint.[cite:374]
- **Potree in browser**: direct HTTPS access to `metadata.json` and related binaries, with CORS enabled.
- **Other web apps**: same as Potree, provided they can fetch the objects over HTTPS and the bucket permits it.

If the bucket is made private in the future, browser-based applications will need a proxy or signed URL workflow instead of direct object URLs.

## Recommended Usage Rules

### Production rules

- Only deploy tested code from `main`.
- Do not experiment directly in the production container.
- Keep production point cloud paths stable.
- Re-upload assets only after testing them in dev.

### Development rules

- Test all Potree path changes in `dev` first.
- Keep a working local copy of the point cloud datasets before replacing objects in the bucket.
- When debugging a single broken cloud, sync only that folder rather than the entire bucket.
- Use browser devtools network tab to verify whether files are loaded from local paths or from S3.

## Troubleshooting

### Dev or production site not reachable

Check:

```bash
docker ps
cat /opt/services/caddy/Caddyfile
docker exec caddy caddy reload --config /etc/caddy/Caddyfile
```

### DNS points to wrong IP

Use:

```bash
dig thebelvedereglacier.it +short
dig dev.thebelvedereglacier.it +short
```

### CORS errors in browser

This usually means the bucket CORS policy is missing or incorrect. Re-apply the bucket CORS configuration and test with `curl -I` including an `Origin` header.[cite:723]

### Potree loads some clouds but not others

This usually means one of the following:

- wrong path in `viewer.js`
- broken or incomplete upload for a dataset
- Potree conversion mismatch for that specific dataset
- metadata and octree binary mismatch

Compare a broken dataset folder against a working one and re-sync only the affected dataset.

## Suggested Next Improvements

The current direct-bucket setup is good for testing and operational use, but it still exposes bucket object URLs to the browser. A future hardening step would be to keep the bucket private and expose Potree data through signed URLs or an application proxy. Hetzner Object Storage supports standard S3-compatible tooling for such workflows.[cite:374][cite:178]
