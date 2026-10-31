Below is a production-style **multi-stage Dockerfile** for an ASP.NET Core Web API, followed by a line-by-line explanation of each stage.

## 1. Project layout assumed

```
MySolution/
├── MyApi.sln
├── Dockerfile            ← at solution root
├── .dockerignore
├── src/
│   ├── MyApi/            (Web API project, MyApi.csproj)
│   └── MyApi.Core/       (class library referenced by MyApi)
```

Putting the Dockerfile at the solution root gives the build access to every referenced project.

## 2. The Dockerfile

```dockerfile
# syntax=docker/dockerfile:1

# ============ Stage 1: BUILD ============
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src

# 1. Copy only the project files first (for layer caching)
COPY ["src/MyApi/MyApi.csproj", "src/MyApi/"]
COPY ["src/MyApi.Core/MyApi.Core.csproj", "src/MyApi.Core/"]

# 2. Restore NuGet packages
RUN dotnet restore "src/MyApi/MyApi.csproj"

# 3. Copy the remaining source code
COPY . .

# 4. Publish (compile and produce the deployable output)
WORKDIR /src/src/MyApi
RUN dotnet publish "MyApi.csproj" \
    -c $BUILD_CONFIGURATION \
    -o /app/publish \
    --no-restore \
    /p:UseAppHost=false

# ============ Stage 2: RUNTIME ============
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS final
WORKDIR /app

# Port the app listens on inside the container
ENV ASPNETCORE_HTTP_PORTS=8080 \
    ASPNETCORE_ENVIRONMENT=Production
EXPOSE 8080

# Copy only the published output from the build stage
COPY --from=build /app/publish .

# Run as the built-in non-root user
USER $APP_UID

ENTRYPOINT ["dotnet", "MyApi.dll"]
```

## 3. Step-by-step explanation

**Why multi-stage?**
The SDK image is large (about 800 MB+) because it holds compilers, MSBuild and NuGet. The app only needs the ASP.NET Core runtime to run, so you compile in one stage and copy just the output into a small runtime image (about 220 MB, or less with the chiseled variant). Source code, the SDK and intermediate files never ship to production.

| Instruction | What it does and why |
|---|---|
| `# syntax=docker/dockerfile:1` | Uses the latest BuildKit Dockerfile syntax, which enables features like cache mounts. |
| `FROM ...sdk:10.0 AS build` | Base image with the full .NET SDK. `AS build` names the stage so later stages can reference it. |
| `ARG BUILD_CONFIGURATION=Release` | A build-time variable. You can override it with `--build-arg BUILD_CONFIGURATION=Debug`. |
| `WORKDIR /src` | Sets the working directory and creates it if needed. Later commands run relative to it. |
| `COPY *.csproj` **before** source | **Key optimization.** Docker caches each layer. `.csproj` files rarely change, so the slow `dotnet restore` layer stays cached, and code-only changes skip the restore. |
| `RUN dotnet restore` | Downloads NuGet packages for the project and all referenced projects. |
| `COPY . .` | Copies the rest of the source. `.dockerignore` controls what's excluded. |
| `dotnet publish -c Release -o /app/publish` | Compiles in Release mode and writes the deployable DLLs, `appsettings.json` and other files to `/app/publish`. |
| `--no-restore` | Packages were already restored, so this skips doing it again. |
| `/p:UseAppHost=false` | Skips generating the native executable launcher. You run the app with `dotnet MyApi.dll` instead, which keeps the output smaller. |
| `FROM ...aspnet:10.0 AS final` | A fresh, slim base with only the ASP.NET Core runtime. Nothing from the build stage carries over unless you copy it. |
| `ENV ASPNETCORE_HTTP_PORTS=8080` | Since .NET 8, containers listen on **8080** by default rather than 80, because non-root users can't bind ports below 1024. Setting it explicitly makes this clear. |
| `EXPOSE 8080` | Documentation only. It tells readers and tools which port the app uses. It does **not** publish the port; `-p` does that. |
| `COPY --from=build` | Pulls only the published output out of the build stage. |
| `USER $APP_UID` | .NET 8+ images include a non-root user called `app` (UID 1654). Running as non-root is a security best practice and is often required by AKS policies. |
| `ENTRYPOINT ["dotnet","MyApi.dll"]` | The process that starts the container. The exec (JSON array) form means the app runs as PID 1 and receives `SIGTERM` correctly for graceful shutdown. |

