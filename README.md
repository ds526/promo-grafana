# promo-grafana
learning prometheus and grafana.

# Build the web-app
Done — I created a runnable learning project in a canvas titled **Prometheus-learning-webapp** (you can see the files and code there). It contains:

* a small Express backend that instruments metrics with `prom-client` and exposes `/metrics`
* a minimal React frontend that calls the backend and shows the raw `/metrics` output
* `prometheus.yml` and `docker-compose.yml` to run Prometheus locally
* package.json files and a README with quick-start commands

```
# Prometheus Learning Web App

This repository is a small full-stack learning project that demonstrates how to gather metrics from a web application and expose them for Prometheus to scrape. It includes:

- a simple Node/Express backend that records metrics using `prom-client` and exposes `/metrics`
- a React frontend (single-file app) that makes requests to the backend and displays some simple metrics and charts
- a `prometheus.yml` config and `docker-compose.yml` to run Prometheus locally scraping the backend
- a README with quick start instructions

---

## File: README.md

```md
# Prometheus Learning Web App

This is a small full-stack app for learning how to gather metrics and expose them to Prometheus.

## Requirements
- Node 18+ / npm
- Docker & docker-compose (optional, for running Prometheus)

## Quick start (local)

1. Install dependencies for backend & frontend:

```bash
# Install server deps
cd server
npm install

# Install frontend deps
cd ../client
npm install
```

2. Start the backend (from `server` folder):

```bash
node server.js
# or `npm start` if you prefer
```

The backend will run on port 4000 and expose `/metrics` and an example API at `/api/hello`.

3. Start the frontend (from `client` folder):

```bash
npm run dev
# or `npm start` depending on the setup
```

The frontend will run on port 3000 and has buttons to call the API; these calls will be instrumented by the backend and show up in `/metrics`.

## Run Prometheus with Docker Compose

A `docker-compose.yml` is provided. Run:

```bash
docker-compose up --build
```

Prometheus UI will be available at http://localhost:9090 and is configured to scrape `http://host.docker.internal:4000/metrics` (on macOS/Windows) or `http://localhost:4000/metrics` (on Linux) depending on your Docker networking — check `prometheus.yml`.

## What you can learn
- How to use `prom-client` to expose counters, histograms, and gauges
- How to instrument HTTP request durations and error counts
- How Prometheus scrapes `/metrics`
- How to visualise metrics in Prometheus and feed them to Grafana (optional)
```
```

---

## File: server/server.js

```js
// Minimal Express server that instruments metrics using prom-client
const express = require('express');
const client = require('prom-client');
const morgan = require('morgan');

const app = express();
const collectDefaultMetrics = client.collectDefaultMetrics;
collectDefaultMetrics({ prefix: 'myapp_' });

// Create custom metrics
const httpRequestDurationMicroseconds = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.005, 0.01, 0.05, 0.1, 0.3, 1, 3]
});

const processedJobs = new client.Counter({
  name: 'processed_jobs_total',
  help: 'Total number of processed jobs'
});

// simple logger
app.use(morgan('dev'));

// middleware to measure duration
app.use((req, res, next) => {
  const end = httpRequestDurationMicroseconds.startTimer();
  res.on('finish', () => {
    end({ route: req.path, method: req.method, status_code: res.statusCode });
  });
  next();
});

app.get('/api/hello', async (req, res) => {
  // simulate variable work
  const work = Math.random() * 500;
  await new Promise(r => setTimeout(r, work));
  processedJobs.inc();
  res.json({ msg: 'Hello from instrumented app', work_ms: work });
});

// metrics endpoint
app.get('/metrics', async (req, res) => {
  try {
    res.set('Content-Type', client.register.contentType);
    res.end(await client.register.metrics());
  } catch (ex) {
    res.status(500).end(ex);
  }
});

const port = process.env.PORT || 4000;
app.listen(port, () => console.log(`Server listening on ${port}`));
```

---

## File: client/src/App.jsx

```jsx
import React, { useState } from 'react';

export default function App() {
  const [result, setResult] = useState(null);
  const [metrics, setMetrics] = useState('');

  async function callApi() {
    const r = await fetch('/api/hello');
    const json = await r.json();
    setResult(json);
  }

  async function fetchMetrics() {
    const r = await fetch('/metrics');
    const text = await r.text();
    setMetrics(text);
  }

  return (
    <div className="p-6 max-w-3xl mx-auto">
      <h1 className="text-2xl font-bold mb-4">Prometheus Learning App</h1>

      <div className="mb-4">
        <button className="px-4 py-2 rounded border" onClick={callApi}>Call /api/hello</button>
        <button className="ml-3 px-4 py-2 rounded border" onClick={fetchMetrics}>Fetch /metrics</button>
      </div>

      <div className="mb-4">
        <h2 className="font-semibold">API result</h2>
        <pre>{result ? JSON.stringify(result, null, 2) : 'No result yet'}</pre>
      </div>

      <div>
        <h2 className="font-semibold">Raw /metrics</h2>
        <textarea rows={12} className="w-full mt-2 p-2 border" value={metrics} readOnly />
      </div>
    </div>
  );
}
```

