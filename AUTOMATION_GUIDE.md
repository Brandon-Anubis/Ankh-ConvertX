# ConvertX Automation Workflow Guide

This guide provides detailed instructions for integrating ConvertX API endpoints into automation workflows, CI/CD pipelines, scripts, and third-party applications.

---

## Table of Contents
1. [Quick Start](#quick-start)
2. [Authentication Strategies](#authentication-strategies)
3. [Common Automation Workflows](#common-automation-workflows)
4. [CI/CD Integration Examples](#cicd-integration-examples)
5. [Scripting Examples](#scripting-examples)
6. [Monitoring & Health Checks](#monitoring--health-checks)
7. [Error Handling & Retry Logic](#error-handling--retry-logic)
8. [Rate Limiting & Best Practices](#rate-limiting--best-practices)

---

## Quick Start

### Prerequisites
- ConvertX instance running and accessible (e.g., `http://localhost:3000` or `https://convert.yourdomain.com`)
- User account created (first user is admin)
- `JWT_SECRET` configured for production deployments

### Basic Workflow
1. Authenticate to get JWT cookie
2. Upload files for conversion
3. Start conversion process
4. Poll for completion
5. Download converted files

---

## Authentication Strategies

### Option 1: Session-Based (Recommended for Scripts)
Maintain a persistent cookie jar across requests.

**Bash/cURL:**
```bash
#!/bin/bash
CONVERTX_URL="http://localhost:3000"
COOKIE_FILE="/tmp/convertx_cookies.txt"

# Authenticate once
curl -X POST "$CONVERTX_URL/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@example.com","password":"your-password"}' \
  -c "$COOKIE_FILE" \
  -s -o /dev/null

# Use cookie for subsequent requests
curl -X GET "$CONVERTX_URL/converters" \
  -b "$COOKIE_FILE"
```

**Python:**
```python
import requests

class ConvertXClient:
    def __init__(self, base_url, email, password):
        self.base_url = base_url
        self.session = requests.Session()
        self.authenticate(email, password)
    
    def authenticate(self, email, password):
        """Authenticate and store JWT cookie in session"""
        response = self.session.post(
            f"{self.base_url}/login",
            json={"email": email, "password": password},
            allow_redirects=False
        )
        if response.status_code not in [302, 200]:
            raise Exception(f"Authentication failed: {response.text}")
    
    def healthcheck(self):
        """Check if service is healthy"""
        response = self.session.get(f"{self.base_url}/healthcheck")
        return response.json()

# Usage
client = ConvertXClient(
    base_url="http://localhost:3000",
    email="admin@example.com",
    password="your-password"
)
print(client.healthcheck())
```

**Node.js:**
```javascript
const axios = require('axios');
const tough = require('tough-cookie');
const { wrapper } = require('axios-cookiejar-support');

class ConvertXClient {
    constructor(baseUrl, email, password) {
        this.baseUrl = baseUrl;
        const cookieJar = new tough.CookieJar();
        this.client = wrapper(axios.create({
            baseURL: baseUrl,
            jar: cookieJar,
            withCredentials: true,
            maxRedirects: 0,
            validateStatus: (status) => status >= 200 && status < 400
        }));
        
        this.authenticate(email, password);
    }
    
    async authenticate(email, password) {
        try {
            await this.client.post('/login', { email, password });
        } catch (error) {
            if (error.response?.status !== 302) {
                throw new Error(`Authentication failed: ${error.message}`);
            }
        }
    }
    
    async healthcheck() {
        const response = await this.client.get('/healthcheck');
        return response.data;
    }
}

// Usage
(async () => {
    const client = new ConvertXClient(
        'http://localhost:3000',
        'admin@example.com',
        'your-password'
    );
    console.log(await client.healthcheck());
})();
```

### Option 2: Per-Request Authentication
Extract JWT token and send as cookie header.

```bash
# Extract JWT token from login response
TOKEN=$(curl -X POST "$CONVERTX_URL/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@example.com","password":"your-password"}' \
  -c - -s | grep 'auth' | awk '{print $7}')

# Use token in subsequent requests
curl -X GET "$CONVERTX_URL/converters" \
  -H "Cookie: auth=$TOKEN"
```

### Option 3: Unauthenticated Mode (Local Development Only)
Set `ALLOW_UNAUTHENTICATED=true` in environment variables. **Not recommended for production.**

```bash
# No authentication required
curl -X POST "$CONVERTX_URL/conversions" \
  -H "Content-Type: application/json" \
  -d '{"fileType":"png"}'
```

---

## Common Automation Workflows

### Workflow 1: Batch File Conversion

**Use Case:** Convert multiple files programmatically.

**Python Example:**
```python
import requests
import time
import json
from pathlib import Path

class ConvertXAutomation:
    def __init__(self, base_url, email, password):
        self.base_url = base_url
        self.session = requests.Session()
        self.authenticate(email, password)
    
    def authenticate(self, email, password):
        response = self.session.post(
            f"{self.base_url}/login",
            json={"email": email, "password": password},
            allow_redirects=False
        )
        if response.status_code not in [302, 200]:
            raise Exception("Authentication failed")
    
    def get_job_id(self):
        """Get jobId cookie from home page visit"""
        response = self.session.get(self.base_url, allow_redirects=True)
        return self.session.cookies.get('jobId')
    
    def upload_files(self, file_paths):
        """Upload multiple files"""
        files = []
        filenames = []
        
        for file_path in file_paths:
            path = Path(file_path)
            filenames.append(path.name)
            files.append(('file', (path.name, open(file_path, 'rb'))))
        
        response = self.session.post(
            f"{self.base_url}/upload",
            files=files
        )
        
        # Close file handles
        for _, (_, file_handle) in files:
            file_handle.close()
        
        if response.status_code != 200:
            raise Exception(f"Upload failed: {response.text}")
        
        return filenames
    
    def start_conversion(self, filenames, target_format, converter):
        """Start conversion process"""
        response = self.session.post(
            f"{self.base_url}/convert",
            data={
                'convert_to': f"{target_format},{converter}",
                'file_names': json.dumps(filenames)
            },
            allow_redirects=False
        )
        
        if response.status_code != 302:
            raise Exception(f"Conversion start failed: {response.text}")
        
        # Extract job ID from redirect location
        location = response.headers.get('Location', '')
        job_id = location.split('/')[-1]
        return job_id
    
    def check_conversion_status(self, job_id):
        """Check conversion status"""
        response = self.session.get(
            f"{self.base_url}/results/{job_id}",
            allow_redirects=True
        )
        
        # Simple check: if page loads successfully, check for completion
        # In production, parse HTML or add a JSON status endpoint
        return 'completed' in response.text.lower()
    
    def download_file(self, user_id, job_id, filename, output_path):
        """Download a converted file"""
        response = self.session.get(
            f"{self.base_url}/download/{user_id}/{job_id}/{filename}",
            stream=True
        )
        
        if response.status_code != 200:
            raise Exception(f"Download failed: {response.status_code}")
        
        with open(output_path, 'wb') as f:
            for chunk in response.iter_content(chunk_size=8192):
                f.write(chunk)
    
    def download_archive(self, job_id, output_path):
        """Download all files as tar archive"""
        response = self.session.get(
            f"{self.base_url}/archive/{job_id}",
            stream=True
        )
        
        if response.status_code != 200:
            raise Exception(f"Archive download failed: {response.status_code}")
        
        with open(output_path, 'wb') as f:
            for chunk in response.iter_content(chunk_size=8192):
                f.write(chunk)
    
    def convert_files(self, file_paths, target_format, converter, 
                     output_dir, max_wait_time=300):
        """Complete conversion workflow"""
        print(f"Starting conversion of {len(file_paths)} files...")
        
        # Step 1: Visit home page to get jobId cookie
        job_id = self.get_job_id()
        print(f"Job ID: {job_id}")
        
        # Step 2: Upload files
        filenames = self.upload_files(file_paths)
        print(f"Uploaded {len(filenames)} files")
        
        # Step 3: Start conversion
        job_id = self.start_conversion(filenames, target_format, converter)
        print(f"Conversion started for job: {job_id}")
        
        # Step 4: Poll for completion
        start_time = time.time()
        while time.time() - start_time < max_wait_time:
            if self.check_conversion_status(job_id):
                print("Conversion completed!")
                break
            print("Waiting for conversion to complete...")
            time.sleep(5)
        else:
            raise Exception("Conversion timeout")
        
        # Step 5: Download archive
        output_file = Path(output_dir) / f"converted_{job_id}.tar"
        self.download_archive(job_id, output_file)
        print(f"Downloaded archive to: {output_file}")
        
        return str(output_file)

# Usage
if __name__ == "__main__":
    automation = ConvertXAutomation(
        base_url="http://localhost:3000",
        email="admin@example.com",
        password="your-password"
    )
    
    files_to_convert = [
        "/path/to/image1.png",
        "/path/to/image2.png",
        "/path/to/document.pdf"
    ]
    
    result = automation.convert_files(
        file_paths=files_to_convert,
        target_format="jpg",
        converter="imagemagick",
        output_dir="/tmp/converted"
    )
    
    print(f"Conversion complete: {result}")
```

### Workflow 2: Watch Folder Automation

**Use Case:** Monitor a folder and automatically convert new files.

**Python Example:**
```python
import time
from pathlib import Path
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

class ConvertXWatcher(FileSystemEventHandler):
    def __init__(self, convertx_client, target_format, converter, output_dir):
        self.client = convertx_client
        self.target_format = target_format
        self.converter = converter
        self.output_dir = output_dir
        self.processed = set()
    
    def on_created(self, event):
        if event.is_directory:
            return
        
        file_path = event.src_path
        
        # Avoid processing the same file twice
        if file_path in self.processed:
            return
        
        self.processed.add(file_path)
        
        # Wait for file to be fully written
        time.sleep(1)
        
        try:
            print(f"New file detected: {file_path}")
            result = self.client.convert_files(
                file_paths=[file_path],
                target_format=self.target_format,
                converter=self.converter,
                output_dir=self.output_dir
            )
            print(f"Conversion complete: {result}")
        except Exception as e:
            print(f"Error converting {file_path}: {e}")
            self.processed.remove(file_path)

# Usage
if __name__ == "__main__":
    client = ConvertXAutomation(
        base_url="http://localhost:3000",
        email="admin@example.com",
        password="your-password"
    )
    
    watch_dir = "/path/to/watch"
    output_dir = "/path/to/output"
    
    event_handler = ConvertXWatcher(
        convertx_client=client,
        target_format="pdf",
        converter="libreoffice",
        output_dir=output_dir
    )
    
    observer = Observer()
    observer.schedule(event_handler, watch_dir, recursive=False)
    observer.start()
    
    print(f"Watching {watch_dir} for new files...")
    
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        observer.stop()
    
    observer.join()
```

### Workflow 3: Web Hook Integration

**Use Case:** Convert files when triggered by webhook from external service.

**Flask Example:**
```python
from flask import Flask, request, jsonify
import tempfile
import requests
from pathlib import Path

app = Flask(__name__)

# Initialize ConvertX client
convertx = ConvertXAutomation(
    base_url="http://localhost:3000",
    email="admin@example.com",
    password="your-password"
)

@app.route('/webhook/convert', methods=['POST'])
def convert_webhook():
    """
    Webhook endpoint that accepts file URL and conversion parameters
    POST /webhook/convert
    {
        "file_url": "https://example.com/file.png",
        "target_format": "jpg",
        "converter": "imagemagick",
        "callback_url": "https://example.com/callback"
    }
    """
    data = request.json
    
    file_url = data.get('file_url')
    target_format = data.get('target_format')
    converter = data.get('converter')
    callback_url = data.get('callback_url')
    
    if not all([file_url, target_format, converter]):
        return jsonify({"error": "Missing required parameters"}), 400
    
    try:
        # Download file from URL
        response = requests.get(file_url, stream=True)
        response.raise_for_status()
        
        # Save to temporary file
        with tempfile.NamedTemporaryFile(delete=False, suffix=Path(file_url).suffix) as tmp_file:
            for chunk in response.iter_content(chunk_size=8192):
                tmp_file.write(chunk)
            tmp_path = tmp_file.name
        
        # Convert file
        result = convertx.convert_files(
            file_paths=[tmp_path],
            target_format=target_format,
            converter=converter,
            output_dir=tempfile.gettempdir()
        )
        
        # Clean up
        Path(tmp_path).unlink()
        
        # Send callback if provided
        if callback_url:
            requests.post(callback_url, json={
                "status": "success",
                "result": result
            })
        
        return jsonify({
            "status": "success",
            "result": result
        }), 200
        
    except Exception as e:
        if callback_url:
            requests.post(callback_url, json={
                "status": "error",
                "message": str(e)
            })
        
        return jsonify({
            "status": "error",
            "message": str(e)
        }), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

## CI/CD Integration Examples

### GitHub Actions

```yaml
# .github/workflows/convert-files.yml
name: Convert Files

on:
  push:
    paths:
      - 'assets/**/*.png'
  workflow_dispatch:

jobs:
  convert:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      
      - name: Install dependencies
        run: |
          pip install requests
      
      - name: Convert images
        env:
          CONVERTX_URL: ${{ secrets.CONVERTX_URL }}
          CONVERTX_EMAIL: ${{ secrets.CONVERTX_EMAIL }}
          CONVERTX_PASSWORD: ${{ secrets.CONVERTX_PASSWORD }}
        run: |
          python scripts/convert_images.py
      
      - name: Commit converted files
        run: |
          git config --local user.email "action@github.com"
          git config --local user.name "GitHub Action"
          git add converted/
          git commit -m "Add converted files" || echo "No changes"
          git push
```

**Script (scripts/convert_images.py):**
```python
import os
import sys
from pathlib import Path

# Import your ConvertXAutomation class
# from convertx_client import ConvertXAutomation

base_url = os.environ['CONVERTX_URL']
email = os.environ['CONVERTX_EMAIL']
password = os.environ['CONVERTX_PASSWORD']

client = ConvertXAutomation(base_url, email, password)

# Find all PNG files
png_files = list(Path('assets').glob('**/*.png'))

if not png_files:
    print("No PNG files found")
    sys.exit(0)

# Convert to JPG
result = client.convert_files(
    file_paths=[str(f) for f in png_files],
    target_format="jpg",
    converter="imagemagick",
    output_dir="converted"
)

print(f"Converted {len(png_files)} files: {result}")
```

### GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - convert

convert_files:
  stage: convert
  image: python:3.10
  script:
    - pip install requests
    - python scripts/convert_documents.py
  artifacts:
    paths:
      - converted/
    expire_in: 1 week
  only:
    - main
```

### Jenkins Pipeline

```groovy
pipeline {
    agent any
    
    environment {
        CONVERTX_URL = credentials('convertx-url')
        CONVERTX_EMAIL = credentials('convertx-email')
        CONVERTX_PASSWORD = credentials('convertx-password')
    }
    
    stages {
        stage('Convert Files') {
            steps {
                script {
                    sh 'pip install requests'
                    sh 'python scripts/convert_files.py'
                }
            }
        }
        
        stage('Archive Results') {
            steps {
                archiveArtifacts artifacts: 'converted/*', fingerprint: true
            }
        }
    }
}
```

### Docker Compose Integration

```yaml
# docker-compose.yml
version: '3.8'

services:
  convertx:
    image: ghcr.io/c4illin/convertx:latest
    container_name: convertx
    ports:
      - "3000:3000"
    environment:
      - JWT_SECRET=${JWT_SECRET}
      - ALLOW_UNAUTHENTICATED=false
    volumes:
      - ./data:/app/data
    restart: unless-stopped
  
  automation:
    build: ./automation
    container_name: convertx-automation
    depends_on:
      - convertx
    environment:
      - CONVERTX_URL=http://convertx:3000
      - CONVERTX_EMAIL=${CONVERTX_EMAIL}
      - CONVERTX_PASSWORD=${CONVERTX_PASSWORD}
    volumes:
      - ./input:/input
      - ./output:/output
    restart: unless-stopped
```

---

## Scripting Examples

### Bash Script: Bulk Conversion

```bash
#!/bin/bash

# Configuration
CONVERTX_URL="http://localhost:3000"
EMAIL="admin@example.com"
PASSWORD="your-password"
COOKIE_FILE="/tmp/convertx_cookies_$$.txt"
INPUT_DIR="./input"
OUTPUT_DIR="./output"

# Colors for output
GREEN='\033[0;32m'
RED='\033[0;31m'
NC='\033[0m' # No Color

# Cleanup function
cleanup() {
    rm -f "$COOKIE_FILE"
}
trap cleanup EXIT

# Authenticate
echo "Authenticating..."
curl -X POST "$CONVERTX_URL/login" \
  -H "Content-Type: application/json" \
  -d "{\"email\":\"$EMAIL\",\"password\":\"$PASSWORD\"}" \
  -c "$COOKIE_FILE" \
  -s -o /dev/null

if [ $? -ne 0 ]; then
    echo -e "${RED}Authentication failed${NC}"
    exit 1
fi

echo -e "${GREEN}Authenticated successfully${NC}"

# Get job ID by visiting home page
curl -X GET "$CONVERTX_URL/" \
  -b "$COOKIE_FILE" \
  -c "$COOKIE_FILE" \
  -s -o /dev/null

# Upload files
echo "Uploading files from $INPUT_DIR..."
for file in "$INPUT_DIR"/*; do
    if [ -f "$file" ]; then
        echo "Uploading $(basename "$file")..."
        curl -X POST "$CONVERTX_URL/upload" \
          -b "$COOKIE_FILE" \
          -F "file=@$file" \
          -s -o /dev/null
    fi
done

echo -e "${GREEN}Files uploaded${NC}"

# Start conversion
echo "Starting conversion..."
FILES=$(ls -1 "$INPUT_DIR" | jq -R -s -c 'split("\n") | map(select(length > 0))')

RESPONSE=$(curl -X POST "$CONVERTX_URL/convert" \
  -b "$COOKIE_FILE" \
  -d "convert_to=pdf,libreoffice" \
  -d "file_names=$FILES" \
  -s -w "\n%{redirect_url}" \
  -o /dev/null)

JOB_ID=$(echo "$RESPONSE" | grep -oP 'results/\K[^/]+')

if [ -z "$JOB_ID" ]; then
    echo -e "${RED}Failed to start conversion${NC}"
    exit 1
fi

echo -e "${GREEN}Conversion started - Job ID: $JOB_ID${NC}"

# Poll for completion
echo "Waiting for conversion to complete..."
MAX_ATTEMPTS=60
ATTEMPT=0

while [ $ATTEMPT -lt $MAX_ATTEMPTS ]; do
    STATUS=$(curl -X GET "$CONVERTX_URL/results/$JOB_ID" \
      -b "$COOKIE_FILE" \
      -s | grep -o "completed" | head -1)
    
    if [ "$STATUS" = "completed" ]; then
        echo -e "${GREEN}Conversion completed!${NC}"
        break
    fi
    
    ATTEMPT=$((ATTEMPT + 1))
    echo "Still processing... ($ATTEMPT/$MAX_ATTEMPTS)"
    sleep 5
done

if [ $ATTEMPT -ge $MAX_ATTEMPTS ]; then
    echo -e "${RED}Conversion timeout${NC}"
    exit 1
fi

# Download archive
echo "Downloading converted files..."
mkdir -p "$OUTPUT_DIR"

curl -X GET "$CONVERTX_URL/archive/$JOB_ID" \
  -b "$COOKIE_FILE" \
  -o "$OUTPUT_DIR/converted_$JOB_ID.tar"

echo -e "${GREEN}Downloaded to: $OUTPUT_DIR/converted_$JOB_ID.tar${NC}"

# Extract archive
cd "$OUTPUT_DIR"
tar -xf "converted_$JOB_ID.tar"
rm "converted_$JOB_ID.tar"

echo -e "${GREEN}Conversion complete!${NC}"
```

### PowerShell Script: Windows Automation

```powershell
# Configuration
$convertxUrl = "http://localhost:3000"
$email = "admin@example.com"
$password = "your-password"
$inputDir = ".\input"
$outputDir = ".\output"

# Create session
$session = New-Object Microsoft.PowerShell.Commands.WebRequestSession

# Authenticate
Write-Host "Authenticating..." -ForegroundColor Yellow
$loginBody = @{
    email = $email
    password = $password
} | ConvertTo-Json

try {
    Invoke-WebRequest -Uri "$convertxUrl/login" `
        -Method Post `
        -Body $loginBody `
        -ContentType "application/json" `
        -WebSession $session `
        -MaximumRedirection 0 `
        -ErrorAction SilentlyContinue | Out-Null
    
    Write-Host "Authenticated successfully" -ForegroundColor Green
} catch {
    Write-Host "Authentication failed" -ForegroundColor Red
    exit 1
}

# Get job ID
Invoke-WebRequest -Uri "$convertxUrl/" `
    -Method Get `
    -WebSession $session | Out-Null

# Upload files
Write-Host "Uploading files..." -ForegroundColor Yellow
$files = Get-ChildItem -Path $inputDir -File

foreach ($file in $files) {
    Write-Host "Uploading $($file.Name)..."
    
    $boundary = [System.Guid]::NewGuid().ToString()
    $fileBytes = [System.IO.File]::ReadAllBytes($file.FullName)
    $fileContent = [System.Text.Encoding]::GetEncoding("iso-8859-1").GetString($fileBytes)
    
    $body = @"
--$boundary
Content-Disposition: form-data; name="file"; filename="$($file.Name)"
Content-Type: application/octet-stream

$fileContent
--$boundary--
"@
    
    Invoke-WebRequest -Uri "$convertxUrl/upload" `
        -Method Post `
        -Body $body `
        -ContentType "multipart/form-data; boundary=$boundary" `
        -WebSession $session | Out-Null
}

Write-Host "Files uploaded" -ForegroundColor Green

# Start conversion
Write-Host "Starting conversion..." -ForegroundColor Yellow
$fileNames = $files | ForEach-Object { $_.Name }
$fileNamesJson = $fileNames | ConvertTo-Json -Compress

$convertBody = @{
    convert_to = "jpg,imagemagick"
    file_names = $fileNamesJson
}

$response = Invoke-WebRequest -Uri "$convertxUrl/convert" `
    -Method Post `
    -Body $convertBody `
    -WebSession $session `
    -MaximumRedirection 0 `
    -ErrorAction SilentlyContinue

$jobId = $response.Headers.Location.Split('/')[-1]
Write-Host "Conversion started - Job ID: $jobId" -ForegroundColor Green

# Poll for completion
Write-Host "Waiting for conversion..." -ForegroundColor Yellow
$maxAttempts = 60
$attempt = 0

while ($attempt -lt $maxAttempts) {
    $statusPage = Invoke-WebRequest -Uri "$convertxUrl/results/$jobId" `
        -Method Get `
        -WebSession $session
    
    if ($statusPage.Content -match "completed") {
        Write-Host "Conversion completed!" -ForegroundColor Green
        break
    }
    
    $attempt++
    Write-Host "Still processing... ($attempt/$maxAttempts)"
    Start-Sleep -Seconds 5
}

if ($attempt -ge $maxAttempts) {
    Write-Host "Conversion timeout" -ForegroundColor Red
    exit 1
}

# Download archive
Write-Host "Downloading files..." -ForegroundColor Yellow
New-Item -ItemType Directory -Force -Path $outputDir | Out-Null

Invoke-WebRequest -Uri "$convertxUrl/archive/$jobId" `
    -Method Get `
    -WebSession $session `
    -OutFile "$outputDir\converted_$jobId.tar"

Write-Host "Downloaded to: $outputDir\converted_$jobId.tar" -ForegroundColor Green
Write-Host "Conversion complete!" -ForegroundColor Green
```

---

## Monitoring & Health Checks

### Basic Health Check Script

```bash
#!/bin/bash
CONVERTX_URL="http://localhost:3000"

# Health check
response=$(curl -s -o /dev/null -w "%{http_code}" "$CONVERTX_URL/healthcheck")

if [ "$response" = "200" ]; then
    echo "OK: ConvertX is healthy"
    exit 0
else
    echo "CRITICAL: ConvertX returned status $response"
    exit 2
fi
```

### Prometheus Monitoring Integration

```python
from prometheus_client import start_http_server, Gauge, Counter
import time
import requests

# Metrics
convertx_status = Gauge('convertx_status', 'ConvertX service status (1=up, 0=down)')
conversion_count = Counter('convertx_conversions_total', 'Total number of conversions')
conversion_errors = Counter('convertx_conversion_errors_total', 'Total conversion errors')

def check_health(url):
    try:
        response = requests.get(f"{url}/healthcheck", timeout=5)
        return response.status_code == 200
    except:
        return False

if __name__ == '__main__':
    # Start Prometheus metrics server
    start_http_server(8000)
    
    convertx_url = "http://localhost:3000"
    
    while True:
        # Check health
        is_healthy = check_health(convertx_url)
        convertx_status.set(1 if is_healthy else 0)
        
        time.sleep(30)
```

### Nagios/Icinga Plugin

```bash
#!/bin/bash
# check_convertx.sh - Nagios plugin for ConvertX

CONVERTX_URL="${1:-http://localhost:3000}"
TIMEOUT=10

# States
STATE_OK=0
STATE_WARNING=1
STATE_CRITICAL=2
STATE_UNKNOWN=3

response=$(curl -s -o /dev/null -w "%{http_code}:%{time_total}" --max-time $TIMEOUT "$CONVERTX_URL/healthcheck" 2>&1)
exit_code=$?

if [ $exit_code -ne 0 ]; then
    echo "CRITICAL - Cannot connect to ConvertX at $CONVERTX_URL"
    exit $STATE_CRITICAL
fi

http_code=$(echo "$response" | cut -d: -f1)
response_time=$(echo "$response" | cut -d: -f2)

if [ "$http_code" = "200" ]; then
    echo "OK - ConvertX is healthy | response_time=${response_time}s"
    exit $STATE_OK
else
    echo "CRITICAL - ConvertX returned HTTP $http_code"
    exit $STATE_CRITICAL
fi
```

---

## Error Handling & Retry Logic

### Python Retry Decorator

```python
import time
from functools import wraps

def retry(max_attempts=3, delay=5, backoff=2, exceptions=(Exception,)):
    """
    Retry decorator with exponential backoff
    """
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            attempt = 0
            current_delay = delay
            
            while attempt < max_attempts:
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    attempt += 1
                    if attempt >= max_attempts:
                        raise
                    
                    print(f"Attempt {attempt} failed: {e}")
                    print(f"Retrying in {current_delay} seconds...")
                    time.sleep(current_delay)
                    current_delay *= backoff
            
        return wrapper
    return decorator

# Usage
class RobustConvertXClient(ConvertXAutomation):
    @retry(max_attempts=3, delay=2, exceptions=(requests.RequestException,))
    def authenticate(self, email, password):
        return super().authenticate(email, password)
    
    @retry(max_attempts=5, delay=5, exceptions=(requests.RequestException,))
    def upload_files(self, file_paths):
        return super().upload_files(file_paths)
    
    @retry(max_attempts=3, delay=3, exceptions=(requests.RequestException,))
    def download_archive(self, job_id, output_path):
        return super().download_archive(job_id, output_path)
```

### Circuit Breaker Pattern

```python
import time
from enum import Enum

class CircuitState(Enum):
    CLOSED = 1
    OPEN = 2
    HALF_OPEN = 3

class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failures = 0
        self.last_failure_time = None
        self.state = CircuitState.CLOSED
    
    def call(self, func, *args, **kwargs):
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.timeout:
                self.state = CircuitState.HALF_OPEN
            else:
                raise Exception("Circuit breaker is OPEN")
        
        try:
            result = func(*args, **kwargs)
            self.on_success()
            return result
        except Exception as e:
            self.on_failure()
            raise e
    
    def on_success(self):
        self.failures = 0
        self.state = CircuitState.CLOSED
    
    def on_failure(self):
        self.failures += 1
        self.last_failure_time = time.time()
        
        if self.failures >= self.failure_threshold:
            self.state = CircuitState.OPEN

# Usage
breaker = CircuitBreaker(failure_threshold=5, timeout=60)

def safe_convert(client, *args, **kwargs):
    return breaker.call(client.convert_files, *args, **kwargs)
```

---

## Rate Limiting & Best Practices

### Best Practices

1. **Reuse Sessions**: Maintain persistent HTTP sessions to avoid repeated authentication
2. **Connection Pooling**: Use connection pools for concurrent requests
3. **Timeout Configuration**: Always set reasonable timeouts (10-30s for API calls)
4. **Graceful Degradation**: Handle service unavailability gracefully
5. **Logging**: Log all API interactions for debugging
6. **Cleanup**: Always delete jobs after downloading to free storage

### Rate Limiting (Client-Side)

```python
import time
from collections import deque

class RateLimiter:
    def __init__(self, max_requests, time_window):
        self.max_requests = max_requests
        self.time_window = time_window
        self.requests = deque()
    
    def wait_if_needed(self):
        now = time.time()
        
        # Remove old requests outside time window
        while self.requests and self.requests[0] < now - self.time_window:
            self.requests.popleft()
        
        # Check if we've hit the limit
        if len(self.requests) >= self.max_requests:
            sleep_time = self.time_window - (now - self.requests[0])
            if sleep_time > 0:
                print(f"Rate limit reached, waiting {sleep_time:.2f}s")
                time.sleep(sleep_time)
        
        self.requests.append(time.time())

# Usage: max 10 requests per 60 seconds
rate_limiter = RateLimiter(max_requests=10, time_window=60)

for file in files:
    rate_limiter.wait_if_needed()
    client.upload_file(file)
```

### Concurrent Processing

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

def convert_file_batch(files, target_format, converter, max_workers=5):
    """Convert multiple files concurrently"""
    
    results = []
    errors = []
    
    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        # Submit all tasks
        future_to_file = {
            executor.submit(
                client.convert_files,
                [file],
                target_format,
                converter,
                "/tmp/output"
            ): file
            for file in files
        }
        
        # Process completed tasks
        for future in as_completed(future_to_file):
            file = future_to_file[future]
            try:
                result = future.result()
                results.append((file, result))
                print(f"✓ {file}: {result}")
            except Exception as e:
                errors.append((file, str(e)))
                print(f"✗ {file}: {e}")
    
    return results, errors

# Usage
files = [f"file{i}.png" for i in range(20)]
results, errors = convert_file_batch(
    files,
    target_format="jpg",
    converter="imagemagick",
    max_workers=3  # Limit concurrent conversions
)

print(f"\nSuccessful: {len(results)}, Failed: {len(errors)}")
```

---

## Additional Resources

- **API Documentation**: See `API_ENDPOINTS.md` for complete endpoint reference
- **Environment Variables**: See README.md for configuration options
- **Supported Formats**: Visit `/converters` endpoint for full list
- **Docker Deployment**: See README.md for Docker setup instructions

---

## Troubleshooting

### Common Issues

**Authentication fails with 401**
- Check credentials are correct
- Ensure JWT_SECRET is set consistently
- Verify cookies are being stored and sent

**File upload fails**
- Check file size limits (default: unlimited)
- Verify jobId cookie is set (visit home page first)
- Ensure file paths are accessible

**Conversion timeout**
- Large files take longer to convert
- Increase `max_wait_time` parameter
- Check ConvertX server logs for errors
- Verify converter binaries are installed

**Download fails with 302 redirect**
- Ensure user is authenticated
- Verify job belongs to authenticated user
- Check job ID is correct

**Archive is empty or corrupt**
- Wait for conversion to fully complete
- Check disk space on ConvertX server
- Verify all files converted successfully

---

## Support

For issues related to ConvertX itself, visit the upstream repository:
- **GitHub**: https://github.com/C4illin/ConvertX
- **Issues**: https://github.com/C4illin/ConvertX/issues

For API integration questions, refer to the `API_ENDPOINTS.md` documentation included in this repository.
