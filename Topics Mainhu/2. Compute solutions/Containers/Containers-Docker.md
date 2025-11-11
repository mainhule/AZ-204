# 🐳 Docker Best Practices

## 📦 Base Images

### Voor .NET Applicaties

**SDK Image** (voor builden):
```
mcr.microsoft.com/dotnet/sdk:6.0
mcr.microsoft.com/dotnet/sdk:7.0
mcr.microsoft.com/dotnet/sdk:8.0
```

**ASP.NET Runtime Image** (voor draaien):
```
mcr.microsoft.com/dotnet/aspnet:6.0
mcr.microsoft.com/dotnet/aspnet:7.0
mcr.microsoft.com/dotnet/aspnet:8.0
```

**Console Runtime Image** (voor console apps):
```
mcr.microsoft.com/dotnet/runtime:6.0
```

### Keuze Criteria

| Image Type | Gebruik | Size | Bevat |
|-----------|---------|------|-------|
| SDK | Development, building | ~700 MB | Compiler, tools, runtime |
| ASP.NET Runtime | Web applicaties | ~200 MB | ASP.NET libraries, runtime |
| Runtime | Console apps | ~180 MB | .NET runtime alleen |

---

## 🏗️ Multi-stage Builds

Multi-stage builds scheiden de build en runtime environments, wat resulteert in **kleinere, veiligere images**.

### Voordelen

✅ **Kleinere images:** Alleen runtime dependencies in final image  
✅ **Betere security:** Build tools niet in productie image  
✅ **Snellere deployments:** Kleinere images = sneller downloaden  
✅ **Duidelijke scheiding:** Build stage vs Runtime stage  
✅ **Layer caching:** Efficiënte builds door layers

---

## 🔨 Multi-stage Build Structuur

### Build Stage (Compile)

1. Gebruik SDK image voor build tools
2. Kopieer project files en restore dependencies
3. Kopieer source code
4. Compileer en publish applicatie

### Runtime Stage (Production)

1. Gebruik lightweight runtime image
2. Kopieer alleen compiled artifacts van build stage
3. Stel entrypoint in
4. Configureer exposed ports

---

## 📋 Voorbeeld 1: Simpele Multi-stage Build

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /app

# Copy project file en restore dependencies
COPY *.csproj ./
RUN dotnet restore

# Copy source en build
COPY . ./
RUN dotnet publish -c Release -o out

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:6.0
WORKDIR /app

# Copy artifacts van build stage
COPY --from=build /app/out .

# Start applicatie
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

**Waarom werkt dit?**
- Dependencies worden eerst gekopieerd en gerestored (layer caching)
- Source code wordt apart gekopieerd
- Final image bevat alleen runtime + compiled app
- Image size reduction: ~700MB → ~200MB

---

## 📋 Voorbeeld 2: Multi-project Solution

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /app

