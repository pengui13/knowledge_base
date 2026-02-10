# Django PostgreSQL Connection Pooling

Every Django request needs a database connection. Each connection takes time: TCP handshake, SSL, authentication, memory allocation. On a busy server, this overhead adds up fast.

## The Cost of Database Connections

```python
import time
import psycopg2

start = time.perf_counter()
conn = psycopg2.connect(
    host='localhost',
    dbname='myapp',
    user='postgres',
    password='password',
)
elapsed = time.perf_counter() - start
print(f'Connection time: {elapsed * 1000:.2f}ms')
# Typical output: 5-50ms locally, 50-200ms over network
```

Key constraints:
- PostgreSQL has a maximum connection limit (default: 100)
- Each connection consumes memory (~10 MB)
- With multiple Django workers, you hit the limit fast

## Django 5.1+ Native Connection Pooling

Django 5.1 added built-in connection pooling using psycopg3's connection pool.

```bash
pip install "psycopg[binary,pool]"
```

> **Note:** This requires psycopg3 (the `psycopg` package), not psycopg2.

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': config('DB_NAME'),
        'USER': config('DB_USER'),
        'PASSWORD': config('DB_PASSWORD'),
        'HOST': config('DB_HOST'),
        'PORT': config('DB_PORT', default='5432'),
        'OPTIONS': {
            'pool': True,  # Enable connection pooling
        },
    }
}
```

## Before Django 5.1: CONN_MAX_AGE

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        # ... connection details
        'CONN_MAX_AGE': 60,  # Keep connections open for 60 seconds
    }
}
```

## PgBouncer: Infrastructure-Level Pooling

For high-scale deployments, PgBouncer provides connection pooling at the infrastructure level.

```
[Django Workers] → [PgBouncer] → [PostgreSQL]
     Many              Few            One
   connections      connections     database
```

### Installation

```bash
sudo apt install pgbouncer
```

### Configuration

```ini
; /etc/pgbouncer/pgbouncer.ini

[databases]
; Map logical database names to actual databases
myapp = host=localhost port=5432 dbname=myapp_production

[pgbouncer]
; Connection settings
listen_addr = 127.0.0.1
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

; Pool settings
pool_mode = transaction          ; Key setting - see below
max_client_conn = 1000           ; Max connections from clients
default_pool_size = 20           ; Connections to PostgreSQL per database
min_pool_size = 5                ; Minimum connections to keep open
reserve_pool_size = 5            ; Extra connections for burst traffic
reserve_pool_timeout = 3         ; Seconds before using reserve pool

; Timeouts
server_idle_timeout = 600        ; Close idle server connections
client_idle_timeout = 0          ; 0 = no timeout for client connections
server_connect_timeout = 15      ; Timeout for connecting to PostgreSQL
server_login_retry = 15          ; Retry interval on connection failure

; Logging
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1
stats_period = 60

; Admin console
admin_users = pgbouncer_admin
```

### Pool Modes Explained

**Session Pooling** (`pool_mode = session`)
- Connection assigned when client connects, released when disconnects
- Safest but least efficient
- Use when you have long-lived connections

**Transaction Pooling** (`pool_mode = transaction`)
- Connection assigned at transaction start, released at commit/rollback
- Best balance of safety and efficiency

**Statement Pooling** (`pool_mode = statement`)
- Connection assigned per statement
- Most aggressive pooling

### Django Configuration with PgBouncer

```python
# config/settings/production.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'myapp',  # Matches [databases] section
        'USER': config('DB_USER'),
        'PASSWORD': config('DB_PASSWORD'),
        'HOST': '127.0.0.1',  # PgBouncer host
        'PORT': '6432',       # PgBouncer port (not 5432!)
        'CONN_MAX_AGE': 0,    # Let PgBouncer handle pooling
        'OPTIONS': {
            'pool': False,    # Disable Django's native pooling
        },
        'DISABLE_SERVER_SIDE_CURSORS': True,  # Required for transaction pooling
    }
}
```

> **Important:** Transaction pooling breaks server-side cursors (e.g., `.iterator()`). You must disable them with `DISABLE_SERVER_SIDE_CURSORS`.

## Monitoring PgBouncer

```bash
psql -p 6432 -U pgbouncer_admin pgbouncer
```

```sql
SHOW POOLS;    -- Show pools
SHOW STATS;    -- Show stats
SHOW SERVERS;  -- Actual PostgreSQL connections
SHOW CLIENTS;  -- Connections from Django
SHOW CONFIG;   -- Show configuration
```

## When to Use What

| Scenario | Solution |
|----------|----------|
| Only Django uses PostgreSQL | Django Native Pooling |
| AWS Lambda / Serverless | PgBouncer |
| High-scale with multiple services | PgBouncer |
