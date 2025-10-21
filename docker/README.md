# OpenSIPS Docker Setup

This Docker setup provides a containerized OpenSIPS instance with environment variable support and no volume mounts required.

## Quick Start

1. **Configure environment variables:**
   ```bash
   cd docker
   cp config/.env.example config/.env
   # Edit config/.env with your values
   ```

2. **Build and run:**
   ```bash
   docker compose up -d --build
   ```

3. **View logs:**
   ```bash
   docker compose logs -f opensips
   ```

## Configuration

### Environment Variables

The OpenSIPS configuration uses environment variables that can be set in `config/.env`:

```bash
MI_PORT=8080           # Management Interface port
B2B_SIP_PORT=5060      # SIP listening port
```

### How It Works

1. **Config Template**: Your `opensips.cfg` uses `${VARIABLE}` syntax for placeholders
2. **Entrypoint Magic**: At container startup, `envsubst` replaces only specified variables
3. **No Preprocessor Needed**: No `substenv.sh` or volume mounts required
4. **Immutable Config**: The config template is baked into the Docker image

### Modifying the Configuration

Edit `config/opensips.cfg` and rebuild:

```bash
docker compose up -d --build
```

**Important**: Only `${OPENSIPS_MPATH}`, `${MI_PORT}`, and `${B2B_SIP_PORT}` are substituted. OpenSIPS script variables like `$var(name)` are preserved.

### Adding New Environment Variables

1. Add the variable to `config/.env`:
   ```bash
   MY_NEW_VAR=value
   ```

2. Use it in `config/opensips.cfg`:
   ```
   socket=udp:${MY_NEW_VAR}:5060
   ```

3. Update the Dockerfile to include it in the `envsubst` command (line ~87):
   ```dockerfile
   envsubst '$OPENSIPS_MPATH $MI_PORT $B2B_SIP_PORT $MY_NEW_VAR' < ...
   ```

## Networking

The docker-compose creates a network called `opensips-voip` that other containers can join.

### Connecting from Another Docker Compose

```yaml
services:
  your-service:
    # ... your service config ...
    networks:
      - opensips-voip

networks:
  opensips-voip:
    external: true
```

## Management

### Access OpenSIPS CLI

```bash
docker compose exec opensips opensips-cli
```

### Check Status

```bash
docker compose exec opensips opensips-cli -x mi sr_get_status
```

### Restart OpenSIPS

```bash
docker compose restart opensips
```

## Ports

- **5060/udp, 5060/tcp**: SIP signaling (exposed to host)
- **8080/udp**: Management Interface (configurable via `MI_PORT`)

## Troubleshooting

### Check Generated Config

```bash
docker compose exec opensips cat /usr/local/etc/opensips/opensips.cfg
```

### View Environment Variables

```bash
docker compose exec opensips env | grep -E "(MI_PORT|B2B_SIP_PORT|OPENSIPS_MPATH)"
```

### Config Parse Errors

If you see syntax errors, ensure your OpenSIPS script variables use `$var(name)` syntax, not `${name}` which would be replaced by envsubst.

### Verify Listening Ports

```bash
docker compose logs opensips | grep -i "listening"
```

## Build Arguments

- `OPENSIPS_CLI=true`: Installs opensips-cli tool (default: false in Dockerfile, true in docker-compose)

## Development vs Production

**Development** (current setup):
- Config baked into image
- Environment variables for customization
- Fast rebuilds

**Production**:
- Use multi-stage builds (already configured)
- Set specific versions via build args
- Use secrets management for sensitive data

## Files Structure

```
docker/
├── Dockerfile              # Multi-stage OpenSIPS build
├── docker-compose.yml      # Service definition
├── README.md              # This file
└── config/
    ├── .env               # Environment variables (gitignored)
    ├── .env.example       # Example environment variables
    └── opensips.cfg       # OpenSIPS configuration template
```

## How to Override Environment Variables

You can override environment variables in multiple ways:

1. **Via .env file** (recommended):
   ```bash
   echo "B2B_SIP_PORT=5070" >> config/.env
   docker compose up -d
   ```

2. **Via docker-compose.yml**:
   ```yaml
   environment:
     - B2B_SIP_PORT=5070
   ```

3. **Via command line**:
   ```bash
   B2B_SIP_PORT=5070 docker compose up -d
   ```
