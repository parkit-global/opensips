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
    && rm -rf /var/lib/apt/lists/*

# Copy OpenSIPS binaries and configs from builder
COPY --from=builder /usr/local /usr/local/

# Create opensips user
RUN useradd -r -s /bin/false opensips

# Enable stderr logging for Docker
RUN sed -i 's/stderror_enabled=no/stderror_enabled=yes/' /usr/local/etc/opensips/opensips.cfg

# Create the directory for MI FIFO
RUN mkdir -p /run/opensips && chmod 777 /run/opensips

# Expose SIP ports
# 5060 - SIP UDP/TCP
# 5061 - SIP TLS
EXPOSE 5060/udp 5060/tcp 5061/tcp

# Create volumes for configuration and logs
VOLUME ["/usr/local/etc/opensips", "/var/log/opensips"]

# Set working directory
WORKDIR /usr/local/etc/opensips

# Run as root (OpenSIPS will drop privileges if configured)
# You can modify opensips.cfg to run as opensips user
# -F = foreground mode, -DD = debug level 2
CMD ["opensips", "-F"]
