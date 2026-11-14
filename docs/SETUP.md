# Development Setup

How the local environment for **fx-payments-lakehouse** is set up, why, and how to start it again.

## Stack

| Component | Version | Role |
|---|---|---|
| WSL2 (Ubuntu 24.04) | – | Linux environment on Windows (ARM64) |
| Python | 3.11 (via `uv`) | Pipeline code, producer, notebooks |
| Java | OpenJDK 17 | Required by Spark |
| PySpark | 3.5.3 | Batch and Structured Streaming processing |
| Delta Lake (`delta-spark`) | 3.2.1 | Table format for Bronze/Silver/Gold |
| Kafka | 3.9.0 (Docker) | Event stream for payment events |
| PostgreSQL | 16 (Docker) | Serving layer for Gold tables |

- **WSL2 instead of native Windows:** Spark runs badly on native Windows. Ubuntu in WSL gives a real Linux environment.
- **Project lives inside WSL (`~/projects`), not on `C:`:** files under `/mnt/c/...` are much slower for Spark and Python.
- **Docker for Kafka and Postgres:** nothing installed on the machine itself, and anyone can reproduce the environment with one command.
- **`uv` for Python:** fast, and manages the Python version (3.11) and the virtual environment.
- **PySpark and delta-spark versions are pinned together:** they must be compatible. If you upgrade one, check Delta Lake's compatibility table and upgrade the other to match.

## Project structure

```
fx-payments-lakehouse/
├── data/              # Delta tables (bronze/silver/gold) – not in Git
├── checkpoints/       # Spark Streaming checkpoints – not in Git
├── notebooks/         # Exploration and walkthrough notebooks
├── producer/          # Kafka producer that generates payment events
├── src/               # Reusable code (Spark session, transformations)
├── tests/             # pytest tests
├── docker-compose.yml # Kafka + Postgres
├── requirements.txt   # Python dependencies
├── .env               # Local credentials – not in Git
└── .gitignore
```

## First-time setup (on a new machine)

1. Install **Docker Desktop**, then enable Settings → Resources → WSL integration → **Ubuntu**.
2. In **PowerShell**:
   ```powershell
   wsl --install -d Ubuntu
   wsl --set-default Ubuntu
   ```
3. In **Ubuntu**:
   ```bash
   sudo apt update
   sudo apt install -y openjdk-17-jdk git
   curl -LsSf https://astral.sh/uv/install.sh | sh
   source ~/.bashrc
   sudo usermod -aG docker $USER   # then restart WSL (see below)
   ```
4. Set up Git and an SSH key, and add the public key on GitHub:
   ```bash
   git config --global user.name "<name>"
   git config --global user.email "<email>"
   ssh-keygen -t ed25519 -C "wsl-laptop"
   cat ~/.ssh/id_ed25519.pub
   ssh -T git@github.com
   ```
5. Clone and install:
   ```bash
   mkdir -p ~/projects && cd ~/projects
   git clone git@github.com:lsmk1783/fx-payments-lakehouse.git
   cd fx-payments-lakehouse
   uv venv --python 3.11
   source .venv/bin/activate
   uv pip install -r requirements.txt
   ```
6. Create `.env` (not committed):
   ```
   POSTGRES_USER=de
   POSTGRES_PASSWORD=de
   POSTGRES_DB=warehouse
   POSTGRES_PORT=5433
   ```

## Daily routine

1. Open **Docker Desktop** and wait for "Engine running".
2. Open **Ubuntu** (Windows Terminal or `wsl -d Ubuntu` in PowerShell).
3. Start working:
   ```bash
   cd ~/projects/fx-payments-lakehouse
   source .venv/bin/activate
   docker compose up -d
   code .
   ```
4. When finished:
   ```bash
   docker compose stop
   ```

## Useful commands

**Docker**
```bash
docker compose up -d        # start Kafka + Postgres in the background
docker compose ps           # check what's running
docker compose logs kafka   # see a container's logs (or postgres)
docker compose stop         # stop the containers, keeping the data
docker compose down         # remove the containers (data stays in the volume)
docker compose down -v      # remove everything, including Postgres data (careful)
```

**Postgres** (port **5433** on the host, because 5432 may be taken by a local Postgres)
```bash
docker compose exec postgres psql -U de -d warehouse            # interactive shell
docker compose exec postgres psql -U de -d warehouse -c "\dt"   # list tables
```
From Spark/Python: `jdbc:postgresql://localhost:5433/warehouse`

**Kafka** (port **9092**)
```bash
# list topics
docker compose exec kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
# create a topic
docker compose exec kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic payments
# read messages from a topic (Ctrl+C to stop)
docker compose exec kafka /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic payments --from-beginning
```

**Python environment**
```bash
source .venv/bin/activate          # activate (prompt shows the project name)
uv pip install <package>           # install a package
uv pip freeze                      # see installed versions
```

**Spark**
- The Spark UI is at http://localhost:4040 while a Spark session is running.

**Git**
```bash
git status
git add .
git commit -m "message"
git push
```

## Things to remember

- **Always work in the Ubuntu terminal.** A prompt like `smyko@lsmksystem:~$` is Ubuntu. A prompt containing `CLANGARM64` is Git Bash, which is wrong for this project.
- **`wsl` commands only work in PowerShell,** never inside Ubuntu.
- **`wsl --shutdown` also stops Docker Desktop.** Reopen Docker Desktop afterwards.
- **VS Code must show "WSL: Ubuntu" in the bottom-left corner.** If it doesn't, open the project from Ubuntu with `code .`.
- **Never install Docker with `apt` inside Ubuntu.** Docker Desktop already provides it.
- **`data/`, `checkpoints/` and `.env` are never committed.** Data is regenerated by the pipeline.
- **Credentials in `.env` are for local development only.**

## Troubleshooting

| Problem | Fix |
|---|---|
| `permission denied ... docker.sock` | `sudo usermod -aG docker $USER`, then `wsl --shutdown` in PowerShell and reopen Ubuntu and Docker Desktop |
| `open //./pipe/dockerDesktopLinuxEngine` | You're in Git Bash, or Docker Desktop isn't running |
| `docker ps` fails in Ubuntu | Check Docker Desktop is open and WSL integration for Ubuntu is on |
| VS Code asks to allow host `wsl.localhost` | Cancel, install the **WSL** extension, reopen with `code .` from Ubuntu |
| `java: command not found` | Install it inside Ubuntu: `sudo apt install openjdk-17-jdk` |
| Spark is very slow | Check the project is under `~/projects`, not `/mnt/c/...` |