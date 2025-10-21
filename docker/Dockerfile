# Multi-stage build for OpenSIPS
FROM debian:bookworm AS builder

# Install build dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    bison \
    flex \
    m4 \
    libncurses-dev \
    libssl-dev \
    default-libmysqlclient-dev \
    libpq-dev \
    libxml2-dev \
    libexpat1-dev \
    libpcre2-dev \
    libcurl4-openssl-dev \
    && rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /usr/src/opensips

# Copy OpenSIPS source code
COPY . .

# Build OpenSIPS with commonly used modules
# Exclude some modules that have complex dependencies or are rarely used
RUN make all && \
    make install

# Runtime stage - smaller final image
FROM debian:bookworm-slim

ARG OPENSIPS_CLI=false

# Install runtime dependencies only
RUN apt-get update && apt-get install -y \
    libssl3 \
    libmariadb3 \
    libpq5 \
    libxml2 \
    libexpat1 \
    libpcre2-8-0 \
    libcurl4 \
    ca-certificates \
    gettext-base \
    bash \
    findutils \
    python3 \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*

# Install opensips-cli if requested
RUN if [ "$OPENSIPS_CLI" = "true" ]; then \
        apt-get update && \
        apt-get install -y default-libmysqlclient-dev build-essential && \
        pip3 install --no-cache-dir --break-system-packages opensipscli && \
        apt-get remove -y default-libmysqlclient-dev build-essential && \
        apt-get autoremove -y && \
        rm -rf /var/lib/apt/lists/*; \
    fi

# Copy OpenSIPS binaries and configs from builder
COPY --from=builder /usr/local /usr/local/

# Copy custom config as template
COPY docker/config/opensips.cfg /usr/local/etc/opensips/opensips.cfg.template

# Create opensips user
RUN useradd -r -s /bin/false opensips

# Create the directory for MI FIFO
RUN mkdir -p /run/opensips && chmod 777 /run/opensips

# Create entrypoint script that auto-detects module path and substitutes env vars
RUN echo '#!/bin/bash' > /usr/local/bin/opensips-entrypoint.sh && \
    echo 'set -e' >> /usr/local/bin/opensips-entrypoint.sh && \
    echo '' >> /usr/local/bin/opensips-entrypoint.sh && \
    echo '# Auto-detect module path if not set' >> /usr/local/bin/opensips-entrypoint.sh && \
    echo 'if [ -z "$OPENSIPS_MPATH" ]; then' >> /usr/local/bin/opensips-entrypoint.sh && \
    echo '  export OPENSIPS_MPATH=$(dirname $(find /usr/local -name "tm.so" -path "*/opensips/modules/*" 2>/dev/null | head -1))/' >> /usr/local/bin/opensips-entrypoint.sh && \
    echo '  echo "Auto-detected OPENSIPS_MPATH: $OPENSIPS_MPATH"' >> /usr/local/bin/opensips-entrypoint.sh && \
    echo 'fi' >> /usr/local/bin/opensips-entrypoint.sh && \
    echo '' >> /usr/local/bin/opensips-entrypoint.sh && \
    echo '# Generate config from template with env substitution (only specific vars)' >> /usr/local/bin/opensips-entrypoint.sh && \
    echo 'echo "Generating config from template with env vars..."' >> /usr/local/bin/opensips-entrypoint.sh && \
    echo 'envsubst '"'"'$OPENSIPS_MPATH $MI_PORT $B2B_SIP_PORT'"'"' < /usr/local/etc/opensips/opensips.cfg.template > /usr/local/etc/opensips/opensips.cfg' >> /usr/local/bin/opensips-entrypoint.sh && \
    echo '' >> /usr/local/bin/opensips-entrypoint.sh && \
    echo 'exec "$@"' >> /usr/local/bin/opensips-entrypoint.sh && \
    chmod +x /usr/local/bin/opensips-entrypoint.sh

# Create volumes for configuration and logs
VOLUME ["/usr/local/etc/opensips", "/var/log/opensips"]

# Set working directory
WORKDIR /usr/local/etc/opensips

# Run as root (OpenSIPS will drop privileges if configured)
# You can modify opensips.cfg to run as opensips user
# -F = foreground mode, -DD = debug level 2
ENTRYPOINT ["/usr/local/bin/opensips-entrypoint.sh"]
CMD ["opensips", "-F"]
