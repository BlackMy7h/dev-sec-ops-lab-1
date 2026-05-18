# ─── DevSecOps Lab - Session 3 ────────────────────────────────────────────────
# Production-hardened Dockerfile for Flask application
# Follows CIS Docker Benchmark recommendations
# Uses Alpine Linux for minimal attack surface and reduced CVE footprint
# ──────────────────────────────────────────────────────────────────────────────

# Stage 1: Build dependencies
FROM python:3.11-alpine AS builder

WORKDIR /build

# Security: Update base image packages
RUN apk update && apk upgrade && rm -rf /var/cache/apk/*

# Install only build-time dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt && \
    pip uninstall -y wheel && \
    pip install --no-cache-dir --user wheel>=0.46.2


# Stage 2: Production image (minimal attack surface)
FROM python:3.11-alpine AS production

# Security: Update base image packages
RUN apk update && apk upgrade && rm -rf /var/cache/apk/*

# Security: Install shadow utilities for user management and dumb-init
RUN apk add --no-cache shadow dumb-init

# Security: Upgrade wheel and setuptools to remove CVE-2026-24049
RUN pip install --upgrade wheel>=0.46.2 setuptools

# Security: Create non-root user
RUN groupadd -r appgroup && useradd -r -g appgroup -d /app -s /sbin/nologin appuser

WORKDIR /app

# Copy only installed packages from builder (not build tools)
COPY --from=builder /root/.local /home/appuser/.local

# Copy application code
COPY --chown=appuser:appgroup part1/app_secure.py .
COPY --chown=appuser:appgroup part1/templates/ ./templates/

# Security: No writable filesystem in production
# Only /tmp and specific dirs are writable
RUN mkdir -p /app/data && chown appuser:appgroup /app/data

# Switch to non-root user
USER appuser

# PATH for user-installed packages
ENV PATH=/home/appuser/.local/bin:$PATH

# Expose on high port (not privileged)
EXPOSE 8000

# Security: Use dumb-init as PID 1 (handles signals correctly)
# Use gunicorn (production WSGI server, not Flask dev server)
ENTRYPOINT ["dumb-init", "--"]
CMD ["gunicorn", \
     "--bind", "0.0.0.0:8000", \
     "--workers", "2", \
     "--timeout", "30", \
     "--access-logfile", "-", \
     "--error-logfile", "-", \
     "--log-level", "info", \
     "app_secure:app"]

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/')" || exit 1
