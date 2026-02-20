# Docker — Image vs Container

## The analogy

**Image is a class. Container is an instance of that class.**

You can have as many containers as your host machine's resources (RAM, CPU) allow — there's no Docker limit.

---

## Image

A static, read-only blueprint — it contains your code, dependencies, OS files, environment variables, everything baked in at build time. It sits on disk doing nothing. You can share it, push it to Docker Hub, pull it on another machine.

Built once with `docker build` (or `build: .` in Compose).

---

## Container

A running instance of that image — the image brought to life. It has its own process, its own filesystem layer on top of the image, its own network.

---

## One image, many containers

In your Compose file, `build: .` appears only once — on `backend`. That's the only service that triggers `docker build`. Every other service just reuses the same built image:

```yaml
backend:
  build: .                        # only this one actually builds
  image: cryphos-app:latest       # tags the built image

liquidations:
  image: cryphos-app:latest       # reuses — no build
  container_name: cryphos-liquidations

collector:
  image: cryphos-app:latest       # reuses — no build

klines_socket:
  image: cryphos-app:latest       # reuses — no build

celery:
  image: cryphos-app:latest       # reuses — no build

celery-beat:
  image: cryphos-app:latest       # reuses — no build

tg:
  image: cryphos-app:latest       # reuses — no build
```

One image (`cryphos-app:latest`), but each service is a separate container running a different command. Same codebase, same dependencies, same environment — just different entry points. Instead of building 6 images, you build one.

---

## container_name is optional

Without it, Docker auto-generates a name like `cryphos_liquidations_1`. With it, you get a predictable name for easier debugging:

```bash
# without container_name
docker logs cryphos_liquidations_1

# with container_name
docker logs cryphos-liquidations
```

Pure convenience, nothing functional.

---

## Dockerfile vs docker-compose.yaml

The Dockerfile defines everything baked into the image — OS, Python version, dependencies, environment variables, working directory. Every container that runs from that image inherits all of it automatically.

docker-compose.yaml can then **add or override** on top per container:

```yaml
celery:
  image: cryphos-app:latest       # inherits everything from Dockerfile
  environment:
    PYTHONPATH: /app               # adds/overrides env var
  working_dir: /app/apis          # overrides WORKDIR from Dockerfile
  command: celery -A apis worker  # overrides CMD from Dockerfile
```

| | Purpose |
|---|---|
| **Dockerfile** | What's the same for everyone — base image, installed packages, copied code |
| **docker-compose.yaml** | What's different per container — command, extra env vars, volumes, ports |

Or in class terms:
- **Dockerfile** = the class definition
- **docker-compose.yaml** = how you instantiate each object with different arguments

---

## Port binding

```yaml
db:
  ports:
    - "127.0.0.1:5432:5432"
```

Format is `host:container`.

Without `ports`, Postgres runs inside the container on port 5432 but is completely isolated — only other containers on the same Docker network can reach it. Your `backend`, `celery` etc. connect fine without any port binding.

With `ports`, you can connect from your **host machine** — useful for dev tools like DBeaver, TablePlus, pgAdmin, or `psql` from your terminal.

Binding to `127.0.0.1` specifically means only you on localhost can access it, not anyone on the internet. In pure production with no need for local DB access, you'd remove the port binding entirely.

**Rule of thumb:**
- Inter-container communication → no ports needed, Docker internal DNS handles it
- Access from host machine → expose ports, bind to `127.0.0.1` for safety
