# Simple ShinyProxy Setup Guide - Complete Workflow

## Part 1: Install and Setup ShinyProxy

### Step 1: Install Docker
```bash
sudo apt update
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io
sudo systemctl start docker
sudo systemctl enable docker
```

### Step 2: Install ShinyProxy
```bash
wget https://www.shinyproxy.io/downloads/shinyproxy_3.2.0_amd64.deb
sudo apt install ./shinyproxy_3.2.0_amd64.deb
```

### Step 3: Configure Docker Permissions
```bash
sudo usermod -a -G docker shinyproxy
sudo usermod -a -G docker $USER
newgrp docker
```

### Step 4: Configure ShinyProxy Port
```bash
sudo nano /etc/shinyproxy/application.yml
```

Change port from 8080 to 8081:
```yaml
proxy:
  port: 8081
```

### Step 5: Start ShinyProxy
```bash
sudo systemctl restart shinyproxy
sudo systemctl enable shinyproxy
sudo systemctl status shinyproxy
```

## Part 2: Build Docker Image

### Step 1: Create Project Directory
```bash
mkdir ~/my-golem-app
cd ~/my-golem-app
```

### Step 2: Create Dockerfile
```bash
nano Dockerfile
```

Add this content:
```dockerfile
FROM rocker/r-ver:4.4.1

# Install system dependencies
RUN apt-get update && apt-get install -y \
    curl \
    gnupg2 \
    apt-transport-https \
    unixodbc \
    unixodbc-dev \
    libssl-dev \
    libxml2-dev \
    libcurl4-openssl-dev \
    && rm -rf /var/lib/apt/lists/*

# Install Microsoft SQL Server drivers (if needed)
RUN curl https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor -o /usr/share/keyrings/microsoft-prod.gpg \
    && echo "deb [arch=amd64,armhf,arm64 signed-by=/usr/share/keyrings/microsoft-prod.gpg] https://packages.microsoft.com/ubuntu/22.04/prod jammy main" > /etc/apt/sources.list.d/mssql-release.list \
    && apt-get update \
    && ACCEPT_EULA=Y apt-get install -y msodbcsql18 mssql-tools18 \
    && rm -rf /var/lib/apt/lists/*

# Install R packages
RUN R -e "install.packages(c( \
    'remotes', 'config', 'curl', 'DBI', 'dplyr', 'DT', 'httr', \
    'loggit', 'odbc', 'readr', 'shiny', 'shinycssloaders', 'shinyjs', \
    'stringr', 'tibble', 'withr', 'zip', 'bslib', 'fakir', 'bsicons', \
    'golem', 'AzureAuth', 'AzureKeyVault', 'AzureRMR', 'AzureStor' \
    ), repos='https://cran.rstudio.com/')"

WORKDIR /app

# Copy your package file
COPY YourPackage_0.0.0.9004.tar.gz /app/

# Install your package
RUN R -e "install.packages('YourPackage_0.0.0.9004.tar.gz', repos = NULL, type = 'source')"

ENV GOLEM_CONFIG_ACTIVE=production
ENV R_CONFIG_ACTIVE=production

EXPOSE 3838

CMD ["R", "-e", "options(golem.app.prod = TRUE); options('shiny.host'='0.0.0.0'); options('shiny.port'=3838); YourPackage::run_app()"]
```

### Step 3: Copy Your Package File
```bash
cp /path/to/your/YourPackage_0.0.0.9004.tar.gz ~/my-golem-app/
```

### Step 4: Build Docker Image
```bash
docker build -t my-golem-app:latest .
```

## Part 3: Test Docker Image

### Step 1: Run Container Standalone
```bash
docker run --rm -p 3838:3838 my-golem-app:latest
```

### Step 2: Test in Browser
Open: `http://localhost:3838`

### Step 3: Stop Container
Press `Ctrl+C` or in another terminal:
```bash
docker stop $(docker ps -q --filter ancestor=my-golem-app:latest)
```

## Part 4: Add to ShinyProxy

### Step 1: Edit ShinyProxy Configuration
```bash
sudo nano /etc/shinyproxy/application.yml
```

### Step 2: Add Your App to specs Section
Add this to the `specs:` section:
```yaml
specs:
  - id: my-golem-app
    display-name: "My Golem App"
    description: "Your custom Golem application"
    container-image: my-golem-app:latest
    port: 3838
    access-groups: [ scientists, mathematicians ]
```

### Step 3: Restart ShinyProxy
```bash
sudo systemctl restart shinyproxy
```

### Step 4: Check Status
```bash
sudo systemctl status shinyproxy
```

### Step 5: Access Web Interface
Open: `http://localhost:8081`
Login: username `jack`, password `password`

## Part 5: Check Logs

### Method 1: ShinyProxy Service Logs
```bash
# Real-time logs
sudo journalctl -u shinyproxy -f

# Recent logs
sudo journalctl -u shinyproxy --since "10 minutes ago"
```

### Method 2: ShinyProxy Log File
```bash
sudo tail -f /var/log/shinyproxy/shinyproxy.log
```

### Method 3: Container Logs (when app is running)
```bash
# List running containers
docker ps

# Get logs from specific container
docker logs -f <container_id>
```

### Method 4: Enable Container Log Storage
Add to `application.yml`:
```yaml
proxy:
  store-container-logs: true
  container-log-path: /var/log/shinyproxy/container-logs
```

Then restart and view:
```bash
sudo systemctl restart shinyproxy
sudo tail -f /var/log/shinyproxy/container-logs/*.log
```

## Quick Reference Commands

### Docker Management
```bash
# List images
docker images

# Remove image
docker rmi my-golem-app:latest

# List containers
docker ps -a

# Remove container
docker rm <container_id>
```

### ShinyProxy Management
```bash
# Restart service
sudo systemctl restart shinyproxy

# Check status
sudo systemctl status shinyproxy

# View logs
sudo journalctl -u shinyproxy -f
```

### Update Workflow
1. Make changes to your Golem app
2. Create new package: `YourPackage_0.0.0.9005.tar.gz`
3. Update Dockerfile with new version
4. Rebuild: `docker build -t my-golem-app:v2 .`
5. Update `application.yml` with new image: `my-golem-app:v2`
6. Restart ShinyProxy: `sudo systemctl restart shinyproxy`

## Troubleshooting

### If ShinyProxy won't start
```bash
sudo systemctl status shinyproxy
sudo journalctl -u shinyproxy --since "5 minutes ago"
```

### If Docker permission denied
```bash
sudo usermod -a -G docker $USER
newgrp docker
```

### If container won't start
```bash
docker run --rm -it my-golem-app:latest /bin/bash
docker run --rm -it my-golem-app:latest R -e "library(YourPackage)"
```

### If app doesn't load in ShinyProxy
```bash
# Check ShinyProxy logs
sudo journalctl -u shinyproxy -f

# Check container logs
docker logs -f $(docker ps -q --filter ancestor=my-golem-app:latest)
```
