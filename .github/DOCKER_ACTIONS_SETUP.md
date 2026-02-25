# GitHub Actions Docker Setup

## Quick Start

The GitHub Actions workflows are configured in `.github/workflows/`. Two workflows are included:

### 1. `docker.yml` - Main Build & Push Pipeline
- **Trigger:** Push to main/develop, PRs, manual dispatch
- **Steps:**
  - Validates docker-compose.yml syntax
  - Lints Dockerfiles with hadolint
  - Builds multi-service images using buildx with layer caching
  - Scans images with Docker Scout for vulnerabilities
  - Pushes to ghcr.io (GitHub Container Registry)
  - Tests services with health checks

### 2. `docker-security.yml` - Security Scanning
- **Trigger:** Push to main, weekly schedule
- **Steps:**
  - Trivy filesystem and image scanning
  - Hadolint linting
  - Uploads results to GitHub Security tab

## Setup Instructions

### 1. Enable GitHub Container Registry (ghcr.io)

1. Go to **Settings** → **Packages and registries** → **Actions**
2. Ensure "Allow write access to packages" is enabled
3. The workflow uses `GITHUB_TOKEN` automatically (no extra secrets needed)

### 2. (Optional) Configure Docker Hub Push

To also push to Docker Hub, add these secrets to your repository:

1. Go to **Settings** → **Secrets and variables** → **Actions**
2. Create new secrets:
   - `DOCKERHUB_USERNAME`: Your Docker Hub username
   - `DOCKERHUB_TOKEN`: Your Docker Hub personal access token

3. Update the `docker.yml` workflow to include Docker Hub login:

```yaml
- name: Log in to Docker Hub
  if: github.event_name != 'pull_request'
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}
```

### 3. (Optional) Configure Docker Build Cloud

For faster builds across multiple runners, enable Docker Build Cloud:

1. Go to [Docker Build Cloud](https://app.docker.com/build-cloud)
2. Create a build config for this repository
3. Add `DOCKER_BUILD_CLOUD` secret with your build token
4. Update docker.yml to use buildx with Docker Build Cloud

### 4. Set Environment Variables

The workflows are pre-configured to use:
- `REGISTRY: ghcr.io` - GitHub Container Registry
- `IMAGE_NAME: ${{ github.repository }}` - Auto-populated repo name

These are already set in the workflow files.

## Workflow Customization

### Modify Trigger Events
Edit the `on:` section in workflows to control when they run:

```yaml
on:
  push:
    branches: [main]
    tags: ['v*']  # Add for semantic versioning
  pull_request:
  workflow_dispatch:
```

### Add Manual Approval Before Push
Add a job dependency:

```yaml
needs: [validate, build, approval]  # Create approval job first
```

### Push to Multiple Registries
Add additional login steps and modify build output.

### Customize Service Build Matrix
The `build` job has a `strategy.matrix.service` list:

```yaml
matrix:
  service:
    - n8n
    - ollama
    - open-webui
    - <add_more_services>
```

## Image Tags & Naming

Images are tagged automatically based on:
- **Branch:** `ghcr.io/user/repo/service:branch-name`
- **Semantic version:** `ghcr.io/user/repo/service:v1.2.3`
- **Latest:** `ghcr.io/user/repo/service:latest` (main branch only)
- **Commit SHA:** `ghcr.io/user/repo/service:branch-sha12345`

## Health Checks

The `test` job verifies services are running. Customize health checks by modifying:

```yaml
- name: Health check - n8n
  run: docker compose exec -T n8n curl -f http://localhost:5678 || exit 1
```

Replace with your service's actual health endpoint.

## Monitoring & Debugging

1. **View workflow runs:** Repository → **Actions** tab
2. **View logs:** Click a workflow run, then click a job
3. **Rerun failed jobs:** Use the "Re-run" button on GitHub
4. **Scout results:** **Security** → **Code scanning** → **Docker Scout**
5. **Trivy results:** **Security** → **Code scanning** → **Trivy**

## GPU Profile Notes

This starter kit includes `--profile gpu-nvidia` for GPU-enabled services. The CI/CD workflows currently:
- Run on standard GitHub runners (no GPU)
- Can be modified to use self-hosted runners with GPU support

To enable GPU in CI:
1. Set up a self-hosted runner with NVIDIA GPU
2. Update the `runs-on` field:
   ```yaml
   runs-on: [self-hosted, linux, gpu]
   ```

## Best Practices

✅ **Layer Caching:** Workflows use GitHub Actions cache (gha mode) for buildx to speed up builds

✅ **Build Arguments:** Include `BUILD_DATE` and `VCS_REF` for image traceability

✅ **Security Scanning:** Both Docker Scout and Trivy run automatically

✅ **Health Checks:** Services are tested before being pushed to registry

✅ **Parallel Builds:** Services build in parallel using `strategy.matrix`

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `docker compose config` fails | Check `.env` file syntax and compose.yml formatting |
| Build timeouts | Enable Docker Build Cloud or use self-hosted runners |
| Authentication fails | Verify GITHUB_TOKEN is available (should be automatic) |
| Scout scan fails | Check Docker Scout is enabled in Docker CLI |
| Health checks timeout | Increase sleep time or adjust health endpoints |

## Next Steps

1. Commit `.github/workflows/docker.yml` and `.github/workflows/docker-security.yml`
2. Push to your repository
3. Monitor the first run in the **Actions** tab
4. Adjust health check endpoints and service names as needed
5. Review security scan results under **Security** tab
