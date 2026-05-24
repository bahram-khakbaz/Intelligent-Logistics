# SKY-DX Intelligent Logistics, Geo-Spatial & Routing Infrastructure

This repository contains the complete enterprise-grade backend infrastructure, data-processing pipelines, and Android client application for the **DigiExpress** distribution and smart logistics routing network. The system processes spatial data, provides high-performance multi-criteria vehicle routing (VRP), reverse geocoding, real-time fleet monitoring, and live voice-guided in-app navigation.

---

## 🏗️ Architecture Overview

The infrastructure follows a decoupled microservices architecture designed to maintain high availability and low latency under high concurrent load:


```

```
              [ Fleet Android Application ]
                            │
                    (Port 80 HTTP)
                            ▼
                    [ Nginx Gateway ]
                            │
     ┌──────────────────────┼──────────────────────┐
     ▼                      ▼                      ▼

```

[ / (FastAPI) ]        [ /vroom/ (VRP) ]       [ /osrm/ (Routing) ]
(Python Backend)     (Optimization Engine)       (OSRM CH Engine)
│                      │                      │
┌─────┴─────┐                └──────────┬───────────┘
▼           ▼                           ▼
[PostgreSQL] [Redis]                 [OSRM Data Files]
(PostGIS)   (Cache)                (iran-260404.osrm)

```

1. **Edge Gateway (Nginx):** Acts as the primary entry point, handling path-based routing, connection pooling, and extended request timeouts.
2. **API & Orchestration Layer (FastAPI):** Python 3.12 gateway utilizing asynchronous engines to clean, enrich, and transform coordinate points into valid logical postal contexts.
3. **Spatial Database Engine (PostgreSQL + PostGIS + pgRouting):** High-performance transactional geo-database heavily optimized for hardware utilization (shared buffers, parallel workers).
4. **Caching Layer (Redis 7):** Handles LRU context eviction and micro-caching of routing tables and high-frequency reverse-geocoding coordinates.
5. **Core Routing & Churning Engines (OSRM & VROOM):** High-speed point-to-point Contraction Hierarchies (CH) path matrix calculation linked directly into a Vehicle Routing Problem (VRP) solver.
6. **Telemetry & Monitoring Layer (Prometheus & Grafana):** Automated instrumented metric gathering tracking routing workloads down to 5-second intervals.

---

## 🛠️ Server Environment Provisioning

Follow these precise steps to bootstrap a fresh Ubuntu server into an operational enterprise node.

### 1. Upgrade System & Install Fundamental Packages
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl gnupg lsb-release ca-certificates git build-essential htop python3-pip python3-venv

