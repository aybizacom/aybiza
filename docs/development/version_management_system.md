# AYBIZA AI Voice Agent Platform Version Management System

This document outlines the automated version management system for the AYBIZA platform, including versioning strategy, automated version bump scripts, and integration with CI/CD pipelines.

## 1. Versioning Strategy

### Semantic Versioning

AYBIZA follows [Semantic Versioning 2.0.0](https://semver.org/) with the format: `MAJOR.MINOR.PATCH`.

- **MAJOR**: Incompatible API changes
- **MINOR**: New functionality in a backward-compatible manner
- **PATCH**: Backward-compatible bug fixes

### Version Lifecycle

1. **Development**: `0.x.y` versions (pre-1.0)
2. **Production-Ready**: `1.0.0+` versions
3. **Pre-release**: Append `-alpha.1`, `-beta.1`, or `-rc.1` for pre-release versions

## 2. Version Storage

### Primary Version Source

The authoritative version is stored in a central `VERSION` file at the root of the repository:

```
# /workspaces/aybiza/VERSION
0.1.0
```

### Propagation to Project Files

The version number is propagated to:

1. `mix.exs` (Elixir project version)
2. Docker image tags
3. CI/CD pipeline variables
4. Documentation

## 3. Automated Version Bump Script

### Version Bump Script (`version_bump.exs`)

```elixir
#!/usr/bin/env elixir

defmodule Aybiza.VersionBump do
  @version_file "VERSION"
  @mix_files [
    "mix.exs",
    "apps/voice_gateway/mix.exs",
    "apps/voice_pipeline/mix.exs",
    "apps/conversation_engine/mix.exs",
    "apps/agent_manager/mix.exs",
    "apps/call_analytics/mix.exs",
    "apps/security/mix.exs"
  ]
  @changelog_file "CHANGELOG.md"
  
  def run(args) do
    bump_type = parse_args(args)
    current_version = read_current_version()
    new_version = bump_version(current_version, bump_type)
    
    IO.puts("Bumping version: #{current_version} -> #{new_version}")
    
    # Update version files
    File.write!(@version_file, new_version)
    update_mix_files(new_version)
    update_changelog(new_version)
    
    # Git operations
    System.cmd("git", ["add", @version_file])
    Enum.each(@mix_files, &System.cmd("git", ["add", &1]))
    System.cmd("git", ["add", @changelog_file])
    System.cmd("git", ["commit", "-m", "Bump version to #{new_version}"])
    System.cmd("git", ["tag", "-a", "v#{new_version}", "-m", "Version #{new_version}"])
    
    IO.puts("Version bumped to #{new_version} and committed")
    IO.puts("Next steps:")
    IO.puts("  1. Review changes (git show)")
    IO.puts("  2. Push changes (git push && git push --tags)")
  end
  
  defp parse_args(args) do
    case args do
      ["major"] -> :major
      ["minor"] -> :minor
      ["patch"] -> :patch
      _ -> 
        IO.puts("Usage: mix run version_bump.exs [major|minor|patch]")
        exit({:shutdown, 1})
    end
  end
  
  defp read_current_version do
    case File.read(@version_file) do
      {:ok, version} -> String.trim(version)
      {:error, _} -> "0.1.0" # Default starting version
    end
  end
  
  defp bump_version(version, type) do
    [major, minor, patch] = 
      version
      |> String.split(".")
      |> Enum.map(&String.to_integer/1)
    
    case type do
      :major -> "#{major + 1}.0.0"
      :minor -> "#{major}.#{minor + 1}.0"
      :patch -> "#{major}.#{minor}.#{patch + 1}"
    end
  end
  
  defp update_mix_files(new_version) do
    Enum.each(@mix_files, fn file_path ->
      if File.exists?(file_path) do
        content = File.read!(file_path)
        updated_content = Regex.replace(~r/version: "([0-9]+\.[0-9]+\.[0-9]+)"/, content, 
                                       "version: \"#{new_version}\"")
        File.write!(file_path, updated_content)
      end
    end)
  end
  
  defp update_changelog(new_version) do
    today = Date.utc_today() |> Date.to_string()
    
    unless File.exists?(@changelog_file) do
      initial_content = """
      # Changelog
      All notable changes to this project will be documented in this file.
      
      The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
      and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).
      
      """
      File.write!(@changelog_file, initial_content)
    end
    
    content = File.read!(@changelog_file)
    
    # Add new version entry
    new_entry = """
    
    ## [#{new_version}] - #{today}
    ### Added
    - 
    
    ### Changed
    - 
    
    ### Fixed
    - 
    
    """
    
    # Insert after the header
    updated_content = Regex.replace(~r/(# Changelog.*?and this project adheres to \[Semantic Versioning\]\(.*?\)\.)/s, 
                                   content, "\\1#{new_entry}")
    
    File.write!(@changelog_file, updated_content)
  end
end

Aybiza.VersionBump.run(System.argv())
```

### Shell Script Wrapper for Easy Use

```bash
#!/bin/bash
# /workspaces/aybiza/bump-version.sh

set -e  # Exit on error

# Validate input
if [ "$#" -ne 1 ] || [[ ! "$1" =~ ^(major|minor|patch)$ ]]; then
    echo "Usage: $0 [major|minor|patch]"
    exit 1
fi

BUMP_TYPE=$1

# Run the Elixir script
mix run version_bump.exs $BUMP_TYPE

# Display success message
echo "✅ Version bump complete!"
echo "Don't forget to run 'git push && git push --tags' to publish changes"
```

## 4. GitHub Actions Integration

Create a GitHub Action to automate version bumping in CI/CD:

```yaml
# .github/workflows/version-bump.yml
name: Version Bump

on:
  workflow_dispatch:
    inputs:
      bump_type:
        description: 'Type of version bump'
        required: true
        default: 'patch'
        type: choice
        options:
          - patch
          - minor
          - major

jobs:
  bump-version:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Set up Elixir
        uses: erlef/setup-beam@v1
        with:
          elixir-version: '1.18.3'
          otp-version: '27.3.4'

      - name: Configure Git
        run: |
          git config --global user.name "GitHub Actions Bot"
          git config --global user.email "actions@github.com"

      - name: Bump Version
        run: |
          mix run version_bump.exs ${{ github.event.inputs.bump_type }}

      - name: Push Changes
        run: |
          git push
          git push --tags
```

## 5. Version Information in App

Create an Elixir module to provide version information:

```elixir
# /workspaces/aybiza/apps/voice_gateway/lib/aybiza/version.ex
defmodule Aybiza.Version do
  @moduledoc """
  Provides version information for the AYBIZA platform.
  """
  
  @version_file Path.join([File.cwd!(), "VERSION"])
  
  @doc """
  Returns the current version of the AYBIZA platform
  """
  def version do
    case File.read(@version_file) do
      {:ok, version} -> String.trim(version)
      {:error, _} -> "unknown"
    end
  end
  
  @doc """
  Returns a map of component versions for the platform
  """
  def components do
    %{
      elixir: System.version(),
      otp: :erlang.system_info(:otp_release) |> List.to_string(),
      phoenix: Application.spec(:phoenix, :vsn) |> to_string(),
      aybiza: version()
    }
  end
  
  @doc """
  Returns the build information for the current release
  """
  def build_info do
    %{
      version: version(),
      build_date: Application.get_env(:aybiza, :build_date, "development"),
      git_sha: Application.get_env(:aybiza, :git_sha, "development")
    }
  end
end
```

## 6. Versioning Docker Images

Update Docker build process to use the VERSION file:

```dockerfile
# Dockerfile modifications
FROM hexpm/elixir:1.18.3-erlang-27.3.4-debian-bookworm-20250517 AS build

# ... existing build steps ...

# Set version information
ARG APP_VERSION
ENV APP_VERSION=${APP_VERSION}
ENV BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ')
ENV GIT_SHA=${GIT_SHA}

# Add version info to compiled app
RUN echo "BUILD_INFO = [app_version: \"${APP_VERSION}\", build_date: \"${BUILD_DATE}\", git_sha: \"${GIT_SHA}\"]" > /app/config/build_info.exs
```

Script to build with versioning:

```bash
#!/bin/bash
# /workspaces/aybiza/scripts/build-docker.sh

VERSION=$(cat VERSION)
GIT_SHA=$(git rev-parse --short HEAD)

docker build \
  --build-arg APP_VERSION=$VERSION \
  --build-arg GIT_SHA=$GIT_SHA \
  -t aybiza/voice-agent:$VERSION \
  -t aybiza/voice-agent:latest \
  -f Dockerfile.prod .

echo "Built aybiza/voice-agent:$VERSION"
```

## 7. Version Update Process

### Standard Process for Developers

1. Complete feature/bugfix work
2. Run tests and ensure all tests pass
3. Bump version:
   ```bash
   ./bump-version.sh patch  # or minor/major as appropriate
   ```
4. Push changes:
   ```bash
   git push && git push --tags
   ```
5. CI/CD pipeline picks up the new tag and builds/deploys accordingly

### Using GitHub Actions UI

1. Navigate to Actions tab in GitHub repository
2. Select "Version Bump" workflow
3. Click "Run workflow"
4. Select branch and bump type (patch/minor/major)
5. Click "Run workflow"

### Version Documentation

Update version details in relevant documentation files:

1. `CHANGELOG.md` (automatically updated by script)
2. `26_version_management_best_practices.md` 
3. `27_version_update_guide.md`

## 8. Release Process

### Release Flow

1. **Development Branch**: Continuous development with patch version increments
2. **Staging Branch**: Minor/major version increments for feature completions
3. **Main Branch**: Production releases with stable versions

### Release Checklist

1. Ensure all tests pass
2. Update CHANGELOG.md with detailed release notes
3. Bump version using appropriate increment
4. Create a GitHub release with release notes
5. Deploy to production

### Hotfix Process

1. Create hotfix branch from main/production branch
2. Fix the issue and test the fix
3. Bump patch version
4. Create pull request back to main/production
5. After merge and deployment, create pull request to development branch

## 9. Version Propagation in Web Interface

Add version information to the UI:

```elixir
# /workspaces/aybiza/apps/voice_gateway/lib/aybiza_web/components/layouts/app.html.heex
<footer class="mt-auto py-4 bg-light">
  <div class="container">
    <div class="d-flex justify-content-between">
      <span>AYBIZA AI Voice Agent Platform</span>
      <span>Version <%= Aybiza.Version.version() %></span>
    </div>
  </div>
</footer>
```

## 10. Integration with Environment-Specific Configurations

Configure version-specific features:

```elixir
# config/runtime.exs
import Config

app_version = Aybiza.Version.version()
version_parts = app_version |> String.split(".") |> Enum.map(&String.to_integer/1)

# Enable features based on version
config :aybiza, :features,
  advanced_analytics: version_compare(version_parts, [0, 2, 0]) >= 0,
  voice_synthesis_v2: version_compare(version_parts, [0, 3, 0]) >= 0,
  multi_agent_conversations: version_compare(version_parts, [1, 0, 0]) >= 0

# Version comparison helper
defp version_compare([major1, minor1, patch1], [major2, minor2, patch2]) do
  cond do
    major1 > major2 -> 1
    major1 < major2 -> -1
    minor1 > minor2 -> 1
    minor1 < minor2 -> -1
    patch1 > patch2 -> 1
    patch1 < patch2 -> -1
    true -> 0
  end
end
```

## 11. Implementation Steps

### First-Time Setup

1. Create the `VERSION` file at the root of your project:
   ```bash
   echo "0.1.0" > VERSION
   ```

2. Create the version script:
   ```bash
   mkdir -p scripts
   # Create version_bump.exs as shown above
   # Create bump-version.sh as shown above
   chmod +x scripts/bump-version.sh
   ```

3. Create the version module:
   ```bash
   # Create Aybiza.Version module as shown above
   ```

4. Set up GitHub Actions workflow:
   ```bash
   mkdir -p .github/workflows
   # Create version-bump.yml as shown above
   ```

5. Create initial CHANGELOG.md:
   ```bash
   # Will be created by the version bump script
   ```

6. Run the initial version bump:
   ```bash
   ./scripts/bump-version.sh patch
   ```

### Integration with Elixir Umbrella Project

For the AYBIZA umbrella project structure, ensure the version script updates all `mix.exs` files in the apps directory.

## 12. Version Rollback Procedure

If a version needs to be rolled back:

1. Check out the previous version tag:
   ```bash
   git checkout v0.2.1  # Example previous version
   ```

2. Create a new patch version:
   ```bash
   ./scripts/bump-version.sh patch
   ```

3. Push changes:
   ```bash
   git push && git push --tags
   ```

## Conclusion

This comprehensive version management system provides:

1. **Consistent Versioning**: Semantic versioning across all components
2. **Automation**: Simple scripts for version bumping
3. **CI/CD Integration**: GitHub Actions for automated version management
4. **Documentation**: Automatic CHANGELOG updates
5. **Visibility**: Version information in UI and logs
6. **Feature Flags**: Version-based feature enabling

By implementing this system, AYBIZA will maintain a clear version history, simplify release management, and ensure all team members follow consistent versioning practices.