# Copy solution en alle project files
COPY *.sln .
COPY ProjectA/*.csproj ./ProjectA/
COPY ProjectB/*.csproj ./ProjectB/
COPY WebApp/*.csproj ./WebApp/

# Restore dependencies voor hele solution
RUN dotnet restore

# Copy alle source code
COPY ProjectA/. ./ProjectA/
COPY ProjectB/. ./ProjectB/
COPY WebApp/. ./WebApp/

# Build specifiek project
WORKDIR /app/WebApp
RUN dotnet publish -c Release -o out

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS runtime
WORKDIR /app
COPY --from=build /app/WebApp/out ./
ENTRYPOINT ["dotnet", "WebApp.dll"]
```

---

## 📋 Voorbeeld 3: HTTP en HTTPS Support

```dockerfile
# Base stage - expose ports
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy en restore
COPY ["WebApplication1/WebApplication1.csproj", "WebApplication1/"]
RUN dotnet restore "WebApplication1/WebApplication1.csproj"

# Copy en build
COPY . .
WORKDIR "/src/WebApplication1"
RUN dotnet build "WebApplication1.csproj" -c Release -o /app/build

# Publish stage
FROM build AS publish
RUN dotnet publish "WebApplication1.csproj" -c Release -o /app/publish

# Final stage
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "WebApplication1.dll"]
```

**Features:**
- Separate base stage voor port configuration
- Aparte publish stage voor optimalisatie
- Ondersteunt zowel HTTP (80) als HTTPS (443)

---

## 📋 Voorbeeld 4: Met Build Arguments

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /app

COPY *.csproj ./
RUN dotnet restore

COPY . ./
RUN dotnet publish -c ${BUILD_CONFIGURATION} -o out

FROM mcr.microsoft.com/dotnet/aspnet:6.0
WORKDIR /app
COPY --from=build /app/out .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

**Build met custom configuration:**
```bash
docker build --build-arg BUILD_CONFIGURATION=Debug -t myapp:debug .
```

---

## 🎯 Dockerfile Instructies Uitleg

### WORKDIR
```dockerfile
WORKDIR /app
```
- Stelt de working directory in waar alle commando's worden uitgevoerd
- Maakt directory automatisch aan als die niet bestaat
- Alle `COPY` en `RUN` commando's gebruiken deze directory

### COPY
```dockerfile
# Copy project files
COPY *.csproj ./

# Copy alles behalve .dockerignore
COPY . .

# Copy van andere stage
COPY --from=build /app/out ./

# Copy specifieke folder
COPY ["WebApp/WebApp.csproj", "WebApp/"]
```

### RUN
```dockerfile
# Restore dependencies
RUN dotnet restore

# Build applicatie
RUN dotnet build -c Release

# Publish applicatie
RUN dotnet publish -c Release -o out

# Multiple commands
RUN apt-get update && apt-get install -y curl
```

### EXPOSE
```dockerfile
EXPOSE 80
EXPOSE 443
EXPOSE 5000
```
- Documenteert welke poorten de container gebruikt
- Alleen voor documentatie, opent poort niet automatisch
- Gebruik `-p` flag bij `docker run` om daadwerkelijk te exposen

### ENTRYPOINT
```dockerfile
# Exec form (preferred)
ENTRYPOINT ["dotnet", "MyApp.dll"]

# Shell form
ENTRYPOINT dotnet MyApp.dll
```

### CMD vs ENTRYPOINT
```dockerfile
# ENTRYPOINT = hoofdcommando
ENTRYPOINT ["dotnet", "MyApp.dll"]

# CMD = default argumenten (kunnen worden overridden)
CMD ["--environment", "Production"]

# Result: dotnet MyApp.dll --environment Production
```

---

## 🎯 Best Practices

### 1. Layer Caching Optimalisatie

Kopieer dependencies eerst, dan source code (dependencies veranderen minder vaak):

```dockerfile
# ✅ Goed: Dependencies layer wordt gecached
COPY *.csproj ./
RUN dotnet restore

COPY . .
RUN dotnet build

# ❌ Slecht: Hele layer opnieuw bij source change
COPY . .
RUN dotnet restore
RUN dotnet build
```

### 2. Minimale Final Image

```dockerfile
# ✅ Goed: Alleen runtime (~200MB)
FROM mcr.microsoft.com/dotnet/aspnet:6.0

# ❌ Slecht: Bevat SDK (~700MB)
FROM mcr.microsoft.com/dotnet/sdk:6.0
```

### 3. Multi-stage voor Different Targets

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
# ... build steps ...

# Development image met debugging tools
FROM build AS development
RUN dotnet tool install --global dotnet-ef
ENV PATH="${PATH}:/root/.dotnet/tools"

# Production image zonder tools
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS production
COPY --from=build /app/out .
```

**Build specific target:**
```bash
# Development
docker build --target development -t myapp:dev .

# Production
docker build --target production -t myapp:prod .
```

### 4. Non-root User

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:6.0
WORKDIR /app

# Create user
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app

# Switch to non-root user
USER appuser

COPY --chown=appuser:appuser --from=build /app/out .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### 5. Health Checks

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:6.0
WORKDIR /app

COPY --from=build /app/out .

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD curl -f http://localhost:80/health || exit 1

ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### 6. Environment Variables

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:6.0
WORKDIR /app

# Set environment
ENV ASPNETCORE_ENVIRONMENT=Production \
    DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=false \
    ASPNETCORE_URLS=http://+:80

COPY --from=build /app/out .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### 7. .dockerignore File

Maak een `.dockerignore` file in root directory:

```
# Build outputs
bin/
obj/
out/
publish/

# IDE
.vs/
.vscode/
.idea/

# Git
.git/
.gitignore
.gitattributes

# Documentation
*.md
docs/

# Tests
**/Tests/
**/*Tests.csproj

# OS files
.DS_Store
Thumbs.db
```

---

## 🔧 Build en Run Commands

### Image Builden

```bash
# Basic build
docker build -t myapp:v1 .

# Build met naam en tag
docker build -t myregistry.azurecr.io/myapp:1.0.0 .

# Build specific stage
docker build --target build -t myapp:build .

# Build met build arguments
docker build --build-arg BUILD_CONFIGURATION=Release -t myapp:prod .

# Build zonder cache
docker build --no-cache -t myapp:v1 .

# Build met progress
docker build --progress=plain -t myapp:v1 .
```

### Image Runnen

```bash
# Basic run
docker run -p 8080:80 myapp:v1

# Run met naam
docker run --name myapp-container -p 8080:80 myapp:v1

# Run detached
docker run -d -p 8080:80 myapp:v1

# Run met environment variables
docker run -p 8080:80 \
    -e ASPNETCORE_ENVIRONMENT=Production \
    -e ConnectionStrings__Default="..." \
    myapp:v1

# Run met volume
docker run -p 8080:80 \
    -v $(pwd)/data:/app/data \
    myapp:v1

# Run met restart policy
docker run -d --restart unless-stopped -p 8080:80 myapp:v1
```

### Image Management

```bash
# List images
docker images

# Remove image
docker rmi myapp:v1

# Tag image
docker tag myapp:v1 myapp:latest

# Push naar registry
docker push myregistry.azurecr.io/myapp:1.0.0

# Pull van registry
docker pull myregistry.azurecr.io/myapp:1.0.0

# Inspect image
docker inspect myapp:v1

# Image history (layers)
docker history myapp:v1
```

---

## 📂 Working Directory Structuur

### Container File System

```
/app                    # Root van applicatie (WORKDIR)
  ├── MyApp.dll        # Main assembly
  ├── appsettings.json # Configuration
  ├── wwwroot/         # Static files
  │   ├── css/
  │   ├── js/
  │   └── images/
  └── data/            # App data (volume mount)
```

### Typische Paths

```dockerfile
# Application root
WORKDIR /app

# Source code tijdens build
WORKDIR /src

# Build output
WORKDIR /app/build

# Published artifacts
WORKDIR /app/publish
```

### Volume Mounts

```bash
# Local path → Container path
docker run -v C:\data:/app/data myapp:v1
docker run -v $(pwd)/logs:/app/logs myapp:v1
docker run -v myvolume:/app/data myapp:v1
```

---

## 🔍 Troubleshooting

### Image te groot?

```bash
# Check layer sizes
docker history myapp:v1

# Use dive tool voor analyse
dive myapp:v1
```

**Oplossingen:**
- Gebruik multi-stage builds
- Minimize layers (combine RUN commands)
- Use .dockerignore
- Gebruik Alpine variants (waar mogelijk)

### Build fails bij COPY?

**Check .dockerignore:**
```bash
# Verify wat wordt gekopieerd
docker build --no-cache -t myapp:debug . 2>&1 | grep "COPY"
```

### Runtime errors?

```bash
# Run met debug output
docker run --rm myapp:v1 dotnet MyApp.dll --environment Development

# Check logs
docker logs container_id

# Interactive shell
docker run -it --entrypoint /bin/bash myapp:v1
```

---

## 📊 Image Size Vergelijking

| Strategy | Size | Build Time | Security |
|----------|------|------------|----------|
| Single-stage (SDK) | ~700 MB | Fast | ❌ Low |
| Multi-stage (Runtime) | ~200 MB | Medium | ✅ Good |
| Alpine-based | ~100 MB | Slow | ✅ Best |
| Self-contained | ~80 MB | Slow | ✅ Best |

**Recommended:** Multi-stage met ASP.NET runtime image voor beste balans tussen size, performance en compatibility.
