# Relay Setup Guide

Last checked: 30 Oct 2026.

This guide takes you from the zip file to your own live copy of Relay: your GitHub repo, a Neon database and a public URL. The free path takes about an hour.

## What you are setting up

Relay is a crash-safe runtime for AI agents: if a worker dies mid-run, another worker continues from the last saved step. It has three parts, and the always-running workers decide where you can host it.

```
                    Your host (one container on Render)
                   +----------------------------------------+
 Browser  -------> | API (FastAPI on uvicorn)               | ------> Neon Postgres
 The page at /     |   Serves the page, checks the key      |         All state lives here
 Live steps (SSE)  |   Writes a run row; never calls a model|         runs: the queue
                   |                                        |         steps: the ledger
                   | Worker x 2 (always running)            | <-----> companies: the output
                   |   Polls the database every 0.5 s       |
                   |   Claims a run, saves each step        |
                   |   If one dies, the other takes over    |
                   +----------------------------------------+
                                      |
                                      v  optional
                   Gemini and Tavily, used only after you add their keys
```

The API only writes a run row. A worker picks that row up from Postgres and does the work. All state lives in the database, so the container can sleep or be replaced without losing anything.

**Accounts and tools you need**

| What | Why | Cost | Needed for |
| --- | --- | --- | --- |
| GitHub | Holds your code and runs the tests on every push | Free | Everything |
| Python 3.13 or newer, Git | Run the project and its setup scripts | Free | Everything |
| Docker Desktop | A local Postgres for Step 2 | Free for personal use | Step 2 |
| [Neon](https://neon.com/docs/introduction/plans) | The hosted Postgres database | Free: 100 CU-hours per month, 1 GB | Deploying |
| [Render](https://render.com/docs/free) | Runs the container | Free: 750 hours per month | Deploying |
| [Google AI Studio](https://aistudio.google.com/apikey) | A Gemini key for real AI runs | Free tier | Optional (Step 7) |
| Tavily | Real web search | Free tier | Optional (Step 7) |

The default setup uses a built-in scripted model, so it makes no paid calls and needs no AI key.

## Step 1: Put the code on your own GitHub

The zip holds the code with no git history, so your repo starts clean.

1. Unzip it somewhere with a short path, such as `C:\dev\relay` on Windows. A very long path makes `pip install` fail with "No such file or directory".
2. Point the project's links at your repo. Find and replace `sakshiroy2026/relay` with `YOUR-USERNAME/relay` in `relay/api/static/index.html` (5 links) and `learn/DEPLOY.md` (1 link).
3. Create an empty repo at [github.com/new](https://github.com/new), named `relay`. Don't add a README, .gitignore or license.
4. Push the code:

```bash
cd relay
git init -b main
git add .
git commit -m "Initial commit of Relay"
git remote add origin https://github.com/YOUR-USERNAME/relay.git
git push -u origin main
```

On Windows, the "LF will be replaced by CRLF" warnings are harmless. `.gitignore` already keeps `.env` and `.venv` out of the repo.

Open the repo's **Actions** tab. The `ci` workflow runs on every push (formatting, lint, types, 292 tests, eval gate) and needs no secrets. Wait for it to go green.

## Step 2: Run it on your laptop

This takes about 15 minutes, costs nothing, and proves everything works before you touch the cloud. Run these from the project folder.

**Install**

```bash
python --version
python -m venv .venv
```

The version must be 3.13 or newer. Then activate the environment. You'll need it active in every terminal below.

- Windows PowerShell: `.venv\Scripts\Activate.ps1`
- Mac or Linux: `source .venv/bin/activate`

If PowerShell blocks the script, run `Set-ExecutionPolicy -Scope Process Bypass` in that terminal first.

```bash
pip install -e ".[dev]"
```

**Start a local database**

```bash
docker run -d --name relay-pg -e POSTGRES_USER=relay -e POSTGRES_PASSWORD=relay -e POSTGRES_DB=relay -p 5433:5432 postgres:16
```

**Create a file named `.env` in the project folder**

Leave `DEV_TENANT_ID` blank for now. Don't copy `.env.example`, which is for the VM setup.

```
DATABASE_URL=postgresql://relay:relay@127.0.0.1:5433/relay
DEV_TENANT_ID=
LEASE_SECONDS=15
FAKE_DELAY_SECONDS=2.5
```

**Create the tables and a demo account**

```bash
alembic upgrade head
python -m scripts.seed --tenant demo --mode demo
```

`alembic` prints nothing when it succeeds. The seed prints a tenant id and a key, and the key is shown only once:

```
tenant 'demo': id 56cf11ee-0000-0000-0000-000000000000
RELAY_API_KEY=rly_xxxxxxxx.xxxxxxxxxxxxxxxx
```

Fill in `DEV_TENANT_ID` with that id, and add this line to `.env` with that key:

```
PUBLIC_DEMO_KEY=rly_xxxxxxxx.xxxxxxxxxxxxxxxx
```

**Run it, in three terminals**

```bash
uvicorn relay.api.main:app --port 8000
```

```bash
python -m relay.worker
```

```bash
python -m relay.worker
```

The workers warn that `GEMINI_API_KEY` and `TAVILY_API_KEY` aren't set. That's expected.

**Try it**

1. Open http://localhost:8000 and click `razorpay.com`. Steps appear live, and the run finishes in about 10 seconds.
2. Start another run and press **Kill the worker** straight away. After about 15 to 30 seconds, the other worker takes over and finishes the run.
3. Open http://localhost:8000/ops for the dashboard. The spend shown there is simulated, not a real bill.

**Run the tests**

Stop the two workers with Ctrl+C, then run:

```bash
python -m pytest tests/ -q
```

Expect `292 passed` in about a minute. The tests write to the database named in `.env`, so that file must always point at your local database, never at Neon.

## Step 3: Create the Neon database

Neon is your production database. You set it up once from your laptop, then give its details to your host.

1. Sign up at [neon.com](https://neon.com) and create a project. Pick Postgres 16 if offered, because that's the version CI tests.
2. Pick a region close to your host: AWS Singapore pairs with Render's Singapore region, and AWS US East (N. Virginia) pairs with Vercel's default region.
3. Click **Connect**, leave **Connection pooling** on, and copy the connection string. Its host contains `-pooler`. That's correct, because Relay removes `-pooler` itself for the one connection that needs it (`relay/api/events.py`).
4. With your environment active, create the tables and the demo account on Neon.

Mac, Linux or Git Bash:

```bash
export DATABASE_URL='postgresql://USER:PASSWORD@HOST-pooler.REGION.aws.neon.tech/neondb?sslmode=require'
alembic upgrade head
python -m scripts.seed --tenant demo --mode demo
unset DATABASE_URL
```

Windows PowerShell:

```powershell
$env:DATABASE_URL = 'postgresql://USER:PASSWORD@HOST-pooler.REGION.aws.neon.tech/neondb?sslmode=require'
alembic upgrade head
python -m scripts.seed --tenant demo --mode demo
Remove-Item Env:DATABASE_URL
```

The temporary variable overrides `.env` for that terminal only, so `.env` stays pointed at your local database.

Save the tenant id and key that the seed prints. You'll paste them into your host in Step 5. These are new values, different from the ones in your local `.env`.

## Step 4: Choose where to deploy

Use Render unless you have a reason not to: it's free, needs no code changes, and the original deployment already runs this way.

| Platform | Status | Cost | Main catch |
| --- | --- | --- | --- |
| **Render** (recommended) | Proven: the original runs here | Free | Sleeps after 15 minutes idle, and the next visit waits about a minute |
| **Vercel** | Should work using beta features, but untested | Free for part of each month, otherwise Pro | One worker uses up the free quota in about 17 days, then stops until the next month |
| **Other Docker hosts or a VM** | Should work with the same image, but untested | Varies by host | If the host never sleeps, Neon's free compute runs out in about 17 days |

**Why a normal Vercel deploy won't work.** Vercel Functions only run while handling a request, up to [300 seconds on the free plan](https://vercel.com/docs/functions/limitations). Relay's workers have to run all the time, so they need Vercel's separate [Daemons](https://vercel.com/docs/daemons) feature, which is in public beta.

**Where the 17 days comes from.** Both figures are worked out from the published limits, not measured:

- Vercel's free plan allows [420 GB-hours of daemon memory a month](https://vercel.com/docs/daemons/limits). One 1 GiB worker reaches that in 420 hours.
- Neon's free plan allows [100 CU-hours a month](https://neon.com/docs/introduction/plans). At the smallest size, 0.25 CU, that's 400 hours. Workers query the database every 0.5 seconds, so Neon should never get to scale down while they run.

Render's free plan avoids the Neon limit, because the workers sleep along with the service. For the same reason, don't add an uptime pinger to keep it awake.

## Step 5A: Deploy on Render

Render builds one container holding the API and two workers, and rebuilds it on every push to `main`.

1. Sign in at [render.com](https://render.com) with GitHub, then choose **New > Web Service** and pick your `relay` repo.
2. Use these settings. Dashboard labels may differ slightly.

| Setting | Value |
| --- | --- |
| Language | Docker |
| Branch | `main` |
| Region | The one closest to your Neon project |
| Instance type | Free |
| Dockerfile Path | `./deploy/single/Dockerfile` |
| Docker Build Context | `.` |
| Health Check Path | `/healthz` |

3. Add the environment variables, using the Neon values from Step 3:

| Variable | Value |
| --- | --- |
| `DATABASE_URL` | Your Neon connection string |
| `DEV_TENANT_ID` | The tenant id the seed printed |
| `PUBLIC_DEMO_KEY` | The key the seed printed |

4. Click **Create Web Service**. The first deploy takes several minutes, because the free plan has very little CPU (about 0.1).

The image already sets `LEASE_SECONDS=15` and `FAKE_DELAY_SECONDS=2.5`. Its start script, `deploy/single/start.sh`, runs any new migrations, starts the workers and restarts one if it dies.

Your URL is `https://YOUR-SERVICE.onrender.com`. Go to Step 6 to check it.

## Step 5B: Deploy on Vercel (untested)

This setup has not been run on Vercel, so expect to adjust it. The two images below were tested together in local Docker and completed a run. Vercel's Services and Daemons are both in beta.

1. In Step 3, put Neon in AWS US East (N. Virginia), next to Vercel's `iad1` region.
2. Add a file named `Dockerfile.api` to the repo root. The existing root `Dockerfile` only starts a worker, so importing the repo without this file gives you a site with no web server.

```dockerfile
FROM python:3.13-slim
ENV PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1
WORKDIR /app
COPY pyproject.toml ./
COPY relay ./relay
RUN pip install --no-cache-dir .
CMD ["sh", "-c", "exec uvicorn relay.api.main:app --host 0.0.0.0 --port ${PORT:-80} --proxy-headers --forwarded-allow-ips '*'"]
```

3. Add a file named `vercel.json` to the repo root:

```json
{
  "$schema": "https://openapi.vercel.sh/vercel.json",
  "services": {
    "api": { "runtime": "container", "root": "./", "entrypoint": "Dockerfile.api" }
  },
  "rewrites": [
    { "source": "/(.*)", "destination": { "service": "api" } }
  ],
  "daemons": {
    "worker": {
      "root": "./",
      "entrypoint": "Dockerfile",
      "instances": { "iad1": 1 },
      "resources": { "requests": { "memory": "1Gi" } }
    }
  }
}
```

4. Push, then import the repo at [vercel.com](https://vercel.com) with the framework preset set to **Services**.
5. Add these environment variables for Production: `DATABASE_URL`, `DEV_TENANT_ID` and `PUBLIC_DEMO_KEY` from Step 3, plus `LEASE_SECONDS=15` and `FAKE_DELAY_SECONDS=2.5`.
6. Deploy to production, because daemons don't run on preview deployments. Check the project's logs for the worker's `started` line.

**How this differs from Render**

- Migrations don't run by themselves. When a new one arrives, run the `alembic upgrade head` commands from Step 3 again.
- A stream is cut at 300 seconds on the free plan, and the page then switches to polling.
- With one worker, **Kill the worker** waits for Vercel to restart it. Two instances take over faster but use the memory quota twice as fast, about 9 days.
- To save quota when you aren't demoing, set the worker's instance count to `0`.

On the free plan, reaching a quota [stops the daemons until the first of the next month](https://vercel.com/docs/daemons/limits). An idle worker measured about 0.9% of a CPU locally, which is about 6.6 CPU-hours a month against a 5-hour quota. So the CPU quota could also run out, at about 23 days.

## Step 5C: Deploy on another Docker host

Any host that builds a Dockerfile from your repo and keeps it running can use the same image as Render. Railway, Fly.io and Koyeb are common choices, but none has been tested with Relay, and their free plans change often, so check their pricing pages.

- **Dockerfile:** `deploy/single/Dockerfile`, with the repo root as the build context.
- **Environment variables:** `DATABASE_URL`, `DEV_TENANT_ID` and `PUBLIC_DEMO_KEY`.
- **Port:** the app uses `$PORT` if the host sets it, otherwise 10000.
- **Health check:** `/healthz`.

**On your own server:** use `docker-compose.prod.yml`, which runs the API, three workers and Caddy for HTTPS. The steps are in `learn/DEPLOY.md`. These files were written for an Oracle Cloud VM and are marked in that guide as never built, so expect small fixes.

If the host never sleeps, Neon's free compute won't last the month (Step 4). You can pay for Neon or run your own Postgres on the server.

## Step 6: Check that it works

Your deployment is working when all of these pass. Replace `YOUR-URL` with your site's address.

| Check | Expected result |
| --- | --- |
| `YOUR-URL/healthz` | `{"status":"ok"}` |
| `YOUR-URL/readyz` | `{"status":"ok","db":"ok"}`, which proves the database connection |
| `YOUR-URL/demo-config` | `demo_key` has a value, not `null` |
| `YOUR-URL/` | Click `razorpay.com`: steps stream in and the run succeeds |
| **Kill the worker** during a run | Another worker takes over and finishes, and a banner says it resumed |
| Copy the page's URL (`YOUR-URL/#run=...`) into a private window | The same run opens with no key entered |
| `YOUR-URL/ops` | The dashboard shows your runs and "effects committed twice" at 0 |

From now on, every push to `main` redeploys on Render.

## Step 7 (optional): Turn on real AI

The site works fully without this. Adding a Gemini key puts a **Run with real AI** option on the page, using a real model with canned search results, so no visitor can make your server fetch a URL.

1. Create a key at [Google AI Studio](https://aistudio.google.com/apikey). Keep the project on the free tier, and don't enable billing.
2. Add these variables on your host, then redeploy:

| Variable | Value | Why |
| --- | --- | --- |
| `GEMINI_API_KEY` | Your key | Never commit it or put it in a URL |
| `GEMINI_MODEL` | `gemini-3.5-flash-lite` | Pins the model, so a default change can't move it silently |
| `PUBLIC_REAL_RUNS_PER_DAY` | `30` | The allowance all visitors share each day; `0` turns the option off |

3. Open your site and tick **Run with real AI**. Runs are spaced 45 seconds apart to respect the free tier's per-minute limit, and the page waits its turn.

Good to know:

- **Running locally:** the app reads `GEMINI_API_KEY` from the real environment, so putting it in `.env` isn't enough. Set it in the terminal instead, as in Step 3.
- **Slower models:** the image's 15-second lease keeps the kill demo short and suits Flash-Lite. A slower model can outlast the lease, and another worker would then repeat the call. Raise `LEASE_SECONDS` above 60 if you switch.
- **Real web search:** this needs a `TAVILY_API_KEY` and a private tenant, created with `python -m scripts.seed --tenant me --mode live`. Keep that key to yourself. Never use it as `PUBLIC_DEMO_KEY`, though the API refuses to offer it anyway.

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `pip install` fails with "No such file or directory" on Windows | The folder path is too long | Move the project to a short path such as `C:\dev\relay` |
| `pip` says the Python version isn't supported | Python is older than 3.13 | Install 3.13 and recreate `.venv` |
| Render build fails | Dockerfile path not set | Set it to `./deploy/single/Dockerfile` under Settings > Build & Deploy |
| First visit hangs for about a minute | The free plan is waking up | Expected |
| `/readyz` returns 503 | `DATABASE_URL` is wrong or Neon is unreachable | Check the string, then the host's logs |
| API is up but runs stay `pending` | Workers have died | Look for `worker N exited` in the logs, then check `DATABASE_URL` and `DEV_TENANT_ID` |
| `/demo-config` shows `demo_key: null` | `PUBLIC_DEMO_KEY` is missing or wrong, or its tenant isn't in demo mode | Re-check the key from Step 3 |
| Page says "API key missing or wrong" | Your browser saved an old key, or no demo key is on offer | Clear the key box under the form, then check `/demo-config` |
| Seed says "api key already exists" and you lost the key | Keys can't be recovered | Run the seed with `--tenant demo2 --mode demo`, then update `DEV_TENANT_ID` and `PUBLIC_DEMO_KEY` |
| No "Run with real AI" option | `GEMINI_API_KEY` is missing or `PUBLIC_REAL_RUNS_PER_DAY` is 0 | Set both and redeploy |
| Real-AI runs fail with `model_misconfigured` | The workers can't see `GEMINI_API_KEY` | Add it on the host and redeploy |
| Real-AI runs fail with `model_fatal: daily quota exhausted` | The free tier's daily quota is spent. It's per Google project, not per key | Wait until midnight US Pacific, or lower `PUBLIC_REAL_RUNS_PER_DAY` |

## Security rules

- **Keep secrets out of git.** `.env` is ignored already. Database strings and real API keys belong only in `.env` or your host's settings.
- **The demo key is public on purpose.** `/demo-config` hands it to every visitor. Its tenant runs the free scripted model, and the API refuses to offer a key whose tenant is in `live` mode.
- **Don't run tests or the `chaos/` harnesses against Neon.** They write a lot of data. Keep `.env` on your local database.
- **Rotate anything you leak.** Reset the password in Neon, or create a new key, then update your host.

## Before you publish your copy

- **Links:** after Step 1, update the live link in `README.md` line 55 with your own URL. Older mentions in `learn/` are historical notes and can stay.
- **Results and run ids:** the numbers and run ids in `README.md` and `learn/` were measured on the original deployment. They won't exist in your database. Say where they were measured, or re-run them yourself before claiming them.
- **`evals/results.json`:** keep this file, because it holds the eval results the CI gate checks. Without it, the gate skips the check.
- **License:** the repo has no license file. Add one if you want others to be able to reuse the code.

## What was tested

All of this was tested on 30 Oct 2026 on a Windows laptop, using a fresh unzip of this package.

| Part | How it was tested |
| --- | --- |
| Steps 1 and 2 | Followed in full, with a local bare repo in place of GitHub: demo run, tests and eval gate all passed |
| Step 3 | Run with the temporary variable against a second local database in place of Neon |
| Render's image | Built and run in local Docker: page, kill and takeover, saved link and `/ops` all worked |
| Vercel's two images | Built and run in local Docker: one run succeeded, but nothing ran on Vercel itself |
| Not tested | Sign-up screens on Neon, Render and Vercel, a fresh real Neon database, and real AI |

The original deployment already runs on Render and Neon, so that path is proven in production. Only its setup screens weren't walked through again.

## Sources

Platform limits were checked on 30 Oct 2026 and can change, so confirm them before relying on them.

- [Render: free instances](https://render.com/docs/free)
- [Neon: plans](https://neon.com/docs/introduction/plans)
- [Vercel: Functions limits](https://vercel.com/docs/functions/limitations)
- [Vercel: Daemons](https://vercel.com/docs/daemons)
- [Vercel: Daemons limits](https://vercel.com/docs/daemons/limits)
- [Vercel: Services](https://vercel.com/docs/services)
- [Vercel: Running Docker](https://vercel.com/kb/guide/docker)
- [Google: Gemini API keys](https://ai.google.dev/gemini-api/docs/api-key)
