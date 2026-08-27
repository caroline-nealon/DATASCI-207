# MIDS 204 Applied Data Engineering
## Module 1 Sync Session: Notes and Commands

Everything below is copy-pasteable.

**Every command in this course is written for `bash`.**

- **macOS / Linux:** use your normal terminal. Everything works as written.
- **Windows:** use **Git Bash**, which ships with [Git for Windows](https://git-scm.com/download/win).

Where the two differ, you will see a **macOS / Linux** block and a **Windows (Git Bash)** block. Where you see neither, the command is identical on all platforms.

> **Windows students: do the [one-time setup](#0-windows-one-time-setup) in Section 0 first.** It takes a minute and prevents two failures that are hard to diagnose.

---

## 0. Windows One-Time Setup

Skip this section if you are on macOS or Linux.

### Set your terminal's encoding to UTF-8

Git Bash reports its encoding to Python as Windows-1252. The FastAPI CLI prints a box-drawing banner on startup, and those characters do not exist in Windows-1252, so `fastapi dev` crashes before your server ever starts. The traceback says `UnicodeEncodeError` and gives no hint that encoding is the underlying problem.

Fix it once, permanently:

```bash
echo 'export PYTHONIOENCODING=utf-8' >> ~/.bashrc
```

Then reload:

```bash
source ~/.bashrc
```

Verify:

```bash
echo $PYTHONIOENCODING
```

You should see `utf-8`. This is the single most likely thing to derail a Windows student in Module 1.

### Keep your project paths short

Windows has a 260-character path limit. `uv init` runs `git init`, which creates deeply nested files such as `.git/hooks/applypatch-msg.sample`. On a long path that fails with `Filename too long` and leaves you a broken, half-initialized project.

Work somewhere shallow. A folder directly under your home directory is ideal:

```bash
mkdir -p ~/dev && cd ~/dev
```

Avoid working inside deep cloud-synced folders such as `~/OneDrive/School/Semester 3/...`.

---

## 1. Course Overview

**What this course is**
- Fundamentals of Data Engineering
- Technology agnostic
- A wide sample of concepts

**What this course is not**
- Full-depth analysis of any one topic
- Technology-stack specific
- A way to solve all your data problems

**How to get the most out of it**
- Ask questions
- Participate on Slack
- Look at assignments early
- Project assignments build on top of each other
- The final project builds on the prior projects

**GenAI policy**
- Use every tool available to you to learn: office hours, Slack, lectures, GenAI
- GenAI can be a tool for learning. Use it to understand the material, then do your own work
- Submitting work you did not do violates the University honor code

---

## 2. Topics Covered This Week

- Data Engineering roles and responsibilities
- Data lifecycles
- Software development lifecycles
- VMs and containers

Discussion prompts: *What made sense? What didn't make sense? What needs more clarity?*

---

## 3. Dependency Management with `uv`

Docs: <https://docs.astral.sh/uv/>

### Install uv

The same command works on all three platforms, Git Bash included. The installer detects MINGW/MSYS and downloads the native Windows `uv.exe`:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

It installs to `~/.local/bin` and appends a `PATH` line to your shell profile. To use it in the shell you already have open:

```bash
source ~/.local/bin/env
```

Verify:

```bash
uv --version
```

If it still says `command not found`, close and reopen your terminal.

### Create a project

```bash
uv init my_special_project --app --no-package
```

```bash
cd my_special_project
```

> **Why `--no-package`?** As of uv 0.12, `uv init --app` builds an installable package with a `src/my_special_project/` directory and a `[build-system]` section in your `pyproject.toml`. That is the right default for shipping real software, and it is more machinery than we need today. `--no-package` gives the flat layout this demo uses: a single `main.py` at the project root. If you follow a tutorial that shows a bare `main.py` after plain `uv init`, this flag is the difference.

Look at what it generated:

```bash
ls -la
```

```bash
cat pyproject.toml
```

> `tree` also works, though it is absent by default in Git Bash and on macOS. `ls -la` works everywhere.

You should have:

| File | What it is |
|---|---|
| `pyproject.toml` | Project metadata and declared dependencies |
| `.python-version` | Pins the Python version for this project |
| `main.py` | Entry point stub |
| `README.md` | Empty readme |
| `.gitignore` | Ignores `__pycache__/`, `.venv`, build artifacts |

### Add dependencies

```bash
uv add "fastapi[standard]"
```

Quote it. In `bash` and `zsh`, unquoted `fastapi[standard]` gets eaten by glob expansion.

Look at `pyproject.toml` again. The dependency is now declared there:

```bash
cat pyproject.toml
```

Add a dev-only dependency, meaning test tooling and linters that stay out of production:

```bash
uv add --dev pytest
```

That one lands in a separate `[dependency-groups]` section rather than in `dependencies`. Open `pyproject.toml` once more and confirm you can see the split. That separation is the entire point.

### Lock and sync

```bash
uv lock
```

```bash
uv sync
```

- `uv lock` resolves every dependency, and every transitive dependency, to an **exact** version and writes `uv.lock`.
- `uv sync` makes your `.venv` match `uv.lock` exactly, installing what is missing and removing what should not be there.

Commit `pyproject.toml` and `uv.lock`. Never commit `.venv/`.

> `.venv` is not portable. It contains absolute paths and platform-specific binaries: a `.venv` built on macOS uses `bin/`, one built on Windows uses `Scripts/`. Copying one between machines fails. That is what `uv.lock` is for. You commit the lockfile, and each machine rebuilds its own `.venv` from it.

### Run things

```bash
uv run pytest
```

```bash
uv run python main.py
```

`uv run` executes inside the project venv without you activating anything. You do not need `source .venv/bin/activate`.

---

## 4. FastAPI

Docs and example: <https://fastapi.tiangolo.com/#example>

### Why REST APIs exist

An API is an **interface method**, a contract between two systems. Instead of every consumer needing your database credentials, your schema, and your Python code, they make an HTTP request and get structured data back. The interface stays stable while the implementation behind it changes freely. That decoupling is the entire point, and it is why nearly every data platform component you will touch this term speaks HTTP.

### The demo app

Replace the contents of `main.py` with:

```python
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
def read_root():
    return {"Hello": "World"}


@app.get("/items/{item_id}")
def read_item(item_id: int, q: str | None = None):
    return {"item_id": item_id, "q": q}
```

What is happening:
- `@app.get("/")` registers a handler for `GET /`
- `{item_id}` is a **path parameter**. The `item_id: int` type hint makes FastAPI parse and validate it, so a request to `/items/abc` returns a `422` with a clear error, for free
- `q: str | None = None` is an optional **query parameter**
- Returned dicts are serialized to JSON automatically

### Run it

Development, with auto-reload on every file save:

```bash
uv run fastapi dev
```

Production, with no reloading and no file watcher:

```bash
uv run fastapi run
```

Change the `"Hello": "World"` string while `dev` is running and the server reloads itself. Do the same under `run` and nothing happens until you restart. That is the whole difference.

Stop either one with `Ctrl+C`.

> **Windows:** if this crashes instantly with `UnicodeEncodeError: 'charmap' codec can't encode characters`, you skipped [Section 0](#0-windows-one-time-setup). Set `PYTHONIOENCODING=utf-8` and try again.

### Try it in a browser

- <http://127.0.0.1:8000/> for the root endpoint
- <http://127.0.0.1:8000/items/5?q=someval> for a path param plus query param
- <http://127.0.0.1:8000/docs> for the auto-generated interactive API docs

Expected responses:

| Request | Response |
|---|---|
| `/` | `{"Hello":"World"}` |
| `/items/5?q=someval` | `{"item_id":5,"q":"someval"}` |
| `/items/5` | `{"item_id":5,"q":null}` |
| `/items/abc` | HTTP 422 validation error |

The query parameter is named **`q`**, because that is what the function signature declares. `?param=someval` will be silently ignored and you will get `"q":null` back.

The `/docs` page is generated from your type hints. You never wrote it, and it stays in sync with your code.

---

## 5. Docker

**A Docker container is a way to package your application so it runs and executes in multiple environments,** including your laptop, a teammate's laptop, CI, and production, identically in each.

### The mental model

```
Code  --build-->  Image  --run-->  Container
```

- **Code** is your source, plus a `Dockerfile` describing how to package it
- **Image** is an immutable, built artifact. Like a class.
- **Container** is a running instance of an image. Like an object. One image, many containers.

### Installing Docker

**macOS / Linux**

Install [Docker Desktop](https://docs.docker.com/get-docker/) on macOS, or Docker Engine from your package manager on Linux. The `docker` command is available in your terminal immediately.

**Windows (Git Bash)**

Docker Desktop requires the **WSL2 backend**. If you have never used WSL, install it first. Open Git Bash **as Administrator** (right-click the Git Bash icon, choose "Run as administrator") and run:

```bash
wsl --install --no-launch
```

This enables the Windows features and downloads WSL. It does not restart your machine on its own. `--no-launch` keeps it from opening a distro setup prompt at the end.

Before running anything, confirm the window is actually elevated. "Run as administrator" is easy to miss, and the commands below fail quietly enough that people assume they worked:

```bash
net session >/dev/null 2>&1 && echo "ELEVATED" || echo "NOT ELEVATED"
```

If you would rather enable the features with an explicit no-restart flag, run these two instead, still as Administrator:

```bash
MSYS_NO_PATHCONV=1 dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
```

```bash
MSYS_NO_PATHCONV=1 dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

`MSYS_NO_PATHCONV=1` is required here for the reason described in the Git Bash quirks section below. Without it, Git Bash rewrites `/online` into a Windows path and DISM rejects the arguments.

With `/norestart`, DISM stays silent about restarting. That silence is the flag working, and it means you get no visible confirmation that anything is queued. Check for yourself:

```bash
MSYS_NO_PATHCONV=1 reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing\PackagesPending" 2>/dev/null | grep -q "DISM Package Manager" && echo "STAGED: reboot to activate" || echo "NOTHING STAGED: rerun the commands in an elevated window"
```

**Reboot when you are ready.** The features stay inactive until you do.

After the reboot, install the WSL2 kernel:

```bash
wsl --update
```

If that fails to reach the Microsoft Store, pull it over plain HTTPS instead:

```bash
wsl --update --web-download
```

Then confirm:

```bash
wsl --status
```

You want `Default Version: 2` with no complaint about a missing kernel. A line reading `The WSL 2 kernel file is not found` means the update step did not complete.

> **Why the reboot changes what `wsl.exe` accepts.** Before the features are active, `wsl.exe` is a launcher stub that understands only `--install`, `--list`, `--status`, and `--help`. Anything else prints generic usage text with no error explaining the rejection. After the reboot it becomes the full inbox WSL, which is when `--update` and `--set-default-version` start working. Note that the inbox build has no `--no-distribution` flag, since that one belongs to the Microsoft Store build of WSL. Running `wsl --update` then replaces the inbox build with the Store build, which is why `wsl --version` starts returning a version table afterwards.

You do not need Ubuntu or any other Linux distribution. `wsl -l -v` reporting "no installed distributions" is the expected state. Docker Desktop registers its own internal `docker-desktop` distro when it starts.

With WSL2 ready, install Docker Desktop. Either route works:

```bash
winget install --id Docker.DockerDesktop --accept-package-agreements --accept-source-agreements
```

Or download the [Docker Desktop installer](https://docs.docker.com/desktop/install/windows-install/) and run it, selecting the WSL2 backend when asked.

Both routes raise a UAC prompt, and neither restarts your machine on its own. The installer may ask you to sign out of Windows at the end, which you can decline and fold into your next restart.

**Close Git Bash and open it again after the install finishes.** The installer adds `C:\Program Files\Docker\Docker\resources\bin` to your system `PATH`, and any shell already open is still running with the older `PATH`. Skipping this gives you `docker: command not found` on a perfectly good install.

Then launch Docker Desktop from the Start Menu. The first run asks you to accept its service agreement and takes a minute to register its WSL distro. Wait for the whale icon in the system tray to stop animating before running any `docker` command, since the CLI and the daemon start independently.

**Yes, `docker` works directly in Git Bash.** Docker Desktop installs `docker.exe` onto the Windows `PATH`, and Git Bash inherits that `PATH`. There is no extra configuration, no aliases, and no WSL shell required. You type the same `docker` commands as everyone else in the class. The daemon runs inside a WSL2 virtual machine, and that stays invisible to you, since the CLI you are calling is a native Windows binary.

Verify:

```bash
docker --version
```

```bash
docker run hello-world
```

### Dockerfile

```dockerfile
FROM python:3.14-slim

# Install uv.
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

# Copy the application into the container.
COPY . /app

# Install the application dependencies.
WORKDIR /app
RUN uv sync --frozen --no-cache

# Run the application.
CMD ["/app/.venv/bin/fastapi", "run", "main.py", "--port", "80", "--host", "0.0.0.0"]
```

Line by line:

| Instruction | What it does |
|---|---|
| `FROM python:3.14-slim` | Base image. `-slim` trims OS packages you do not need, giving a smaller image and a smaller attack surface. |
| `COPY --from=ghcr.io/...uv` | Pulls the `uv` binary out of a published image instead of installing it via a script. Fast and version-pinnable. |
| `COPY . /app` | Copies your project into the image. |
| `WORKDIR /app` | Sets the working directory for everything after it. |
| `RUN uv sync --frozen --no-cache` | Installs dependencies. `--frozen` means *use `uv.lock` exactly, do not re-resolve*, so the build fails loudly instead of silently drifting. `--no-cache` keeps the image small. |
| `CMD [...]` | The command that runs when the container starts. |

Three details worth internalizing:

- `--host 0.0.0.0` is **required**. The default `127.0.0.1` binds only to the container's own loopback interface, so nothing outside the container can ever reach it.
- `uv sync --frozen` is the reason you commit `uv.lock`. It is what makes the image reproducible.
- The path in `CMD` is `main.py`. `COPY . /app` puts your file at `/app/main.py`, and `WORKDIR /app` means paths resolve from there. Writing `app/main.py` would look for `/app/app/main.py`, and the container would die on startup.

### Build and run

Build the image with a sensible tag:

```bash
docker build -t mids204-fastapi:v1 .
```

Now run it **without** publishing a port:

```bash
docker run mids204-fastapi:v1
```

The logs say the app started. Now open <http://127.0.0.1:80> and you get nothing. The app is listening on port 80 inside the container, and that port is connected to nothing on your machine.

Stop it with `Ctrl+C` and run it again **with** a port binding:

```bash
docker run -p 8000:80 mids204-fastapi:v1
```

Now <http://127.0.0.1:8000/docs> works.

`-p 8000:80` reads as **`-p <host-port>:<container-port>`**. Traffic arriving at port 8000 on your machine is forwarded to port 80 inside the container. Containers are network-isolated by default, so you have to explicitly poke the hole.

### Useful extras

Run detached, with a name:

```bash
docker run -d -p 8000:80 --name api mids204-fastapi:v1
```

```bash
docker ps
```

```bash
docker logs -f api
```

```bash
docker stop api
```

```bash
docker rm api
```

```bash
docker images
```

### Windows only: two Git Bash quirks

Every command above works as written in Git Bash. These two bite the moment you go further, and neither error message points at the real cause. macOS and Linux users can skip this.

**1. Git Bash rewrites arguments that start with `/`.**

Git Bash converts Unix-style paths into Windows ones before handing them to a Windows program. Docker arguments are the classic casualty:

```bash
docker exec -it api /bin/bash
```

Git Bash silently turns `/bin/bash` into `C:/Program Files/Git/bin/bash`, and Docker fails with a confusing "no such file or directory". Prefix the command to turn the rewriting off:

```bash
MSYS_NO_PATHCONV=1 docker exec -it api /bin/bash
```

Same story for volume mounts:

```bash
MSYS_NO_PATHCONV=1 docker run -p 8000:80 -v "$(pwd)":/app mids204-fastapi:v1
```

Doubling the leading slash, as in `//bin/bash`, also works and is shorter. `MSYS_NO_PATHCONV=1` is clearer about why it is there.

`-p 8000:80` is unaffected, since it has no leading slash and nothing to rewrite. This only matters once you start passing paths.

**2. Interactive containers need a real terminal.**

If an interactive `-it` command fails with `the input device is not a TTY`, wrap it:

```bash
winpty docker exec -it api /bin/bash
```

`winpty` ships with Git for Windows. You need it only for **interactive** sessions. `docker build`, `docker run -p ...`, and `docker logs` all work without it.

---

## 6. Full Sequence: Zero to Container

Windows students: complete [Section 0](#0-windows-one-time-setup) first, and run this somewhere shallow such as `~/dev`.

```bash
uv init my_special_project --app --no-package
```

```bash
cd my_special_project
```

```bash
uv add "fastapi[standard]"
```

```bash
uv add --dev pytest
```

```bash
uv lock
```

```bash
uv sync
```

Edit `main.py` with the FastAPI app from Section 4, then:

```bash
uv run fastapi dev
```

Confirm <http://127.0.0.1:8000/docs> loads, then press `Ctrl+C`.

Add the `Dockerfile` from Section 5, then:

```bash
docker build -t mids204-fastapi:v1 .
```

```bash
docker run -p 8000:80 mids204-fastapi:v1
```

Confirm <http://127.0.0.1:8000/docs> loads again, this time served from inside a container.

---

## 7. What Each Tool Is Doing

| Tool | Role |
|---|---|
| **uv** | Python package and project manager. Resolves and installs dependencies, manages the `.venv`, pins the Python version, and runs commands inside the project environment. Replaces the `pip` + `venv` + `pip-tools` + `pyenv` stack with one fast binary. |
| **pyproject.toml** | Declares what your project needs, as loose constraints, written by you. |
| **uv.lock** | Records exactly what got installed, as pinned versions plus hashes, generated by uv. This is what makes builds reproducible. |
| **FastAPI** | Python web framework for building HTTP APIs. Uses type hints for validation and auto-generates interactive docs. |
| **uvicorn** | The ASGI server that listens on a socket and runs your FastAPI app. Comes in via `fastapi[standard]`. `fastapi dev` and `fastapi run` are wrappers around it. |
| **Docker** | Packages your app plus its OS-level dependencies into a portable image, then runs it in an isolated container. |
| **Dockerfile** | The recipe for building that image. |
| **pytest** | Test runner. Added with `--dev` so it is available locally and in CI while staying out of your production image. |
| **Git Bash** *(Windows)* | Provides a POSIX shell on Windows so one set of commands works for the whole class. |
| **WSL2** *(Windows)* | The Linux virtual machine Docker Desktop runs its daemon inside. You install it once and then ignore it. |

**Where the boundary sits:** `uv` handles Python dependencies. Docker handles everything below Python: the interpreter itself, system libraries, and OS packages. That is why the Dockerfile starts `FROM python:3.14-slim` and then runs `uv sync`. Two layers of dependency management, each solving a different problem.

---

## 8. Troubleshooting

### All platforms

| Symptom | Fix |
|---|---|
| `no matches found: fastapi[standard]` | Quote it: `uv add "fastapi[standard]"` |
| `uv: command not found` after install | Run `source ~/.local/bin/env`, or reopen your terminal |
| `tree: command not found` | Absent by default on Windows and macOS. Use `ls -la`. |
| `uv init` gave me `src/` instead of `main.py` | Add `--no-package`. See Section 3. |
| `Address already in use` on port 8000 | Use `uv run fastapi dev --port 8001`, or stop the other process |
| Changes to `main.py` do not show up | You are on `fastapi run`. Use `fastapi dev` for auto-reload. |
| `/items/5?param=x` returns `"q":null` | The parameter is named `q`. Use `?q=x`. |
| App runs in Docker but the browser shows nothing | Missing `-p 8000:80`, or the app is not bound to `--host 0.0.0.0` |
| `uv sync --frozen` fails during Docker build | `uv.lock` is stale or missing. Run `uv lock` locally, and confirm the lockfile is not excluded by `.dockerignore`. |
| Docker build is slow every time | Normal on the first build. Later builds reuse cached layers as long as earlier instructions have not changed. |

### Windows (Git Bash) only

| Symptom | Fix |
|---|---|
| `UnicodeEncodeError: 'charmap' codec can't encode characters` when starting FastAPI | Set `PYTHONIOENCODING=utf-8`. See [Section 0](#0-windows-one-time-setup). |
| `fatal: cannot stat ... Filename too long` during `uv init` | Path too long. Delete the broken folder and start again somewhere shallow such as `~/dev`. |
| Docker: `no such file or directory` for a path that clearly exists | Git Bash rewrote it. Prefix with `MSYS_NO_PATHCONV=1`. |
| Docker: `the input device is not a TTY` | Prefix the interactive command with `winpty`. |
| `docker: command not found` immediately after installing | Your shell read its `PATH` when it opened, before the installer changed it. Close Git Bash and open it again. |
| `docker: command not found` in a freshly opened shell | Confirm `C:\Program Files\Docker\Docker\resources\bin` exists and appears in your system `PATH`. |
| `failed to connect to the docker API at npipe:....` | The CLI is working and the daemon is not running. Start Docker Desktop and wait for the whale icon to stop animating. |
| Docker Desktop will not start | WSL2 is missing. Run `wsl --install`, reboot, try again. |

---

## 9. Reference Links

- uv: <https://docs.astral.sh/uv/>
- FastAPI: <https://fastapi.tiangolo.com/>
- FastAPI first example: <https://fastapi.tiangolo.com/#example>
- FastAPI in containers: <https://fastapi.tiangolo.com/deployment/docker/>
- Docker install: <https://docs.docker.com/get-docker/>
- Git for Windows: <https://git-scm.com/download/win>