---

## File: client/package.json

```json
{
  "name": "prom-learning-client",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "react": "18.2.0",
    "react-dom": "18.2.0"
  },
  "scripts": {
    "start": "vite",
    "build": "vite build",
    "dev": "vite"
  }
}
```

(Note: This example uses Vite for quick dev server.)

---

## File: server/package.json

```json
{
  "name": "prom-learning-server",
  "version": "1.0.0",
  "private": true,
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "prom-client": "^14.0.0",
    "morgan": "^1.10.0"
  }
}
```

---

## File: prometheus.yml

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: 'prom-learning-app'
    static_configs:
      - targets: ['host.docker.internal:4000'] # use localhost on linux if needed
```

---

## File: docker-compose.yml

```yaml
version: '3.8'
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports:
      - 9090:9090
```

---

## Notes & next steps
- You can extend the server to instrument database queries, background jobs, or custom business metrics
- Add middleware to instrument request sizes or response sizes
- Add Grafana in docker-compose and explore dashboards
- Add labels to metrics for richer querying

Have fun exploring Prometheus! If you'd like, I can:
- add Grafana + a sample dashboard to the repo
- add more example metrics (counters, gauges, summaries, histograms)
- convert the client to a full create-react-app / Vite template and provide a downloadable zip


# Prometheus Learning Web App

This repository is a small full-stack learning project that demonstrates how to gather metrics from a web application and expose them for Prometheus to scrape. It includes:

- a simple Node/Express backend that records metrics using `prom-client` and exposes `/metrics`
- a React frontend (single-file app) that makes requests to the backend and displays some simple metrics and charts
- a `prometheus.yml` config and `docker-compose.yml` to run Prometheus locally scraping the backend
- a README with quick start instructions

---

## File: README.md

```md
# Prometheus Learning Web App

This is a small full-stack app for learning how to gather metrics and expose them to Prometheus.

## Requirements
- Node 18+ / npm
- Docker & docker-compose (optional, for running Prometheus)

## Quick start (local)

1. Install dependencies for backend & frontend:

```bash
# Install server deps
cd server
npm install

# Install frontend deps
cd ../client
npm install
```

2. Start the backend (from `server` folder):

```bash
node server.js
# or `npm start` if you prefer
```

The backend will run on port 4000 and expose `/metrics` and an example API at `/api/hello`.

3. Start the frontend (from `client` folder):

```bash
npm run dev
# or `npm start` depending on the setup
```

The frontend will run on port 3000 and has buttons to call the API; these calls will be instrumented by the backend and show up in `/metrics`.

## Run Prometheus with Docker Compose

A `docker-compose.yml` is provided. Run:

```bash
docker-compose up --build
```

Prometheus UI will be available at http://localhost:9090 and is configured to scrape `http://host.docker.internal:4000/metrics` (on macOS/Windows) or `http://localhost:4000/metrics` (on Linux) depending on your Docker networking — check `prometheus.yml`.

## What you can learn
- How to use `prom-client` to expose counters, histograms, and gauges
- How to instrument HTTP request durations and error counts
- How Prometheus scrapes `/metrics`
- How to visualise metrics in Prometheus and feed them to Grafana (optional)
```
```

---

## File: server/server.js

```js
// Minimal Express server that instruments metrics using prom-client
const express = require('express');
const client = require('prom-client');
const morgan = require('morgan');

const app = express();
const collectDefaultMetrics = client.collectDefaultMetrics;
collectDefaultMetrics({ prefix: 'myapp_' });

// Create custom metrics
const httpRequestDurationMicroseconds = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.005, 0.01, 0.05, 0.1, 0.3, 1, 3]
});

const processedJobs = new client.Counter({
  name: 'processed_jobs_total',
  help: 'Total number of processed jobs'
});

// simple logger
app.use(morgan('dev'));

// middleware to measure duration
app.use((req, res, next) => {
  const end = httpRequestDurationMicroseconds.startTimer();
  res.on('finish', () => {
    end({ route: req.path, method: req.method, status_code: res.statusCode });
  });
  next();
});

app.get('/api/hello', async (req, res) => {
  // simulate variable work
  const work = Math.random() * 500;
  await new Promise(r => setTimeout(r, work));
  processedJobs.inc();
  res.json({ msg: 'Hello from instrumented app', work_ms: work });
});

// metrics endpoint
app.get('/metrics', async (req, res) => {
  try {
    res.set('Content-Type', client.register.contentType);
    res.end(await client.register.metrics());
  } catch (ex) {
    res.status(500).end(ex);
  }
});

const port = process.env.PORT || 4000;
app.listen(port, () => console.log(`Server listening on ${port}`));
```

