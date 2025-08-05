# Spacelift 💖 Tailscale

Puts [Tailscale][] into [Spacelift][], for accessing things on the tailnet from Terraform, etc easily.

[Tailscale]: https://tailscale.com/
[Spacelift]: https://spacelift.io/

Based on the base AWS spacelift image, with tailscale added to save re-downloading every run.

The Readme is written mentioning Terraform but it will work out the box for OpenTofu, Pulumi, Ansible, etc as well. The original commands defined in your Spacelift workflow are still invoked by Spacelift, we just wrap some setup/teardown around them for Tailscale.

See Howee Dunnit below for implementation details.

## Usage

There is some up front configuration required, then it'll Just Work™ every time you trigger a run in Spacelift for that stack.

There's three things that need configuring, the `runner_image` for the stack, some before/after phase hooks and the `TS_AUTH_KEY` for authenticating to the Tailnet.

Spacelift has multiple ways of configuring these settings, see [Configuration][] documentation for more info. Below is a suggested way to configure it, but not essential.

[Configuration]: https://docs.spacelift.io/concepts/configuration/

This mechanism relies on Terraform providers using HTTP libraries that pay attention to the `http_proxy` environment variable for using a HTTP Proxy to communicate via. The default `net/http` library in Golang's stdlib does pay attention to this, so providers like `hashicorp/nomad` Just Work™ by pointing at the tailscale MagicDNS hostname of a nomad server.

