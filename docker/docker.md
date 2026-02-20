# Dockerfile + uv — Line by Line Explained

> **Context:** Django project using `uv` as package manager, Redis, PostgreSQL (psycopg2).

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    UV_PYTHON_PREFERENCE=only-system \
    VIRTUAL_ENV=/opt/venv \
    PATH="/opt/venv/bin:${PATH}"

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
  build-essential libpq-dev \
  && rm -rf /var/lib/apt/lists/*

COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

COPY pyproject.toml .
RUN uv venv /opt/venv && uv pip install -e .

COPY . .
```

---

## FROM python:3.12-slim

Base image — starts with an official Python 3.12 image.  
`slim` means stripped down (no extra OS packages) → smaller image size.

---

## ENV

Each `ENV` sets an environment variable that persists across all subsequent layers and inside the running container.

> **Tip:** Combine multiple `ENV` lines into one to reduce image layers:
> ```dockerfile
> ENV PYTHONDONTWRITEBYTECODE=1 \
>     PYTHONUNBUFFERED=1 \
>     UV_PYTHON_PREFERENCE=only-system \
>     VIRTUAL_ENV=/opt/venv \
>     PATH="/opt/venv/bin:${PATH}"
> ```

### `PYTHONDONTWRITEBYTECODE=1`
Stops Python from writing `.pyc` bytecode cache files. Keeps the container clean, slightly smaller image.

### `PYTHONUNBUFFERED=1`
Forces Python stdout/stderr to be unbuffered — logs appear in real time in `docker logs` instead of being held in a buffer. Essential for debugging.

### `UV_PYTHON_PREFERENCE=only-system`
Tells `uv` to use the system Python (the one from the base image) rather than trying to download its own. Avoids conflicts.

### `VIRTUAL_ENV=/opt/venv` and `PATH="/opt/venv/bin:${PATH}"`

Declares where the venv will live and makes it the default Python/pip for the container.

**Why `/opt/venv`?**  
On Linux, `/opt` is the standard directory for "optional" or third-party software that doesn't belong to the OS. Just a convention — you could use any path, but `/opt/venv` is widely used.

**How PATH works:**  
PATH is a list of directories Linux searches left to right when you type a command. By default it looks like:
```
/usr/local/bin:/usr/bin:/bin
```
After the ENV line it becomes:
```
/opt/venv/bin:/usr/local/bin:/usr/bin:/bin
```
Since `/opt/venv/bin` is now **first**, Linux finds the venv's `python` and `pip` before the system ones.

If you appended instead of prepended, it would be ignored:
```dockerfile
ENV PATH="${PATH}:/opt/venv/bin"   # wrong — system Python wins
```

**Why not `RUN source activate`?**  
Each `RUN` in a Dockerfile runs in a separate shell — activating a venv in one `RUN` has zero effect on the next. Manipulating `PATH` via `ENV` persists across all layers, which is why this approach is used instead.

> **Important:** These `ENV` lines do NOT create the venv. They just declare where it will be. The venv is actually created later by `uv venv /opt/venv`.

---

## WORKDIR /app

Sets the working directory. All subsequent commands run from `/app`. Creates it if it doesn't exist.

Note: Dockerfile instructions (`WORKDIR`, `FROM`, `COPY`, `RUN`) don't use `=` — they just take arguments directly. `ENV` is the exception because it sets key-value variables.

---

## RUN apt-get update && apt-get install ...

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
  build-essential libpq-dev \
  && rm -rf /var/lib/apt/lists/*
```

- `apt-get update` — refreshes the package list
- `build-essential` — C compiler tools, needed to build some Python packages from source
- `libpq-dev` — PostgreSQL client headers, required to build `psycopg2`
- `--no-install-recommends` — skips optional packages, keeps image lean
- `rm -rf /var/lib/apt/lists/*` — deletes the package index cache after installing, reduces image size

---

## COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

Copies the `uv` binary from its official Docker image into your image.

**Why `COPY` and not `RUN`?**  
You could install `uv` with `RUN`, but it's slower and messier:
```dockerfile
RUN pip install uv
# or
RUN curl -fsSL https://astral.sh/uv/install.sh | sh
```
`COPY --from` just grabs a single prebuilt binary — no pip resolution, no extra layers, no curl.

**How `--from` works:**  
Normally `COPY` copies from your local machine. `--from` changes the source to another Docker image:
```dockerfile
COPY --from=<some-image> <path-inside-that-image> <destination-in-your-image>
```
Docker pulls that image temporarily just to extract that one file. Your final image only gets the binary, not the whole source image.

**What is multi-stage?**  
Multi-stage means using multiple `FROM` statements in one Dockerfile, where each stage can copy artifacts from previous ones:
```dockerfile
# Stage 1 - builder (heavy, has compilers etc.)
FROM python:3.12 AS builder
RUN pip install uv
RUN uv build myapp

# Stage 2 - final (lean, only what's needed to run)
FROM python:3.12-slim
COPY --from=builder /built/myapp .
```
The `COPY --from=ghcr.io/astral-sh/uv:latest` line is a lighter version of this — it borrows from an external image without defining its own build stages.

> **Tip:** Pin the uv version instead of using `latest` for reproducible builds:
> ```dockerfile
> COPY --from=ghcr.io/astral-sh/uv:0.5.1 /uv /usr/local/bin/uv
> ```

---

## COPY pyproject.toml .

Copies only `pyproject.toml` first — intentionally, not the whole codebase. See the caching note below.

---

## RUN uv venv /opt/venv && uv pip install -e .

- `uv venv /opt/venv` — **actually creates the virtualenv** at `/opt/venv`
- `uv pip install -e .` — installs your project and all dependencies defined in `pyproject.toml`

Because `VIRTUAL_ENV=/opt/venv` is already set in ENV, uv knows where the venv is — you don't need to pass `--python /opt/venv/bin/python` explicitly. That flag is redundant.

**Why copy `pyproject.toml` alone before `COPY . .`? — Docker layer caching**  
Docker builds images in layers, one per instruction. It caches each layer and reuses it if nothing changed. By copying `pyproject.toml` first and running install, that install layer gets cached. If you only change your application code (not dependencies), Docker skips the install step entirely on the next build — much faster.

If you did `COPY . .` first, any code change would invalidate the install cache and reinstall everything from scratch every time.

> **Tip:** If you use `uv.lock`, copy it alongside `pyproject.toml` for fully reproducible installs:
> ```dockerfile
> COPY pyproject.toml uv.lock .
> RUN uv venv /opt/venv && uv pip install --frozen -e .
> ```

---

## COPY . .

Copies the rest of your source code into `/app`. Done last so code changes don't invalidate the dependency install cache above.

> **Tip:** Add a `.dockerignore` file to avoid copying sensitive or unnecessary files:
> ```
> .git
> __pycache__
> *.pyc
> .env
> *.sqlite3
> .venv
> ```

---

## What's missing — things to add for production

**Non-root user** — running as root inside a container is a security risk:
```dockerfile
RUN adduser --disabled-password --gecos '' appuser
USER appuser
```

**CMD or ENTRYPOINT** — without this, the image doesn't know what to run:
```dockerfile
CMD ["gunicorn", "myproject.wsgi:application", "--bind", "0.0.0.0:8000"]
```

---

## uv.lock

`uv.lock` is a lockfile that records the exact versions of every dependency (and their sub-dependencies) resolved at install time.

Your `pyproject.toml` has loose version ranges:
```toml
dependencies = [
    "django>=4.0",
    "celery>=5.0",
]
```
Without a lockfile, two different builds on two different days might install different versions because a new release came out in between. `uv.lock` freezes the exact resolution so every environment — laptop, CI, Docker, production — gets identical packages.

You commit `uv.lock` to git.

Same concept as `package-lock.json` in Node or `Cargo.lock` in Rust.

---

## Full improved Dockerfile

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    UV_PYTHON_PREFERENCE=only-system \
    VIRTUAL_ENV=/opt/venv \
    PATH="/opt/venv/bin:${PATH}"

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
  build-essential libpq-dev \
  && rm -rf /var/lib/apt/lists/*

COPY --from=ghcr.io/astral-sh/uv:0.5.1 /uv /usr/local/bin/uv

COPY pyproject.toml uv.lock .
RUN uv venv /opt/venv && uv pip install --frozen -e .

COPY . .



```
