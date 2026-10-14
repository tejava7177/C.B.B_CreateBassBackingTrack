# C.B.B – Create Bass BackingTrack 🎸
Real-time tuner (FFT) · Chord progression generation (LSTM) · Automatic accompaniment (MIDI/score/WAV) · Web service

**Deployment:** https://cbbunivproject.store  · **API Docs:** https://cbbunivproject.store/docs

---

## Table of Contents
- [Project Overview](#project-overview)
- [Key Features](#key-features)
- [System Architecture](#system-architecture)
- [Tech Stack](#tech-stack)
- [Quick Start (Local)](#quick-start-local)
- [Docker Deployment](#docker-deployment)
- [AWS Deployment Summary](#aws-deployment-summary)
- [Environment Variables](#environment-variables)
- [API Overview](#api-overview)
- [Model / Data](#model--data)
- [Troubleshooting](#troubleshooting)
- [License & Credits](#license--credits)

---

## Project Overview
C.B.B (Create Bass BackingTrack) is an **all-in-one web tool for guitar and bass players**.  
Users can **tune their instruments** through microphone input in the browser, receive **genre-specific chord progression recommendations**, and generate **automatic accompaniment** (drums, comping, and bass). The generated results can be downloaded as **MIDI / MusicXML / WAV** files.

### Main Page
![ui-home.png](docs/ui-home.png)
### System Architecture
![architecture.png](docs/architecture.png)

- **User (HTTPS)** ⇄ **Nginx** (static files & `/api` proxy) ⇄ **FastAPI**  
- FastAPI ⇄ **PyTorch (LSTM)**, **SongMaker rule engine**, **FluidSynth**  
- Outputs: MIDI/WAV/XML files are stored in the `/recordings` volume

---

## Tech Stack
<p align="center">
  <img src="https://skillicons.dev/icons?i=react,ts,vite,python,fastapi,pytorch,numpy,pandas,docker,nginx,aws,ubuntu,git,github&perline=7" />
</p>

**Frontend**: Vite (React/TS) – fast HMR and a productive build workflow  
**Backend**: FastAPI – type-based design, automatic documentation, and asynchronous performance  
**ML**: PyTorch – convenient LSTM implementation/inference and CPU-friendly deployment  
**Music**: `music21`, `pretty_midi` – standard libraries for MIDI and score processing  
**Audio**: FluidSynth + **GeneralUserGS.sf2** – lightweight and reproducible audio rendering  
**Infra**: Docker + Nginx + EC2 – low-cost deployment with reproducible container images  
**Monitoring**: CloudWatch Logs – centralized logging and alerts

---

## Key Features
1) **Tuner (FFT)**  
   - WebAudio + `getUserMedia` (HTTPS required) → **real-time FFT-based fundamental frequency estimation**  
   - Standard guitar/bass tuning presets with noise suppression and smoothing
   ![ui-tuner.png](docs/ui-tuner.png)

2) **Chord Progression Generation (LSTM + Rule Blending)**  
   - Trains an LSTM on genre-specific normalized chord-token datasets  
   - Suggests **1 standard progression + 2 alternative progressions** using Top-K candidates  
   - Applies **MMR-based diversity selection** + **bucketized (high/mid/low)** score calibration
   ![chordSeed.png](docs/chordSeed.png)
   ![ui-result.png](docs/ui-result.png)

3) **Audio / Score Generation**  
   - Combines genre presets (tempo, groove, instruments) with rhythm patterns  
   - Generates MIDI / MusicXML using `music21` and `pretty_midi`
   ![score.png](docs/score.png)
   ![reuslt-midi.png](docs/reuslt-midi.png)

4) **MIDI → WAV Rendering (FluidSynth)**  
   - Includes **GeneralUserGS.sf2** in the container for reproducible rendering  
   - Upload via `POST /api/render/midi-to-wav` → returns a WAV file
![ui-render.png](docs/ui-render.png)

5) **Deployment / Operations**  
   - Nginx reverse proxy (static content and `/api` separated), Let’s Encrypt (TLS), Docker Compose

---

## Quick Start (Local)
```bash
# 1) Check dependencies
docker -v
docker compose version

# 2) Prepare .env (project root)
cp .env.example .env
# Verify values using container-internal paths (see [Environment Variables] below)

# 3) Build frontend (static files)
cd frontend
npm ci
npm run build
cd ..

# 4) Start containers
docker compose up -d --build

# Frontend: http://localhost
# Backend Swagger: http://localhost/docs
```

---

## Docker Deployment
```bash
# Build images and start services
docker compose up -d --build

# Status / logs
docker compose ps
docker compose logs -f cbb-api
docker compose logs -f nginx

# Redeploy
docker compose up -d --force-recreate
```

**Example Nginx volumes**
- `./frontend/dist:/usr/share/nginx/html:ro` – serves static files
- `./nginx/conf.d:/etc/nginx/conf.d:ro` – proxy configuration

---

## AWS Deployment Summary
- **EC2 (t3.micro)** instance, security groups: 80/443 (HTTP/HTTPS), 22 (SSH)  
- Install **Docker / Compose** → sync repository to `/opt/New_LSTM`  
- Issue TLS certificates with **Let’s Encrypt** (certbot webroot)  
- **CloudWatch Logs**: `awslogs` driver + IAM Role

---

## Environment Variables
> **Important:** Paths are based on the **container filesystem**. (Model files and the SoundFont are included in the image.)

`.env` (project root)
```
# Model directories (container paths)
CBB_MODEL_JAZZ=/app/app/assets/model/jazz
CBB_MODEL_ROCK=/app/app/assets/model/rock
CBB_MODEL_POP=/app/app/assets/model/pop

# Temporary LSTM data (JSON) path
CBB_DATA_DIR=/app/LSTM/cli/data

# Output directory
CBB_RECORDINGS_DIR=/recordings

# SoundFont paths (shared by REST rendering and score generation)
SF2_PATH=/app/app/assets/sf2/GeneralUserGS.sf2
CBB_SOUNDFONT_PATH=/app/app/assets/sf2/GeneralUserGS.sf2
```

---

## API Overview
- **Swagger**: `/docs`
- Examples:  
  - `POST /api/chords/topk` : genre + seed input → top 3 progressions  
  - `POST /api/tracks/generate` : progression/options → generate MIDI/XML  
  - `POST /api/render/midi-to-wav` : upload MIDI → return WAV  
> Refer to Swagger for the exact parameters and schemas.

---

## Model / Data
- **Preprocessing**: normalize genre-specific chord symbols into **root / type / extensions** → `.npy(X,y)`  
- **Model**: LSTM (sequence prediction), CPU inference  
- **Post-processing**: rule-based candidates + **MMR-based diversity selection**  
- **Score calibration**: percentile adjustment into high/mid/low groups using `bucketize_three` for improved readability

---

## Troubleshooting
- **404 at root (`/`)** → redirect root to `/docs` in Nginx or verify the static root configuration (`root /usr/share/nginx/html;`)  
- **Microphone device list does not appear** → **HTTPS is required** by browser security policies  
- **`python-multipart` error** → verify that it is included in `requirements.txt`  
- **`fluidsynth` is not installed** → run `apt-get install -y fluidsynth libsndfile1` inside the container  
- **SoundFont not found** → verify that `SF2_PATH` in `.env` points to a **container-internal path**  
- **CloudWatch logs are not collected** → verify `awslogs-region/group/stream` and attach an EC2 IAM Role with the minimum required CloudWatch Logs permissions  
- **Compose warning: version obsolete** → remove the `version:` key (unnecessary in Compose v2)

---

## License & Credits
- License: MIT  
- SoundFont: **GeneralUserGS.sf2** (subject to its respective license)  
- Thanks: PyTorch, FastAPI, music21, pretty_midi, FluidSynth, Vite/React, AWS, and the open-source community 🙏