(If you're using Tailscale Serve to expose the endpoint the terraform provider needs the full MagicDNS hostname, including the Tailscale domain.)

### Context for hooks & auth

If you manage Spacelift via Terraform, lean on [caius/terraform-spacelift-tailscale](https://registry.terraform.io/modules/caius/tailscale/spacelift/latest) module to setup a context for you for the hooks. You can also specify an `autoattach:` label on the Context to be able to easily associate it with Stacks.

Otherwise you'll need to create a Spacelift Context in the UI and define the following hooks for the before phases (plan/perform/apply/destroy):

- `spacetail up`
- `trap 'spacetail down' EXIT`
- `export HTTP_PROXY=http://127.0.0.1:8080 HTTPS_PROXY=http://127.0.0.1:8080`

And then in the after hooks for all the above phases, the following:

- `unset HTTP_PROXY HTTPS_PROXY`
- `sed -e '/HTTP_PROXY=/d' -e /HTTPS_PROXY/d -i /mnt/workspace/.env_hooks_after` (Due to https://github.com/caius/spacelift-tailscale/issues/14)

The authentication environment variables below can be ClickOps'd into this context as well:

**For OAuth authentication (recommended):**
- `TS_OAUTH_CLIENT_ID`
- `TS_OAUTH_CLIENT_SECRET`
- `TS_OAUTH_TAGS` (optional)

**For direct auth key authentication:**
- `TS_AUTH_KEY`

### Runner Image

The `runner_image` needs configuring through either `.spacelift/config.yml` or the Spacelift Stack Settings UI.

#### Published Images

Pre-built images are automatically published to GitHub Container Registry with the following tags:

- `ghcr.io/kriskit/spacelift-tailscale:latest` - Latest stable release
- `ghcr.io/kriskit/spacelift-tailscale:sha-<commit>` - Specific commit SHA
- `ghcr.io/kriskit/spacelift-tailscale:v<version>` - Semantic version tags

**Recommended configuration:**

```yaml
stacks:
  my-tailnet-stack:
    runner_image: "ghcr.io/kriskit/spacelift-tailscale:latest"
```

**For production environments**, consider pinning to a specific version or SHA for reproducible builds:

```yaml
stacks:
  my-tailnet-stack:
    runner_image: "ghcr.io/kriskit/spacelift-tailscale:v1.0.0"
```

#### Image Optimization

The Docker image uses a multi-stage build process that provides:

- **50% smaller image size** (658MB vs 1.32GB) by eliminating the Go toolchain from the runtime
- **Multi-platform support** for linux/amd64 and linux/arm64 architectures
- **Layer caching** for faster builds and deployments
- **Security improvements** with minimal attack surface in the runtime image

The build process compiles the `get-authkey` utility in a separate Go builder stage, then copies only the compiled binary to the final runtime image based on the official Spacelift runner.

### Tailnet Authentication

Configuration is via various envariables in the Spacelift runner container, "inspired"[^2] by tailscale's `containerboot` binary.

[^2]: copied from. Build on the shoulders of giants, and be consistent.

#### Option 1: OAuth Client Authentication (Recommended)

This method uses OAuth client credentials to generate auth keys automatically, eliminating the need to manually rotate tokens every 90 days.

Required configuration:

- `TS_OAUTH_CLIENT_ID` - Tailscale OAuth client ID
- `TS_OAUTH_CLIENT_SECRET` - Tailscale OAuth client secret

Optional configuration:

- `TS_OAUTH_TAGS` - Comma-separated list of tags to apply to generated auth keys (default: `tag:spacelift`)

To set up OAuth authentication:

1. Create an OAuth client in the [Tailscale admin console](https://login.tailscale.com/admin/settings/oauth)
2. Grant the `auth_keys` scope and select appropriate tags (e.g., `tag:spacelift`)
3. Copy the client ID and secret to your Spacelift context

#### Option 2: Direct Auth Key (Legacy)

Required configuration:

- `TS_AUTH_KEY` - Tailscale auth key (Suggest creating ephemeral & tagged key)

**Note:** Auth keys expire after 90 days maximum and require manual rotation. Consider using OAuth authentication instead.

#### Common Optional Configuration

- `TS_EXTRA_ARGS` - Extra arguments to pass to `tailscale up`. eg, `--ssh` for debugging inside the spacelift container
- `TS_TAILSCALED_EXTRA_ARGS` - Extra arguments to pass to `tailscaled`. eg, `--socks5-server=localhost:1081` to change socks5 port
- `TRACE` - set to non-empty (eg, "1") to debug `spacetail` script

As above we suggest setting these directly on the Context so any Stack you attach the Context to will be able to access the Tailnet.

#### Environment Variables Summary

| Variable | Required | Description | Default |
|----------|----------|-------------|---------|
| `TS_OAUTH_CLIENT_ID` | Yes (OAuth) | Tailscale OAuth client ID | - |
| `TS_OAUTH_CLIENT_SECRET` | Yes (OAuth) | Tailscale OAuth client secret | - |
| `TS_OAUTH_TAGS` | No | Tags for generated auth keys | `tag:spacelift` |
| `TS_AUTH_KEY` | Yes (Legacy) | Direct Tailscale auth key | - |
| `TS_EXTRA_ARGS` | No | Extra arguments for `tailscale up` | `--accept-dns=false --hostname=spacelift-$(hostname)` |
| `TS_TAILSCALED_EXTRA_ARGS` | No | Extra arguments for `tailscaled` | `--socks5-server=localhost:1080 --outbound-http-proxy-listen=localhost:8080` |
| `TRACE` | No | Enable debug logging | - |

#### Migration from Auth Keys to OAuth

If you're currently using `TS_AUTH_KEY`, you can migrate to OAuth authentication to eliminate manual token rotation:

1. **Create OAuth Client**: In the [Tailscale admin console](https://login.tailscale.com/admin/settings/oauth), create a new OAuth client with:
   - Scope: `auth_keys`
   - Tags: Same tags you used for your manual auth keys (e.g., `tag:spacelift`)

2. **Update Spacelift Context**: Replace your existing environment variables:
   ```bash
   # Remove (or keep as backup during migration)
   # TS_AUTH_KEY=tskey-auth-...

   # Add OAuth credentials
   TS_OAUTH_CLIENT_ID=your-client-id
   TS_OAUTH_CLIENT_SECRET=your-client-secret
   TS_OAUTH_TAGS=tag:spacelift  # Optional, defaults to tag:spacelift
   ```

3. **Test**: Run a Spacelift plan/apply to verify OAuth authentication works

4. **Cleanup**: Once confirmed working, remove the old `TS_AUTH_KEY` from your context

**Benefits of OAuth over Auth Keys:**
- No more 90-day expiration limits
- Automatic token generation and renewal
- Better security through scoped access
- Audit trail of token generation

## Building from Source

If you prefer to build your own Docker image instead of using the pre-built ones:

### Prerequisites

- Docker with BuildKit support
- Make (optional, for convenience)

### Build Commands

```bash
# Clone the repository
git clone https://github.com/kriskit/spacelift-tailscale.git
cd spacelift-tailscale

# Build the image locally
make docker-build

# Or build manually with Docker
docker build -t my-spacelift-tailscale .

# Build for multiple platforms (requires buildx)
docker buildx build --platform linux/amd64,linux/arm64 -t my-spacelift-tailscale .
```

### Multi-stage Build Details

The Dockerfile uses a multi-stage build process:

1. **Builder stage**: Uses `golang:alpine` to compile the `get-authkey` utility
2. **Runtime stage**: Uses the official Spacelift runner image with only the compiled binary

This approach significantly reduces the final image size while maintaining all functionality.

### CI/CD Pipeline

The project uses GitHub Actions for automated building and publishing:

- **Automated builds** on every push to main branch and pull requests
- **Multi-platform images** built for linux/amd64 and linux/arm64
- **Semantic versioning** with automatic tag generation
- **GitHub Container Registry** publishing with multiple tag formats
- **Build optimization** with layer caching and build reports

#### Available Tags

The CI/CD pipeline generates several tag formats:

- `latest` - Latest build from main branch
- `v<major>.<minor>.<patch>` - Semantic version tags (e.g., `v1.0.0`)
- `v<major>.<minor>` - Major.minor tags (e.g., `v1.0`)
- `v<major>` - Major version tags (e.g., `v1`)
- `sha-<commit>` - Specific commit SHA tags
- `pr-<number>` - Pull request builds

#### Build Metrics

Each build provides detailed metrics including:

- Image size optimization (50% reduction vs single-stage)
- Multi-platform compatibility
- Build time and caching efficiency
- Security scan results

## Howee Dunnit

Spacelift runs terraform (or other tooling) in containers, and overrides the initial command run in each container. The `/mnt/workspace` directory is mounted into each container and the environment variables are the same as the phases run.

Tailscale needs `tailscaled` running, which we can start in a `before_` phase hook in Spacelift. The tricky bit is we need to stop it before the phase ends, otherwise Spacelift will wait for the phase to time out in the case of the terraform command erroring, and also won't call any of the `after_` phase hooks. (This is due to how Spacelift executes everything, usually this is what you want!)

To work around this, we use a shell `trap` in the `before_` phase hooks to define a command to execute when the shell exits. We use this to stop tailscaled regardless of whether the terraform command errored or not. This means the container exits fairly quickly on completion and Spacelift can deal with the success or failure therein.

Due to running tailscaled with userspace networking, we don't get MagicDNS wiring up requests for us. Packets are routed to the correct IPs without us having to do anything however, so we just need to solve the DNS issue.

The suggested solution from Tailscale documentation is to use either a SOCKS5 or HTTP Proxy. We run http proxy on `localhost:8080` and socks5 on `localhost:1080` in the container by default, so that's likely the easiest way to go. This requires the running process to be able to use either proxy to make connections via. Anything using Go's `net/http` library should be able to use it automatically, which includes Terraform Providers hitting HTTP APIs.

## License

See [LICENSE](./LICENSE) file.
