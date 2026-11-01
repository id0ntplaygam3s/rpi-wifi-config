FROM python:3.11-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    dbus-bin \
    libdbus-1-dev \
    libglib2.0-dev \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# dbus-python's build system looks for dbus-run-session during test config.
# Provide a minimal stub so the build can proceed (tests are skipped in pip).
RUN printf '#!/bin/sh\nexec "$@"\n' > /usr/local/bin/dbus-run-session \
    && chmod +x /usr/local/bin/dbus-run-session

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

RUN useradd -u 1000 -m appuser
USER appuser

COPY --chown=appuser:appuser . .

CMD ["python", "app.py"]
