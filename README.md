# DockerProject
This is a self-learning about docker, how it works and how it is implemented in this project.  

# Tents in SA – Rental Analytics

A data-analysis and machine-learning project for **Tents in SA**, a (fictional) company that rents
rooms, flats, cottages and houses to tenants across South Africa's provinces.

> All data is **synthetic**: randomly generated with a fixed seed. No real people or properties.

**Quick start (if Docker is already installed):**
```bash
git clone <your-repo-url>
cd tents-in-sa
docker compose up --build
```
Then open **http://localhost:5006/app**.

---

## Table of contents
1. [What this project does](#1-what-this-project-does)
2. [What is Docker?](#2-what-is-docker)
3. [How to install Docker](#3-how-to-install-docker)
4. [How to run the project](#4-how-to-run-the-project)
5. [How Docker is used in this project](#5-how-docker-is-used-in-this-project)
6. [Dashboard guide](#6-dashboard-guide)
7. [Dataset columns](#7-dataset-columns)
8. [Project layout](#8-project-layout)
9. [Troubleshooting](#9-troubleshooting)
10. [Caveats](#10-caveats)

---

## 1. What this project does

| Step | Script | Result |
|------|--------|--------|
| Dataset (100 rentals) | `src/generate_data.py` | `data/tents_in_sa_rentals.csv` |
| Exploratory analysis | `src/analysis.py` | summary CSVs + 5 charts in `outputs/` |
| Machine learning | `src/train_model.py` | trained models + `metrics.json` in `outputs/` |
| Dashboard | `src/app.py` (Panel) | interactive website on port 5006 |

**Machine-learning tasks**
- **Regression:** predict the monthly rent of a property (Ridge, Random Forest and Gradient Boosting are compared).
- **Classification:** predict whether a tenant will renew their lease (Logistic Regression and Random Forest are compared).

Models are compared using 5-fold cross-validation, and the best model of each type is saved and used by the dashboard.

---

## 2. What is Docker?

**Docker** is a tool that packages an application together with everything it needs to run:
the right version of Python, the libraries, and the settings. This package runs in an isolated
environment called a **container**.

### The problem it solves
Without Docker, sharing a project often ends with *"it works on my computer, but not on yours"*:
different Python versions, missing libraries, conflicting library versions, different operating systems.
Docker removes this problem: **if it runs in the container on one computer, it runs the same way on any
computer that has Docker.**

### Key terms

| Term | Meaning | Everyday comparison |
|------|---------|---------------------|
| **Image** | A read-only template containing the app and its environment | A recipe / a frozen meal |
| **Container** | A running instance of an image | The meal being served |
| **Dockerfile** | A text file with the instructions for building an image | The written recipe |
| **Docker Compose** | A tool that starts containers from a simple config file (`docker-compose.yml`) | A single button that runs the recipe with the right settings |
| **Volume** | A folder shared between your computer and the container | A shared drawer both sides can use |
| **Port mapping** | Connects a port on your computer to a port inside the container | A door from your computer into the container |

### Containers vs virtual machines
A virtual machine runs a whole extra operating system, which is heavy and slow to start.
A container shares your computer's operating system and only adds what the app needs, so it
is small and starts in seconds.

---

## 3. How to install Docker

Official guide: <https://docs.docker.com/get-started/get-docker/>

### Windows 10 / 11
1. Download **Docker Desktop for Windows** from <https://www.docker.com/products/docker-desktop/>.
2. Run the installer. Keep the option **"Use WSL 2"** ticked (recommended).
3. Restart the computer if asked.
4. Open **Docker Desktop** and wait until it shows **"Engine running"**.
5. If Docker complains about virtualisation, enable *virtualization* (Intel VT-x / AMD-V / SVM) in the
   computer's BIOS/UEFI settings.

### macOS
1. Download **Docker Desktop for Mac** from the same page. Choose **Apple Silicon** (M1/M2/M3/M4) or **Intel**
   depending on your Mac (Apple menu → About This Mac).
2. Open the `.dmg`, drag Docker into **Applications**, and launch it.
3. Wait until Docker shows **"Engine running"**.

### Linux (Ubuntu/Debian example)
Follow the current instructions at <https://docs.docker.com/engine/install/> to install
**Docker Engine** and the **Docker Compose plugin** (or install Docker Desktop for Linux). Then optionally allow your user to run Docker
without `sudo`:
```bash
sudo usermod -aG docker $USER   # log out and back in afterwards
```

### Check that it works
Open a terminal (PowerShell, Command Prompt, Terminal) and run:
```bash
docker --version
docker compose version
```
Both should print a version number. You can also run `docker run hello-world` for a full test.

> Docker Desktop must be **open and running** whenever you use the commands in this README.

---

## 4. How to run the project

### Step 1: Get the project
```bash
git clone <your-repo-url>
cd tents-in-sa
```
(Or unzip `tents-in-sa.zip` and open a terminal inside the unzipped folder.)

### Step 2: Build and start
```bash
docker compose up --build
```
The first run takes a few minutes because Docker downloads Python and installs the libraries.
Later runs start in seconds.

### Step 3: Wait for this message
```
>>> Dashboard starting: open http://localhost:5006/app in your browser
```

### Step 4: Open the dashboard
Go to **http://localhost:5006/app** in a web browser.

### Step 5: Stop
Press `Ctrl+C` in the terminal, then:
```bash
docker compose down
```

### Alternative: without Docker Compose
```bash
docker build -t tents-in-sa .
docker run --rm -p 5006:5006 -v "$(pwd)/outputs:/app/outputs" tents-in-sa
```
(On Windows PowerShell use `${PWD}` instead of `$(pwd)`.)

### Where are the results?
The **`outputs/`** folder in the project fills up with charts (PNG), summary tables (CSV),
the trained models (`.joblib`) and `metrics.json`. This works because of the volume described below.

---

## 5. How Docker is used in this project

Three files make the project runnable anywhere:

### `requirements.txt` – the exact libraries
Lists every Python library with a **pinned version** (for example `pandas==3.0.2`), so everyone
gets identical versions.

### `Dockerfile` – how the image is built
```dockerfile
FROM python:3.12-slim                     # 1. start from a small official Python image

ENV PYTHONDONTWRITEBYTECODE=1 \           # 2. settings: no .pyc files, live log output,
    PYTHONUNBUFFERED=1 \                  #    charts drawn without a screen
    MPLBACKEND=Agg

WORKDIR /app                              # 3. work inside /app in the container

COPY requirements.txt .                   # 4. copy the library list first...
RUN pip install --no-cache-dir -r requirements.txt   # ...and install it (cached by Docker)

COPY src/ ./src/                          # 5. copy the code and the dataset
COPY data/ ./data/

EXPOSE 5006                               # 6. the dashboard listens on port 5006

CMD ["python", "src/main.py"]             # 7. command run when the container starts
```
Copying `requirements.txt` *before* the code is deliberate: Docker caches each step, so changing
your code does not force the libraries to be reinstalled.

### `docker-compose.yml` – how the container is run
```yaml
services:
  tents:
    build: .                 # build the image from the Dockerfile in this folder
    image: tents-in-sa
    container_name: tents-in-sa
    ports:
      - "5006:5006"          # your computer's port 5006 -> container's port 5006
    volumes:
      - ./data:/app/data     # share the data folder
      - ./outputs:/app/outputs   # share the results folder
```
- **Ports:** without `5006:5006` the dashboard would run inside the container but your browser could not reach it.
- **Volumes:** files written inside a container disappear when it stops. Sharing `outputs/` with your computer means
  the charts, tables and models are kept and you can open them directly. Sharing `data/` lets you replace or
  edit the CSV without rebuilding.

### `.dockerignore` – what is kept out of the image
Lists files Docker should not copy in (`.git`, `__pycache__`, `outputs/`, virtual environments), which keeps the image small and clean.

### What happens when you run `docker compose up --build`
```
docker compose up --build
   │
   ├─ 1. BUILD  the image (Python + libraries + project code)
   ├─ 2. START  a container from the image
   └─ 3. RUN    src/main.py inside the container:
            ├─ generate the dataset (only if data/tents_in_sa_rentals.csv is missing)
            ├─ analysis.py     → tables + charts in outputs/
            ├─ train_model.py  → models + metrics in outputs/
            └─ start the Panel dashboard on port 5006
                  ↓
        You open http://localhost:5006/app in your browser
```
Because everything runs inside the container, **the only thing you install on your own computer is Docker.**
No Python, no pip, no libraries.

### Useful Docker commands

| Command | What it does |
|---------|--------------|
| `docker compose up --build` | Build (if needed) and start the project |
| `docker compose up` | Start again without rebuilding |
| `docker compose down` | Stop and remove the container |
| `docker compose build --no-cache` | Rebuild the image from scratch |
| `docker ps` | List running containers |
| `docker images` | List images on your computer |
| `docker compose logs` | Show the container's output |

---

## 6. Dashboard guide
1. **Explore data:** filter by province and property type; see KPIs, charts and the data table.
2. **Rent predictor:** describe a property and get a suggested monthly rent.
3. **Renewal predictor:** describe a tenancy and get a renewal probability with advice.
4. **Model report:** cross-validation scores and the analysis charts.

---

## 7. Dataset columns
File: `data/tents_in_sa_rentals.csv` (100 rows)

| Column | Description |
|--------|-------------|
| `record_id` | Unique ID (TSA-001 … TSA-100) |
| `province`, `city` | Location of the property |
| `property_type` | Room, Flat, Cottage or House |
| `bedrooms`, `bathrooms`, `size_sqm` | Property size |
| `furnished`, `wifi`, `parking`, `security`, `backup_power` | Amenities (1 = yes, 0 = no) |
| `distance_to_centre_km` | Distance to the city centre |
| `tenant_type` | Student, Young Professional, Family, Contract Worker |
| `tenant_age`, `tenant_monthly_income_zar` | Tenant details |
| `lease_months`, `start_date` | Lease details |
| `monthly_rent_zar`, `deposit_zar` | Money (ZAR) |
| `tenant_rating` | Tenant satisfaction (1–5) |
| `late_payment` | 1 if the tenant paid late |
| `renewed` | 1 if the tenant renewed the lease |

---

## 8. Project layout
```
tents-in-sa/
├── Dockerfile               # how to build the environment
├── docker-compose.yml       # how to run it (ports + shared folders)
├── .dockerignore
├── requirements.txt         # pinned Python libraries
├── README.md
├── data/
│   └── tents_in_sa_rentals.csv
├── outputs/                 # created when you run the project
└── src/
    ├── config.py            # paths and constants
    ├── generate_data.py     # builds the synthetic dataset
    ├── analysis.py          # summary tables + charts
    ├── train_model.py       # trains and saves ML models
    ├── app.py               # Panel dashboard
    └── main.py              # container entry point (runs everything)
```

To regenerate the data, delete `data/tents_in_sa_rentals.csv` and run again, or change
`N_RECORDS` / `SEED` in `src/config.py`.

---

## 9. Troubleshooting

| Problem | Fix |
|---------|-----|
| `Cannot connect to the Docker daemon` / `docker: command not found` | Docker Desktop is not running or not installed. Start it and wait for "Engine running". |
| `port is already allocated` | Something else uses port 5006. In `docker-compose.yml` change `"5006:5006"` to `"5007:5006"` and open `http://localhost:5007/app`. |
| Browser shows "can't connect" right after starting | Wait for the `Dashboard starting` line: analysis and training run first. |
| `docker compose` not recognised | Older Docker: use `docker-compose up --build` (with a hyphen). |
| Windows: Docker won't start | Enable virtualisation in BIOS and make sure WSL 2 is installed (`wsl --install` in an admin terminal). |
| Changed the code but nothing changed | Run `docker compose up --build` so the image is rebuilt. |
| Want a completely fresh start | `docker compose down` then `docker compose build --no-cache`. |

---

## 10. Caveats
- Only 100 records: model scores are for demonstration, not for real pricing decisions.
- The renewal model is only modestly better than guessing (about 60% accuracy). That is realistic for a small,
  noisy sample and shows why more data would be needed in practice.
- Provinces with few records (for example Limpopo) give unstable averages.