---

## File: client/src/App.jsx

```jsx
import React, { useState } from 'react';

export default function App() {
  const [result, setResult] = useState(null);
  const [metrics, setMetrics] = useState('');

  async function callApi() {
    const r = await fetch('/api/hello');
    const json = await r.json();
    setResult(json);
  }

  async function fetchMetrics() {
    const r = await fetch('/metrics');
    const text = await r.text();
    setMetrics(text);
  }

  return (
    <div className="p-6 max-w-3xl mx-auto">
      <h1 className="text-2xl font-bold mb-4">Prometheus Learning App</h1>

      <div className="mb-4">
        <button className="px-4 py-2 rounded border" onClick={callApi}>Call /api/hello</button>
        <button className="ml-3 px-4 py-2 rounded border" onClick={fetchMetrics}>Fetch /metrics</button>
      </div>

      <div className="mb-4">
        <h2 className="font-semibold">API result</h2>
        <pre>{result ? JSON.stringify(result, null, 2) : 'No result yet'}</pre>
      </div>

      <div>
        <h2 className="font-semibold">Raw /metrics</h2>
        <textarea rows={12} className="w-full mt-2 p-2 border" value={metrics} readOnly />
      </div>
    </div>
  );
}
```

---

## File: client/package.json

```json
{
  "name": "prom-learning-client",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "react": "18.2.0",
    "react-dom": "18.2.0"
  },
  "scripts": {
    "start": "vite",
    "build": "vite build",
    "dev": "vite"
  }
}
```

(Note: This example uses Vite for quick dev server.)

---

## File: server/package.json

```json
{
  "name": "prom-learning-server",
  "version": "1.0.0",
  "private": true,
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "prom-client": "^14.0.0",
    "morgan": "^1.10.0"
  }
}
```

---

## File: prometheus.yml

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: 'prom-learning-app'
    static_configs:
      - targets: ['host.docker.internal:4000'] # use localhost on linux if needed
```

---

## File: docker-compose.yml

```yaml
version: '3.8'
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports:
      - 9090:9090
```

---

## Notes & next steps
- You can extend the server to instrument database queries, background jobs, or custom business metrics
- Add middleware to instrument request sizes or response sizes
- Add Grafana in docker-compose and explore dashboards
- Add labels to metrics for richer querying

Have fun exploring Prometheus! If you'd like, I can:
- add Grafana + a sample dashboard to the repo
- add more example metrics (counters, gauges, summaries, histograms)
- convert the client to a full create-react-app / Vite template and provide a downloadable zip


# Prometheus Learning Web App

This repository is a small full-stack learning project that demonstrates how to gather metrics from a web application and expose them for Prometheus to scrape. It includes:

- a simple Node/Express backend that records metrics using `prom-client` and exposes `/metrics`
- a React frontend (single-file app) that makes requests to the backend and displays some simple metrics and charts
- a `prometheus.yml` config and `docker-compose.yml` to run Prometheus locally scraping the backend
- a README with quick start instructions

---

## File: README.md

```md
# Prometheus Learning Web App

This is a small full-stack app for learning how to gather metrics and expose them to Prometheus.

## Requirements
- Node 18+ / npm
- Docker & docker-compose (optional, for running Prometheus)

## Quick start (local)

1. Install dependencies for backend & frontend:

```bash
# Install server deps
cd server
npm install

# Install frontend deps
cd ../client
npm install
```

2. Start the backend (from `server` folder):

```bash
node server.js
# or `npm start` if you prefer
```

The backend will run on port 4000 and expose `/metrics` and an example API at `/api/hello`.

3. Start the frontend (from `client` folder):

```bash
npm run dev
# or `npm start` depending on the setup
```

The frontend will run on port 3000 and has buttons to call the API; these calls will be instrumented by the backend and show up in `/metrics`.

## Run Prometheus with Docker Compose

A `docker-compose.yml` is provided. Run:

```bash
docker-compose up --build
```

Prometheus UI will be available at http://localhost:9090 and is configured to scrape `http://host.docker.internal:4000/metrics` (on macOS/Windows) or `http://localhost:4000/metrics` (on Linux) depending on your Docker networking — check `prometheus.yml`.

## What you can learn
- How to use `prom-client` to expose counters, histograms, and gauges
- How to instrument HTTP request durations and error counts
- How Prometheus scrapes `/metrics`
- How to visualise metrics in Prometheus and feed them to Grafana (optional)
```
```

---

## File: server/server.js

```js
// Minimal Express server that instruments metrics using prom-client
const express = require('express');
const client = require('prom-client');
const morgan = require('morgan');

const app = express();
const collectDefaultMetrics = client.collectDefaultMetrics;
collectDefaultMetrics({ prefix: 'myapp_' });

// Create custom metrics
const httpRequestDurationMicroseconds = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.005, 0.01, 0.05, 0.1, 0.3, 1, 3]
});

const processedJobs = new client.Counter({
  name: 'processed_jobs_total',
  help: 'Total number of processed jobs'
});

// simple logger
app.use(morgan('dev'));

// middleware to measure duration
app.use((req, res, next) => {
  const end = httpRequestDurationMicroseconds.startTimer();
  res.on('finish', () => {
    end({ route: req.path, method: req.method, status_code: res.statusCode });
  });
  next();
});

app.get('/api/hello', async (req, res) => {
  // simulate variable work
  const work = Math.random() * 500;
  await new Promise(r => setTimeout(r, work));
  processedJobs.inc();
  res.json({ msg: 'Hello from instrumented app', work_ms: work });
});

// metrics endpoint
app.get('/metrics', async (req, res) => {
  try {
    res.set('Content-Type', client.register.contentType);
    res.end(await client.register.metrics());
  } catch (ex) {
    res.status(500).end(ex);
  }
});

const port = process.env.PORT || 4000;
app.listen(port, () => console.log(`Server listening on ${port}`));
```

---

## File: client/src/App.jsx

```jsx
import React, { useState } from 'react';

export default function App() {
  const [result, setResult] = useState(null);
  const [metrics, setMetrics] = useState('');

  async function callApi() {
    const r = await fetch('/api/hello');
    const json = await r.json();
    setResult(json);
  }

  async function fetchMetrics() {
    const r = await fetch('/metrics');
    const text = await r.text();
    setMetrics(text);
  }

  return (
    <div className="p-6 max-w-3xl mx-auto">
      <h1 className="text-2xl font-bold mb-4">Prometheus Learning App</h1>

      <div className="mb-4">
        <button className="px-4 py-2 rounded border" onClick={callApi}>Call /api/hello</button>
        <button className="ml-3 px-4 py-2 rounded border" onClick={fetchMetrics}>Fetch /metrics</button>
      </div>

      <div className="mb-4">
        <h2 className="font-semibold">API result</h2>
        <pre>{result ? JSON.stringify(result, null, 2) : 'No result yet'}</pre>
      </div>

      <div>
        <h2 className="font-semibold">Raw /metrics</h2>
        <textarea rows={12} className="w-full mt-2 p-2 border" value={metrics} readOnly />
      </div>
    </div>
  );
}
```

---

## File: client/package.json

```json
{
  "name": "prom-learning-client",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "react": "18.2.0",
    "react-dom": "18.2.0"
  },
  "scripts": {
    "start": "vite",
    "build": "vite build",
    "dev": "vite"
  }
}
```

(Note: This example uses Vite for quick dev server.)

---

## File: server/package.json

```json
{
  "name": "prom-learning-server",
  "version": "1.0.0",
  "private": true,
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "prom-client": "^14.0.0",
    "morgan": "^1.10.0"
  }
}
```

---

## File: prometheus.yml

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: 'prom-learning-app'
    static_configs:
      - targets: ['host.docker.internal:4000'] # use localhost on linux if needed
```

---

## File: docker-compose.yml

```yaml
version: '3.8'
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports:
      - 9090:9090
```

---

## Notes & next steps
- You can extend the server to instrument database queries, background jobs, or custom business metrics
- Add middleware to instrument request sizes or response sizes
- Add Grafana in docker-compose and explore dashboards
- Add labels to metrics for richer querying

Have fun exploring Prometheus! If you'd like, I can:
- add Grafana + a sample dashboard to the repo
- add more example metrics (counters, gauges, summaries, histograms)
- convert the client to a full create-react-app / Vite template and provide a downloadable zip