## 4. `.dockerignore` (don't skip this)

```
**/bin/
**/obj/
**/.vs/
**/.vscode/
**/*.user
**/node_modules/
.git/
.gitignore
Dockerfile*
docker-compose*
**/appsettings.Development.json
**/secrets.json
*.md
```

This keeps the build context small, which makes builds faster. It also stops local `bin/obj` folders from polluting the build and keeps secrets out of the image.

## 5. Build and run

```bash
# Build the image (the trailing dot is the build context)
docker build -t myapi:1.0 .

# Run it: map host port 5000 to container port 8080
docker run -d --name myapi -p 5000:8080 \
  -e ConnectionStrings__DefaultConnection="Server=...;" \
  -e ASPNETCORE_ENVIRONMENT=Staging \
  myapi:1.0

# Test it
curl http://localhost:5000/weatherforecast

# Logs and debugging
docker logs -f myapi
docker exec -it myapi /bin/bash
```

**Configuration tip:** a double underscore `__` in an environment variable name maps to `:` in .NET configuration. For example, `ConnectionStrings__DefaultConnection` maps to `ConnectionStrings:DefaultConnection`. Environment variables override `appsettings.json`, so the same image works in every environment.

## 6. Push to Azure Container Registry and deploy

```bash
az acr login --name myregistry
docker tag myapi:1.0 myregistry.azurecr.io/myapi:1.0
docker push myregistry.azurecr.io/myapi:1.0
```

From there you can deploy to Azure Container Apps, App Service for Containers or AKS.

## 7. Production best practices

- **Health checks:** add `builder.Services.AddHealthChecks()` and `app.MapHealthChecks("/health")`, then point Kubernetes or Container Apps liveness and readiness probes at `/health`. The `aspnet` image has no `curl`, so prefer orchestrator probes over a Dockerfile `HEALTHCHECK`.
- **HTTPS:** terminate TLS at the ingress, load balancer or App Gateway, and run plain HTTP on 8080 inside the container. Don't bake certificates into the image.
- **Secrets:** never `COPY` them or set them with `ENV` in the Dockerfile. Use Key Vault, Managed Identity, or secrets injected by the orchestrator.
- **Smaller and more secure images:** `aspnet:10.0-noble-chiseled` is an Ubuntu "distroless" image with no shell or package manager, which means a much smaller attack surface. Alpine variants (`10.0-alpine`) are also small.
- **Pin versions** such as `aspnet:10.0.x` or a digest in CI for reproducible builds.
- **Faster CI restores:** use `RUN --mount=type=cache,target=/root/.nuget/packages dotnet restore ...`.
- **Docker Compose:** use it locally to run the API alongside SQL Server, Redis and similar services.

## 8. Alternative: no Dockerfile at all

The .NET SDK can build container images directly:

```bash
dotnet publish -c Release /t:PublishContainer -p ContainerImageName=myapi
```

It handles base image selection, the non-root user and ports automatically. That's useful for simple APIs. A Dockerfile gives you more control, for example over OS packages, multi-project builds and custom stages.

## Common interview questions on this

- **Why copy the `.csproj` before the rest of the source?** For layer caching, so the restore isn't repeated on every code change.
- **Why multi-stage?** A smaller image, a smaller attack surface, and no SDK or source code in production.
- **`ENTRYPOINT` vs `CMD`?** `ENTRYPOINT` is the fixed executable. `CMD` provides default arguments that `docker run` can override.
- **`EXPOSE` vs `-p`?** `EXPOSE` only documents the port. `-p` actually publishes it to the host.
- **Why port 8080 in .NET 8+?** The images run as non-root by default, and non-root users can't bind privileged ports below 1024.
- **`ARG` vs `ENV`?** `ARG` exists only at build time. `ENV` persists into the running container.