```

### 2. Install Docker Engine & Compose Plugin

```bash
# Add Docker's official GPG key
sudo fold -w 80 /etc/apt/keyrings
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL [https://download.docker.com/linux/ubuntu/gpg](https://download.docker.com/linux/ubuntu/gpg) | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Set up the repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] [https://download.docker.com/linux/ubuntu](https://download.docker.com/linux/ubuntu) \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Verify installation
sudo docker --version && sudo docker compose version

```

### 3. File System Layout Blueprint

Execute the following structure under `/opt`:

```bash
sudo mkdir -p /opt/logistics-stack/{backend,nginx,osrm,osm_packages,vroom,prometheus,grafana}
sudo mkdir -p /opt/osrm-data
sudo chown -R $USER:$USER /opt/logistics-stack /opt/osrm-data

```

---

## 🗺️ Automated OSRM Map Processing Pipeline

To automate downloading, filtering, splitting, and compiling raw OpenStreetMap map primitives (`.osm.pbf`) into highly optimized Contraction Hierarchies (CH) binary files, use the following Python automation script.

### 1. The Pipeline Automation Script (`process_map.py`)

Save this file as `/opt/logistics-stack/process_map.py`. It automates the extraction profile execution and cuts down compilation time.

```python
#!/usr/bin/env python3
import os
import subprocess
import sys

DATA_DIR = "/opt/osrm-data"
PBF_FILE = "iran-260404.osm.pbf"
OSRM_FILE = "iran-260404.osrm"
IMAGE_NAME = "mirror2.chabokan.net/osrm/osrm-backend:latest"

def run_command(command, description):
    print(f"\n🚀 Running: {description}...")
    print(f"Executing: {' '.join(command)}")
    process = subprocess.Popen(command, stdout=subprocess.PIPE, stderr=subprocess.STDOUT, text=True)
    
    while True:
        output = process.stdout.readline()
        if output == '' and process.poll() is not null:
            break
        if output:
            print(output.strip())
            
    rc = process.poll()
    if rc != 0:
        print(f"❌ Error during: {description}. Exit code: {rc}")
        sys.exit(rc)
    print(f"✅ Completed: {description}")

def main():
    if not os.path.exists(os.path.join(DATA_DIR, PBF_FILE)):
        print(f"❌ Target map primitive data file not found at: {os.path.join(DATA_DIR, PBF_FILE)}")
        sys.exit(1)

    # Step 1: Extract graph layout using the default vehicle/car profile
    extract_cmd = [
        "docker", "run", "--rm",
        "-v", f"{DATA_DIR}:/data",
        IMAGE_NAME,
        "osrm-extract", "-p", "/profile/car.lua", f"/data/{PBF_FILE}"
    ]
    run_command(extract_cmd, "OSRM Graph Network Extraction (osrm-extract)")

    # Step 2: Partition the extracted graph segments
    partition_cmd = [
        "docker", "run", "--rm",
        "-v", f"{DATA_DIR}:/data",
        IMAGE_NAME,
        "osrm-partition", f"/data/{OSRM_FILE}"
    ]
    run_command(partition_cmd, "OSRM Cell Partitioning (osrm-partition)")

    # Step 3: Customize cells and compute contraction hierarchies
    customize_cmd = [
        "docker", "run", "--rm",
        "-v", f"{DATA_DIR}:/data",
        IMAGE_NAME,
        "osrm-customize", f"/data/{OSRM_FILE}"
    ]
    run_command(customize_cmd, "OSRM Contraction Hierarchies Customization (osrm-customize)")

    print("\n🎉 OSRM Pipeline Processing Completed Successfully! Data files are ready for container runtime initialization.")

if __name__ == "__main__":
    main()

```

Run the automated pipeline script:

```bash
chmod +x /opt/logistics-stack/process_map.py
python3 /opt/logistics-stack/process_map.py

```

---

## ⚙️ Core Server Configuration Files

### 1. `docker-compose.production.yml`

Save the configuration snippet below directly into `/opt/logistics-stack/docker-compose.production.yml`:

```yaml
version: "3.9"

networks:
  routing_net:
    driver: bridge

volumes:
  postgres_data:
  redis_data:
  grafana_data:
  prometheus_data:

x-logging: &default-logging
  logging:
    driver: "json-file"
    options:
      max-size: "50m"
      max-file: "3"

services:

  postgis:
    image: mirror2.chabokan.net/pgrouting/pgrouting:15-3.5-4.0
    container_name: postgis
    restart: always
    environment:
      POSTGRES_DB: irandb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: StrongPassword123
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    shm_size: 8gb
    command: >
      postgres
      -c shared_buffers=8GB
      -c effective_cache_size=24GB
      -c maintenance_work_mem=2GB
      -c checkpoint_completion_target=0.9
      -c wal_buffers=16MB
      -c default_statistics_target=100
      -c random_page_cost=1.1
      -c effective_io_concurrency=200
      -c work_mem=64MB
      -c min_wal_size=1GB
      -c max_wal_size=4GB
      -c max_worker_processes=12
      -c max_parallel_workers_per_gather=4
      -c max_parallel_workers=12
      -c max_parallel_maintenance_workers=4
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d irandb"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - routing_net
    <<: *default-logging

  redis:
    image: mirror2.chabokan.net/library/redis:7-alpine
    container_name: redis
    restart: always
    sysctls:
      net.core.somaxconn: 511
    command: >
      redis-server
      --appendonly yes
      --maxmemory 2gb
      --maxmemory-policy allkeys-lru
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - routing_net
    <<: *default-logging

  osrm-ch:
    image: mirror2.chabokan.net/osrm/osrm-backend:latest
    container_name: osrm-ch
    restart: always
    ports:
      - "5000:5000"
    volumes:
      - /opt/osrm-data:/data
    command: osrm-routed --algorithm ch /data/iran-260404.osrm
    networks:
      - routing_net
    <<: *default-logging

  vroom:
    image: vroomvrp/vroom-docker:v1.13.0
    container_name: vroom
    restart: always
    ports:
      - "3000:3000"
    environment:
      ROUTER: osrm
      OSRM_HOST: osrm-ch
      OSRM_PORT: 5000
    depends_on:
      - osrm-ch
    networks:
      - routing_net
    <<: *default-logging

  backend:
    image: mirror2.chabokan.net/library/python:3.12-slim
    container_name: backend
    restart: always
    working_dir: /app
    volumes:
      - ./backend:/app
    command: >
      bash -c "
      pip install --no-index --find-links=/app/packages fastapi uvicorn asyncpg sqlalchemy redis requests prometheus-fastapi-instrumentator prometheus_client &&
      uvicorn main:app --host 0.0.0.0 --port 8000
      "
    ports:
      - "8000:8000"
    depends_on:
      postgis:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - routing_net
    <<: *default-logging

  prometheus:
    image: mirror2.chabokan.net/prom/prometheus:latest
    container_name: prometheus
    restart: always
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    networks:
      - routing_net
    <<: *default-logging

  grafana:
    image: mirror2.chabokan.net/grafana/grafana:latest
    container_name: grafana
    restart: always
    ports:
      - "3001:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - routing_net
    <<: *default-logging

  nginx:
    image: mirror2.chabokan.net/library/nginx:stable-alpine
    container_name: nginx
    restart: always
    ports:
      - "80:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - backend
    networks:
      - routing_net
    <<: *default-logging

```

### 2. Reverse Proxy Rules (`nginx/default.conf`)

```nginx
server {
    listen 80;
    server_name localhost;

    access_log /var/log/nginx/logistics_access.log;
    error_log /var/log/nginx/logistics_error.log;

    location / {
        proxy_pass http://backend:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 90s;
        proxy_connect_timeout 90s;
    }

    location /vroom/ {
        proxy_pass http://vroom:3000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 120s;
    }

    location /osrm/ {
        proxy_pass http://osrm-ch:5000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 90s;
    }
}

```

### 3. Monitoring Metrics Scraper (`prometheus/prometheus.yml`)

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'fastapi-backend'
    scrape_interval: 5s
    static_configs:
      - targets: ['backend:8000']

```

---

## ⚡ Orchestration & Control Flow

```bash
cd /opt/logistics-stack

# Boot up the cluster detached with explicit production configuration
docker compose -f docker-compose.production.yml up -d

# Check cluster operational health and status
docker compose -f docker-compose.production.yml ps

# Follow real-time application runtime logs
docker compose -f docker-compose.production.yml logs -f backend

# Shut down cluster preserving volumes mappings
docker compose -f docker-compose.production.yml down

```

---

## 🛠️ Infrastructure Troubleshooting Guide (Runbook)

### 🚨 Issue 1: OSRM container crashes with `File token match failed` or `File size mismatch`

* **Root Cause:** The `.osrm` binary files were built with a different version of the `osrm-backend` image than the version specified in the `docker-compose` file, or the files got corrupted during extraction.
* **Solution:** Ensure that both the execution script and your docker compose map to the exact same image tag (`latest` or specific tags like `v5.27.1`). Wipe the existing compiled graph data and rerun the build pipeline script:
```bash
rm -rf /opt/osrm-data/iran-260404.osrm*
python3 /opt/logistics-stack/process_map.py
docker compose -f docker-compose.production.yml restart osrm-ch

```



### 🚨 Issue 2: `postgis` container crashes unexpectedly or reports `killed` in logs

* **Root Cause:** Kernel Out-Of-Memory (OOM) killer terminating PostgreSQL due to excessive allocation on shared system memory (`shm_size`) without adequate physical RAM availability.
* **Solution:** Verify host system RAM status (`free -m`). If memory limits are constrained, decrease the allocated shared buffers flag down inside `docker-compose.production.yml`:
```yaml
# Reduce memory consumption footprint inside postgis services block:
shm_size: 4gb
command: >
  postgres
  -c shared_buffers=4GB
  -c effective_cache_size=12GB

```



### 🚨 Issue 3: Redis reports `OOM command not allowed when used memory > 'maxmemory'`

* **Root Cause:** Eviction constraint blocks because Redis cache memory reaches its strict 2GB limit without executing LRU logic over non-volatile keys.
* **Solution:** Confirm your configuration uses volatile eviction policies. Ensure the following flag execution matches inside your active runtime:
```bash
docker exec -it redis redis-cli CONFIG GET maxmemory-policy
# If it is not allkeys-lru, set it manually or restart with correct flags:
docker exec -it redis redis-cli CONFIG SET maxmemory-policy allkeys-lru

```



### 🚨 Issue 4: FastAPI Backend cannot install packages / network resolution failure

* **Root Cause:** Standard container names fallback or DNS lookup drops while operating within an offline/internal isolated cluster node network.
* **Solution:** The stack uses `--no-index --find-links=/app/packages`. Ensure you have pre-downloaded the required `.whl` files inside `/opt/logistics-stack/backend/packages/` before triggering the build step.

---

## 📱 Mobile Architecture Core (Android Client Application)

The client app is built in Native **Kotlin**, designed explicitly to interact with this orchestration layer to render vectors, stream telemetry updates, and handle voice-guided rerouting hooks.

### 📦 Dependency Architecture

#### `settings.gradle.kts`

```kotlin
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
        maven { url = java.net.URI("[https://maven.pkg.github.com/maplibre/maplibre-native-android](https://maven.pkg.github.com/maplibre/maplibre-native-android)") }
    }
}

```

#### `build.gradle.kts` (Module: `:app`)

```kotlin
dependencies {
    implementation("androidx.core:core-ktx:1.12.0")
    implementation("androidx.appcompat:appcompat:1.6.1")
    implementation("com.google.android.material:material:1.11.0")
    implementation("androidx.cardview:cardview:1.0.0")

    // Google Play Fused Location Framework
    implementation("com.google.android.gms:play-services-location:21.2.0")

    // MapLibre Vector/Raster Client Maps
    implementation("org.maplibre.gl:android-sdk:11.0.0")

    // Async Networking Components
    implementation("com.squareup.okhttp3:okhttp:4.12.0")
    implementation("org.json:json:20240303")
}

```

#### `AndroidManifest.xml`

```xml
<manifest xmlns:android="[http://schemas.android.com/apk/res/android](http://schemas.android.com/apk/res/android)" package="com.digiexpress.sd">
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-permission android:name="android.permission.INTERNET" />

    <application
        android:allowBackup="true"
        android:supportsRtl="true"
        android:theme="@style/Theme.DigiExpressSD"
        android:usesCleartextTraffic="true"> <activity android:name=".MainActivity" android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>

```

### 💻 Production Main Runtime Controller (`MainActivity.kt`)

```kotlin
package com.digiexpress.sd

import android.Manifest
import android.annotation.SuppressLint
import android.content.pm.PackageManager
import android.graphics.BitmapFactory
import android.graphics.Color
import android.graphics.Typeface
import android.os.Bundle
import android.os.Handler
import android.os.Looper
import android.speech.tts.TextToSpeech
import android.view.View
import android.view.ViewGroup
import android.view.Gravity
import android.widget.Button
import android.widget.FrameLayout
import android.widget.ImageView
import android.widget.LinearLayout
import android.widget.RadioButton
import android.widget.RadioGroup
import android.widget.ScrollView
import android.widget.TextView
import android.widget.Toast
import androidx.appcompat.app.AppCompatActivity
import androidx.cardview.widget.CardView
import androidx.core.app.ActivityCompat
import androidx.core.content.ContextCompat
import androidx.core.graphics.toColorInt
import androidx.core.view.ViewCompat
import androidx.core.view.WindowInsetsCompat
import com.google.android.gms.location.FusedLocationProviderClient
import com.google.android.gms.location.LocationResult
import com.google.android.gms.location.LocationCallback
import com.google.android.gms.location.LocationRequest
import com.google.android.gms.location.LocationServices
import com.google.android.gms.location.Priority
import okhttp3.Call
import okhttp3.Callback
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.OkHttpClient
import okhttp3.Request
import okhttp3.RequestBody.Companion.toRequestBody
import okhttp3.Response
import org.json.JSONObject
import org.maplibre.android.MapLibre
import org.maplibre.android.camera.CameraPosition
import org.maplibre.android.camera.CameraUpdateFactory
import org.maplibre.android.geometry.LatLng
import org.maplibre.android.maps.MapLibreMap
import org.maplibre.android.maps.MapView
import org.maplibre.android.maps.Style
import org.maplibre.android.style.layers.LineLayer
import org.maplibre.android.style.layers.Property
import org.maplibre.android.style.layers.PropertyFactory
import org.maplibre.android.style.layers.RasterLayer
import org.maplibre.android.style.layers.SymbolLayer
import org.maplibre.android.style.sources.GeoJsonSource
import org.maplibre.android.style.sources.RasterSource
import org.maplibre.android.style.sources.TileSet
import org.maplibre.geojson.LineString
import org.maplibre.geojson.Point
import java.io.IOException
import java.util.Locale

class MainActivity : AppCompatActivity(), TextToSpeech.OnInitListener {

    private lateinit var mapView: MapView
    private lateinit var map: MapLibreMap
    private lateinit var fusedLocationClient: FusedLocationProviderClient
    private var ttsEngine: TextToSpeech? = null

    private var userLoc: LatLng? = null
    private var destLoc: LatLng? = null
    private var firstFixDone = false
    private var isNavigating = false
    private var isCameraLocked = true
    private var lastSpokenInstruction = ""

    private var lastRerouteTime: Long = 0
    private var currentRoutePoints: List<LatLng> = emptyList()
    private var initialEstimatedMinutes: Int = 0
    private var totalRouteDistanceMeters: Double = 0.0
    private var targetAddressText: String = "Extracting destination mapping context..."

    private lateinit var userSource: GeoJsonSource
    private lateinit var destSource: GeoJsonSource
    private lateinit var routeSourceMain: GeoJsonSource
    private lateinit var routeSourceAlt1: GeoJsonSource
    private lateinit var routeSourceAlt2: GeoJsonSource

    private lateinit var cardNavigationPanel: CardView
    private lateinit var txtInstruction: TextView
    private lateinit var txtEtaTime: TextView
    private lateinit var cardBottomPanel: CardView
    private lateinit var layoutRouteSelection: LinearLayout
    private lateinit var txtRoutePath: TextView
    private lateinit var btnStartMovement: Button
    private lateinit var btnCancelNavigation: Button
    private lateinit var btnLiveExitNavigation: CardView
    private lateinit var statusBarSpacer: View

    private val backendUrl = "[http://172.16.13.3](http://172.16.13.3)"
    private val locationRequestCode = 1001
    private val client = OkHttpClient()
    private var locationCallback: LocationCallback? = null

    @SuppressLint("SetTextI18n")
    override fun onCreate(savedInstanceState: Bundle?) {
        setTheme(R.style.Theme_DigiExpressSD)
        super.onCreate(savedInstanceState)

        MapLibre.getInstance(this)
        fusedLocationClient = LocationServices.getFusedLocationProviderClient(this)
        ttsEngine = TextToSpeech(this, this, "com.google.android.tts")
        supportActionBar?.hide()

        val mainContainer = FrameLayout(this).apply {
            layoutParams = ViewGroup.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT)
            setBackgroundColor(Color.WHITE)
        }

        val rootLayout = LinearLayout(this).apply {
            orientation = LinearLayout.VERTICAL
            layoutParams = FrameLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT)
        }

        statusBarSpacer = View(this).apply {
            layoutParams = LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, 0)
            setBackgroundColor(Color.WHITE)
        }
        rootLayout.addView(statusBarSpacer)

        ViewCompat.setOnApplyWindowInsetsListener(mainContainer) { _, insets ->
            val statusBarHeight = insets.getInsets(WindowInsetsCompat.Type.statusBars()).top
            statusBarSpacer.layoutParams.height = statusBarHeight
            statusBarSpacer.requestLayout()
            insets
        }

        setupCustomTopHeader(rootLayout)

        val mapContainer = FrameLayout(this).apply {
            layoutParams = LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, 0, 1f)
        }

        mapView = MapView(this).apply {
            layoutParams = FrameLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT)
        }
        mapContainer.addView(mapView)

        setupTopNavigationCard(mapContainer)
        setupLiveExitButton(mapContainer)
        setupBottomControlPanel(mapContainer)
        setupFloatingGpsButton(mapContainer)

        rootLayout.addView(mapContainer)
        mainContainer.addView(rootLayout)

        val splashLayout = LinearLayout(this).apply {
            orientation = LinearLayout.VERTICAL
            gravity = Gravity.CENTER
            setBackgroundColor(Color.WHITE)
            layoutParams = FrameLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT)
        }

        val imgSplashLogo = ImageView(this).apply {
            setImageResource(R.drawable.splash_logo)
            scaleType = ImageView.ScaleType.FIT_CENTER
            layoutParams = LinearLayout.LayoutParams(1400, 770)
        }
        splashLayout.addView(imgSplashLogo)
        mainContainer.addView(splashLayout)

        setContentView(mainContainer)

        Handler(Looper.getMainLooper()).postDelayed({
            splashLayout.animate().alpha(0f).setDuration(350).withEndAction {
                mainContainer.removeView(splashLayout)
            }.start()
        }, 2000)

        mapView.onCreate(savedInstanceState)
        mapView.getMapAsync { m ->
            map = m
            map.setStyle(Style.Builder()) { style ->
                val tiles = RasterSource(
                    "tiles",
                    TileSet("tileset", "[https://osm.digiexpress.ir/tile/](https://osm.digiexpress.ir/tile/){z}/{x}/{y}.png"),
                    256
                )
                style.addSource(tiles)
                style.addLayer(RasterLayer("tile", "tiles"))

                style.addImage("driver_car", BitmapFactory.decodeResource(resources, android.R.drawable.ic_menu_mylocation))
                style.addImage("delivery_target", BitmapFactory.decodeResource(resources, android.R.drawable.ic_menu_compass))

                userSource = GeoJsonSource("user_src")
                destSource = GeoJsonSource("dest_src")
                routeSourceMain = GeoJsonSource("route_main_src")
                routeSourceAlt1 = GeoJsonSource("route_alt1_src")
                routeSourceAlt2 = GeoJsonSource("route_alt2_src")

                style.addSource(userSource)
                style.addSource(destSource)
                style.addSource(routeSourceMain)
                style.addSource(routeSourceAlt1)
                style.addSource(routeSourceAlt2)

                style.addLayer(SymbolLayer("user_layer", "user_src").withProperties(
                    PropertyFactory.iconImage("driver_car"),
                    PropertyFactory.iconAllowOverlap(true)
                ))
                style.addLayer(SymbolLayer("dest_layer", "dest_src").withProperties(
                    PropertyFactory.iconImage("delivery_target"),
                    PropertyFactory.iconAllowOverlap(true)
                ))

                style.addLayer(LineLayer("layer_main_route", "route_main_src").withProperties(
                    PropertyFactory.lineColor("#00E676".toColorInt()),
                    PropertyFactory.lineWidth(8f),
                    PropertyFactory.lineCap(Property.LINE_CAP_ROUND),
                    PropertyFactory.lineJoin(Property.LINE_JOIN_ROUND)
                ))

                style.addLayer(LineLayer("layer_alt1_route", "route_alt1_src").withProperties(
                    PropertyFactory.lineColor("#FF9100".toColorInt()),
                    PropertyFactory.lineWidth(6f),
                    PropertyFactory.lineDasharray(arrayOf(2f, 2f))
                ))

                style.addLayer(LineLayer("layer_alt2_route", "route_alt2_src").withProperties(
                    PropertyFactory.lineColor("#90A4AE".toColorInt()),
                    PropertyFactory.lineWidth(6f)
                ))

                enableMapClick()

                map.addOnCameraMoveStartedListener { reason ->
                    if (reason == MapLibreMap.OnCameraMoveStartedListener.REASON_API_GESTURE) {
                        if (isNavigating && isCameraLocked) {
                            isCameraLocked = false
                            Toast.makeText(this@MainActivity, "Map lock detached. Tap GPS button to center view.", Toast.LENGTH_SHORT).show()
                        }
                    }
                }
                checkPermission()
            }
        }
    }

    override fun onInit(status: Int) {
        if (status == TextToSpeech.SUCCESS) {
            val farsiLocale = Locale.forLanguageTag("fa-IR")
            val result = ttsEngine?.setLanguage(farsiLocale)
            if (result == TextToSpeech.LANG_MISSING_DATA || result == TextToSpeech.LANG_NOT_SUPPORTED) {
                ttsEngine?.language = Locale.Builder().setLanguage("fa").build()
            }
        }
    }

    private fun speakInstruction(text: String) {
        if (text.isNotBlank() && text != lastSpokenInstruction) {
            lastSpokenInstruction = text
            ttsEngine?.speak(text, TextToSpeech.QUEUE_FLUSH, null, "NavID")
        }
    }

    @SuppressLint("SetTextI18n")
    private fun setupCustomTopHeader(root: LinearLayout) {
        val headerCard = CardView(this).apply {
            radius = 0f
            cardElevation = 4f
            setCardBackgroundColor(Color.WHITE)
            layoutParams = LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT)
        }

        val headerLayout = LinearLayout(this).apply {
            orientation = LinearLayout.HORIZONTAL
            gravity = Gravity.CENTER_VERTICAL or Gravity.END
            setPadding(40, 24, 40, 24)
            layoutParams = FrameLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT)
        }

        val txtTitle = TextView(this).apply {
            text = "DigiExpress | Cluster Infrastructure"
            setTextColor("#1F2937".toColorInt())
            textSize = 16f
            setTypeface(null, Typeface.BOLD)
            gravity = Gravity.END
        }

        val imgLogo = ImageView(this).apply {
            setImageResource(R.drawable.splash_logo)
            scaleType = ImageView.ScaleType.FIT_CENTER
        }
        val imgParams = LinearLayout.LayoutParams(140, 42).apply { setMargins(20, 0, 0, 0) }
        imgLogo.layoutParams = imgParams

        headerLayout.addView(txtTitle)
        headerLayout.addView(imgLogo)
        headerCard.addView(headerLayout)
        root.addView(headerCard)
    }

    @SuppressLint("SetTextI18n")
    private fun setupTopNavigationCard(root: FrameLayout) {
        cardNavigationPanel = CardView(this).apply {
            radius = 20f
            cardElevation = 14f
            setCardBackgroundColor("#1E1E24".toColorInt())
            visibility = View.GONE
            setPadding(25, 25, 25, 25)
        }

        val innerLayout = LinearLayout(this).apply {
            orientation = LinearLayout.VERTICAL
            layoutParams = LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT)
        }

        txtInstruction = TextView(this).apply {
            text = "Calculating optimal paths..."
            setTextColor(Color.WHITE)
            textSize = 15f
            gravity = Gravity.END
            setPadding(10, 8, 10, 4)
        }

        txtEtaTime = TextView(this).apply {
            text = "Remaining Time: --"
            setTextColor("#00E676".toColorInt())
            textSize = 13f
            gravity = Gravity.END
            setPadding(10, 0, 10, 8)
        }

        innerLayout.addView(txtInstruction)
        innerLayout.addView(txtEtaTime)
        cardNavigationPanel.addView(innerLayout)

        val params = FrameLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT).apply {
            gravity = Gravity.TOP
            setMargins(30, 30, 30, 0)
        }
        root.addView(cardNavigationPanel, params)
    }

    private fun setupLiveExitButton(root: FrameLayout) {
        btnLiveExitNavigation = CardView(this).apply {
            radius = 35f
            cardElevation = 12f
            setCardBackgroundColor("#E02424".toColorInt())
            visibility = View.GONE
        }

        val imgClose = ImageView(this).apply {
            setImageResource(android.R.drawable.ic_menu_close_clear_cancel)
            setColorFilter(Color.WHITE)
            setPadding(16, 16, 16, 16)
        }
        btnLiveExitNavigation.addView(imgClose)

        btnLiveExitNavigation.setOnClickListener {
            resetNavigationState()
            Toast.makeText(this, "Live navigation cancelled.", Toast.LENGTH_SHORT).show()
        }

        val params = FrameLayout.LayoutParams(80, 80).apply {
            gravity = Gravity.TOP or Gravity.END
            setMargins(0, 210, 40, 0)
        }
        root.addView(btnLiveExitNavigation, params)
    }

    @SuppressLint("SetTextI18n")
    private fun setupBottomControlPanel(root: FrameLayout) {
        cardBottomPanel = CardView(this).apply {
            radius = 28f
            cardElevation = 20f
            setCardBackgroundColor(Color.WHITE)
            visibility = View.GONE
        }

        val panel = LinearLayout(this).apply {
            orientation = LinearLayout.VERTICAL
            setPadding(40, 40, 40, 40)
            layoutParams = LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT)
        }

        layoutRouteSelection = LinearLayout(this).apply {
            orientation = LinearLayout.VERTICAL
            visibility = View.GONE
            layoutParams = LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT)
        }

        txtRoutePath = TextView(this).apply {
            text = "Extracting spatial context..."
            setTextColor("#374151".toColorInt())
            textSize = 14f
            setPadding(10, 5, 10, 15)
            gravity = Gravity.END
        }
        layoutRouteSelection.addView(txtRoutePath)

        btnStartMovement = Button(this).apply {
            text = "Start Live Navigation"
            setBackgroundColor("#E02424".toColorInt())
            setTextColor(Color.WHITE)
            visibility = View.GONE
            layoutParams = LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT).apply {
                setMargins(0, 20, 0, 0)
            }
            setOnClickListener {
                isNavigating = true
                isCameraLocked = true
                cardNavigationPanel.visibility = View.VISIBLE
                btnLiveExitNavigation.visibility = View.VISIBLE
                cardBottomPanel.visibility = View.GONE

                userLoc?.let { loc -> moveCameraToPosition(loc, 18.0, 55.0, map.cameraPosition.bearing) }
                speakInstruction(txtInstruction.text.toString())
                Toast.makeText(this@MainActivity, "Voice tracking initialized.", Toast.LENGTH_SHORT).show()
            }
        }

        btnCancelNavigation = Button(this).apply {
            text = "Reset Destination Node"
            setBackgroundColor("#78909C".toColorInt())
            setTextColor(Color.WHITE)
            visibility = View.GONE
            layoutParams = LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT).apply {
                setMargins(0, 12, 0, 0)
            }
            setOnClickListener {
                resetNavigationState()
                Toast.makeText(this@MainActivity, "Path cleared. Choose alternative.", Toast.LENGTH_SHORT).show()
            }
        }

        panel.addView(layoutRouteSelection)
        panel.addView(btnStartMovement)
        panel.addView(btnCancelNavigation)
        cardBottomPanel.addView(panel)

        val params = FrameLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT).apply {
            gravity = Gravity.BOTTOM
            setMargins(30, 0, 30, 40)
        }
        root.addView(cardBottomPanel, params)
    }

    private fun resetNavigationState() {
        isNavigating = false
        isCameraLocked = true
        destLoc = null
        lastSpokenInstruction = ""
        currentRoutePoints = emptyList()
        initialEstimatedMinutes = 0
        totalRouteDistanceMeters = 0.0
        targetAddressText = "Extracting address data..."
        ttsEngine?.stop()

        cardNavigationPanel.visibility = View.GONE
        btnLiveExitNavigation.visibility = View.GONE
        cardBottomPanel.visibility = View.GONE
        layoutRouteSelection.visibility = View.GONE
        btnStartMovement.visibility = View.GONE

        routeSourceMain.setGeoJson(LineString.fromLngLats(emptyList()))
        routeSourceAlt1.setGeoJson(LineString.fromLngLats(emptyList()))
        routeSourceAlt2.setGeoJson(LineString.fromLngLats(emptyList()))
        destSource.setGeoJson(Point.fromLngLat(0.0, 0.0))
    }

    private fun setupFloatingGpsButton(root: FrameLayout) {
        val gpsCard = CardView(this).apply {
            radius = 45f
            cardElevation = 10f
            setCardBackgroundColor(Color.WHITE)
        }

        val imgGps = ImageView(this)
        imgGps.setImageResource(android.R.drawable.ic_menu_mylocation)
        imgGps.setColorFilter("#1E88E5".toColorInt())
        imgGps.setPadding(18, 18, 18, 18)
        gpsCard.addView(imgGps)

        gpsCard.setOnClickListener {
            userLoc?.let { loc ->
                if (isNavigating) {
                    isCameraLocked = true
                    moveCameraToPosition(loc, 18.0, 55.0, map.cameraPosition.bearing)
                } else {
                    moveCameraToPosition(loc, 17.5, 45.0, 0.0)
                }
            }
        }

        val params = FrameLayout.LayoutParams(90, 90).apply {
            gravity = Gravity.BOTTOM or Gravity.END
            setMargins(0, 0, 35, 300)
        }
        root.addView(gpsCard, params)
    }

    private fun enableMapClick() {
        map.addOnMapClickListener { point ->
            if (isNavigating) return@addOnMapClickListener false
            destLoc = point
            destSource.setGeoJson(Point.fromLngLat(point.longitude, point.latitude))

            fetchTargetAddressGeocode(point.latitude, point.longitude)
            requestMultiRoutingEngine()
            true
        }
    }

    private fun fetchTargetAddressGeocode(lat: Double, lon: Double) {
        targetAddressText = "Fetching geocode metadata..."
        val url = String.format(Locale.US, "%s/api/v1/navigation/reverse?lat=%.6f&lon=%.6f", backendUrl, lat, lon)
        val request = Request.Builder().url(url).get().build()

        client.newCall(request).enqueue(object : Callback {
            override fun onFailure(call: Call, e: IOException) {
                runOnUiThread {
                    targetAddressText = "Network Timeout (Offline Gateway Node)"
                    txtRoutePath.text = String.format(Locale.US, "Destination:\n%s", targetAddressText)
                }
            }
            override fun onResponse(call: Call, response: Response) {
                val bodyStr = response.body?.string() ?: return
                try {
                    val json = JSONObject(bodyStr)
                    val fullAddress = json.optString("compact_address", "Unknown Zone")
                    runOnUiThread {
                        targetAddressText = fullAddress
                        txtRoutePath.text = String.format(Locale.US, "📍 Delivery Destination:\n%s", targetAddressText)
                    }
                } catch (_: Exception) {
                    runOnUiThread {
                        targetAddressText = "Unresolved Geozone Mapping"
                        txtRoutePath.text = String.format(Locale.US, "Destination:\n%s", targetAddressText)
                    }
                }
            }
        })
    }

    @SuppressLint("SetTextI18n")
    private fun requestMultiRoutingEngine() {
        val start = userLoc ?: return
        val end = destLoc ?: return

        val url = String.format(
            Locale.US,
            "%s/api/v1/navigation/routes?origin_lat=%.6f&origin_lon=%.6f&dest_lat=%.6f&dest_lon=%.6f",
            backendUrl, start.latitude, start.longitude, end.latitude, end.longitude
        )

        val request = Request.Builder().url(url).get().build()

        client.newCall(request).enqueue(object : Callback {
            override fun onFailure(call: Call, e: IOException) {
                runOnUiThread { Toast.makeText(this@MainActivity, "Internal Logistics Core Endpoint Disconnected", Toast.LENGTH_SHORT).show() }
            }

            override fun onResponse(call: Call, response: Response) {
                val bodyStr = response.body?.string() ?: return
                try {
                    val json = JSONObject(bodyStr)
                    val routesArray = json.getJSONArray("routes")

                    runOnUiThread {
                        layoutRouteSelection.removeAllViews()
                        layoutRouteSelection.addView(txtRoutePath)
                        txtRoutePath.text = String.format(Locale.US, "📍 Delivery Destination:\n%s", targetAddressText)

                        cardBottomPanel.visibility = View.VISIBLE
                        layoutRouteSelection.visibility = View.VISIBLE
                        btnStartMovement.visibility = View.VISIBLE
                        btnCancelNavigation.visibility = View.VISIBLE

                        val radioGroup = RadioGroup(this@MainActivity).apply {
                            orientation = RadioGroup.VERTICAL
                            layoutParams = LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT)
                        }

                        routeSourceMain.setGeoJson(LineString.fromLngLats(emptyList()))
                        routeSourceAlt1.setGeoJson(LineString.fromLngLats(emptyList()))
                        routeSourceAlt2.setGeoJson(LineString.fromLngLats(emptyList()))

                        val pathDetailsContainer = LinearLayout(this@MainActivity).apply {
                            orientation = LinearLayout.VERTICAL
                            layoutParams = LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT)
                            setPadding(15, 10, 15, 15)
                        }

                        for (i in 0 until routesArray.length()) {
                            val route = routesArray.getJSONObject(i)
                            val polylineEncoded = route.getString("geometry_polyline")
                            val etaText = route.getString("eta_formatted")
                            val durationSeconds = route.optInt("duration_seconds", 1200)
                            val distanceMeters = route.optDouble("total_distance_meters", 1000.0)
                            val tag = route.getString("tag")

                            val decodedPoints = decodePolyline(polylineEncoded).map { Point.fromLngLat(it.first, it.second) }
                            val lineString = LineString.fromLngLats(decodedPoints)

                            val stepsArray = route.optJSONArray("steps")
                            val logBuilder = java.lang.StringBuilder()

                            if (stepsArray != null && stepsArray.length() > 0) {
                                for (j in 0 until stepsArray.length()) {
                                    val stepObj = stepsArray.getJSONObject(j)
                                    val stepInstruction = stepObj.optString("instruction", "")
                                    if (stepInstruction.isNotBlank()) {
                                        logBuilder.append("• ").append(stepInstruction).append("\n")
                                    }
                                }
                            }

                            val fullRouteLog = logBuilder.toString()

                            when (i) {
                                0 -> {
                                    routeSourceMain.setGeoJson(lineString)
                                    currentRoutePoints = decodedPoints.map { LatLng(it.latitude(), it.longitude()) }

                                    totalRouteDistanceMeters = distanceMeters
                                    initialEstimatedMinutes = durationSeconds / 60

                                    if (stepsArray != null && stepsArray.length() > 0) {
                                        txtInstruction.text = stepsArray.getJSONObject(0).optString("instruction", "Follow highlighted route.")
                                        txtEtaTime.text = String.format(Locale.US, "ETA: %d mins", initialEstimatedMinutes)
                                    }

                                    val txtMainPathTitle = TextView(this@MainActivity).apply {
                                        text = "Route Waypoint Directives:"
                                        setTextColor("#1F2937".toColorInt())
                                        textSize = 13f
                                        setTypeface(null, Typeface.BOLD)
                                        gravity = Gravity.END
                                        setPadding(0, 10, 10, 5)
                                    }

                                    val txtMainPathDetails = TextView(this@MainActivity).apply {
                                        text = fullRouteLog.ifBlank { "Direct transit pathway." }
                                        setTextColor("#4B5563".toColorInt())
                                        textSize = 12f
                                        gravity = Gravity.END
                                        setLineSpacing(0f, 1.2f)
                                    }
                                    pathDetailsContainer.addView(txtMainPathTitle)
                                    pathDetailsContainer.addView(txtMainPathDetails)
                                }
                                1 -> routeSourceAlt1.setGeoJson(lineString)
                                2 -> routeSourceAlt2.setGeoJson(lineString)
                            }

                            val routeCard = RadioButton(this@MainActivity).apply {
                                text = String.format(Locale.US, "%s (%s) - %.1f KM", tag, etaText, distanceMeters / 1000.0)
                                id = View.generateViewId()
                                isChecked = (i == 0)
                                setTextColor("#4B5563".toColorInt())
                                setPadding(15, 15, 15, 15)
                                textSize = 14f
                            }
                            radioGroup.addView(routeCard)
                        }

                        layoutRouteSelection.addView(radioGroup)

                        val scrollView = ScrollView(this@MainActivity).apply {
                            layoutParams = LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, 220)
                            addView(pathDetailsContainer)
                        }
                        layoutRouteSelection.addView(scrollView)
                    }
                } catch (err: Exception) {
                    err.printStackTrace()
                }
            }
        })
    }

    private fun moveCameraToPosition(target: LatLng, zoom: Double, tilt: Double, bearing: Double) {
        map.animateCamera(
            CameraUpdateFactory.newCameraPosition(
                CameraPosition.Builder()
                    .target(target)
                    .zoom(zoom)
                    .tilt(tilt)
                    .bearing(bearing)
                    .build()
            ), 750
        )
    }

    private fun decodePolyline(encoded: String): List<Pair<Double, Double>> {
        val list = mutableListOf<Pair<Double, Double>>()
        var i = 0; var lat = 0; var lng = 0
        while (i < encoded.length) {
            var shift = 0; var result = 0; var b: Int
            do { b = encoded[i++].code - 63; result = result or (b and 0x1f shl shift); shift += 5 } while (b >= 0x20)
            lat += if (result and 1 != 0) (result shr 1).inv() else result shr 1
            shift = 0; result = 0
            do { b = encoded[i++].code - 63; result = result or (b and 0x1f shl shift); shift += 5 } while (b >= 0x20)
            lng += if (result and 1 != 0) (result shr 1).inv() else result shr 1
            list.add(Pair(lng / 1e5, lat / 1e5))
        }
        return list
    }

    private fun streamDriverLocationToCore(lat: Double, lon: Double) {
        val url = String.format(Locale.US, "%s/api/v1/addresses/enrich", backendUrl)
        val json = String.format(
            Locale.US,
            "{\"address_text\": \"Active Fleet Sync Beacon\", \"latitude\": %.6f, \"longitude\": %.6f}",
            lat, lon
        )

        val body = json.toRequestBody("application/json".toMediaType())
        val request = Request.Builder().url(url).post(body).build()
        client.newCall(request).enqueue(object : Callback {
            override fun onFailure(call: Call, e: IOException) {}
            override fun onResponse(call: Call, response: Response) { response.close() }
        })
    }

    private fun distanceFromRoute(userLoc: LatLng, route: List<LatLng>): Double {
        var minDistance = Double.MAX_VALUE
        for (point in route) {
            val results = FloatArray(1)
            android.location.Location.distanceBetween(
                userLoc.latitude, userLoc.longitude,
                point.latitude, point.longitude,
                results
            )
            if (results[0] < minDistance) minDistance = results[0].toDouble()
        }
        return minDistance
    }

    @SuppressLint("MissingPermission", "SetTextI18n")
    private fun startGPS() {
        val req = LocationRequest.Builder(Priority.PRIORITY_HIGH_ACCURACY, 1000).apply {
            setMinUpdateIntervalMillis(1000)
            setMaxUpdateDelayMillis(1000)
        }.build()

        locationCallback = object : LocationCallback() {
            override fun onLocationResult(result: LocationResult) {
                val loc = result.locations.maxByOrNull { it.accuracy } ?: result.lastLocation ?: return
                if (firstFixDone && loc.accuracy > 50) return

                val currentLatLng = LatLng(loc.latitude, loc.longitude)
                userLoc = currentLatLng

                userSource.setGeoJson(Point.fromLngLat(loc.longitude, loc.latitude))
                streamDriverLocationToCore(loc.latitude, loc.longitude)

                if (!firstFixDone) {
                    firstFixDone = true
                    moveCameraToPosition(currentLatLng, 17.2, 45.0, 0.0)
                }

                if (isNavigating) {
                    destLoc?.let { dest ->
                        val distanceToDestinationResults = FloatArray(1)
                        android.location.Location.distanceBetween(
                            currentLatLng.latitude, currentLatLng.longitude,
                            dest.latitude, dest.longitude,
                            distanceToDestinationResults
                        )
                        val remainingDistanceMeters = distanceToDestinationResults[0].toDouble()

                        if (remainingDistanceMeters < 50.0) {
                            runOnUiThread {
                                txtInstruction.text = "Arrived at distribution node"
                                txtEtaTime.text = "ETA: 0 Mins"
                                speakInstruction("شما به مقصد رسیدید")
                                Handler(Looper.getMainLooper()).postDelayed({ resetNavigationState() }, 4000)
                            }
                            return
                        }

                        if (totalRouteDistanceMeters > 100.0 && initialEstimatedMinutes > 0) {
                            val progressRatio = remainingDistanceMeters / totalRouteDistanceMeters
                            var currentDynamicMinutes = (initialEstimatedMinutes * progressRatio).toInt()
                            if (currentDynamicMinutes <= 0 && remainingDistanceMeters > 50.0) currentDynamicMinutes = 1

                            runOnUiThread {
                                txtEtaTime.text = String.format(Locale.US, "Remaining: %d Mins (%.1f KM)", currentDynamicMinutes, remainingDistanceMeters / 1000.0)
                            }
                        }
                    }

                    if (currentRoutePoints.isNotEmpty()) {
                        val distanceToRoute = distanceFromRoute(currentLatLng, currentRoutePoints)
                        val currentTime = System.currentTimeMillis()

                        if (distanceToRoute > 40.0 && (currentTime - lastRerouteTime > 5000)) {
                            lastRerouteTime = currentTime
                            runOnUiThread {
                                txtInstruction.text = "Rerouting tracking..."
                                speakInstruction("مسیر جدید محاسبه می‌شود")
                                requestMultiRoutingEngine()
                            }
                        }
                    }

                    if (isCameraLocked) {
                        moveCameraToPosition(currentLatLng, 18.0, 55.0, loc.bearing.toDouble())
                        runOnUiThread { speakInstruction(txtInstruction.text.toString()) }
                    }
                }
            }
        }
        fusedLocationClient.requestLocationUpdates(req, locationCallback!!, mainLooper)
    }

    private fun checkPermission() {
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.ACCESS_FINE_LOCATION) != PackageManager.PERMISSION_GRANTED) {
            ActivityCompat.requestPermissions(this, arrayOf(Manifest.permission.ACCESS_FINE_LOCATION), locationRequestCode)
        } else startGPS()
    }

    override fun onRequestPermissionsResult(requestCode: Int, permissions: Array<out String>, grantResults: IntArray) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults)
        if (requestCode == locationRequestCode && grantResults.isNotEmpty() && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
            startGPS()
        }
    }

    override fun onStart() { super.onStart(); mapView.onStart() }
    override fun onResume() { super.onResume(); mapView.onResume() }
    override fun onPause() { mapView.onPause(); super.onPause() }
    override fun onStop() { mapView.onStop(); super.onStop() }
    override fun onDestroy() {
        ttsEngine?.stop()
        ttsEngine?.shutdown()
        locationCallback?.let { fusedLocationClient.removeLocationUpdates(it) }
        mapView.onDestroy()
        super.onDestroy()
    }
}
----
