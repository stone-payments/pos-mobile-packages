# POS Mobile Packages

Central Maven registry for Stone's POS mobile libraries and artifacts. This repository manages the publication and distribution of POS-related packages, including the HAL API and other mobile dependencies.

**Build System:** Gradle  
**Primary Purpose:** Package management and distribution via GitHub Packages  
**Key Components:** HAL API (Android AAR), KMP publications, dependencies  
**Registry:** `https://maven.pkg.github.com/stone-payments/pos-mobile-packages`

---

## 📦 About This Repository

`pos-mobile-packages` serves as a **central hub for publishing and consuming** POS mobile artifacts:

- **HAL API Publication:** Publishes Android AAR packages from `pos-android-hal` releases
- **Dependency Management:** Hosts shared dependencies for POS projects
- **GitHub Packages Registry:** Provides Maven-compatible package distribution
- **CI/CD Workflows:** Automates the build and publication pipeline

### Key Workflow

The HAL API publication follows a **two-step process**:

1. **pos-android-hal** creates a release with tag `hal-api/x.y.z` (HAL API Release workflow)
2. **pos-mobile-packages** publishes the AAR to GitHub Packages (Publish HAL API workflow)

---

## 🚀 Using Packages from This Registry

### Add Repository

Add the Maven repository to your project's `build.gradle` (root level):

```groovy
repositories {
    maven {
        url = uri("https://maven.pkg.github.com/stone-payments/pos-mobile-packages")
        credentials {
            username = project.findProperty("gpr.user") ?: System.getenv("GITHUB_USERNAME")
            password = project.findProperty("gpr.key") ?: System.getenv("GITHUB_TOKEN")
        }
    }
}
```

### Add Dependencies

In your module's `build.gradle`, reference packages from this registry:

```groovy
dependencies {
    // HAL API
    implementation "co.stone.pos.mobile:hal-api:<version>"
    
    // Other POS packages (as published)
    implementation "co.stone.pos.mobile:other-package:<version>"
}
```

### Authentication

Ensure you have GitHub credentials configured:

- **GITHUB_USERNAME:** Your GitHub username or `x-access-token`
- **GITHUB_TOKEN:** A GitHub Personal Access Token (PAT) with `packages:read` permission

---

## 📋 Publishing Artifacts

### Publish HAL API

To publish a new version of HAL API:

1. Ensure a release was created in `pos-android-hal` with tag `hal-api/x.y.z`
2. Go to this repository's **Actions** tab
3. Select **"Publish HAL API to GitHub Packages"** workflow
4. Click **"Run workflow"**
5. Enter the tag name (e.g., `hal-api/3.6.6`)
6. Monitor the workflow until completion

For detailed instructions, see [`DEPLOY.md`](./DEPLOY.md).

### Manual Publication

For testing or alternative scenarios:

```bash
# Set version
echo "localPublishVersionName=x.y.z" >> local.properties

# Build and publish (from pos-android-hal or relevant module)
./gradlew :hal-api:publishAllPublicationsToGitHubPackagesRepository \
  -PGITHUB_PUBLISH_USERNAME=x-access-token \
  -PGITHUB_PUBLISH_TOKEN=$GITHUB_TOKEN
```

---

## 🔧 Repository Secrets

The following secrets must be configured at the **organization level** (`stone-payments`) and accessible to this repository:

| Secret | Purpose | Scope |
|--------|---------|-------|
| `GH_HARDWARE_MAMBA_SETUP_PRIVATE_KEY` | GitHub App token for cross-repo checkout | Org-level |
| `PACKAGECLOUD_READ_TOKEN` | PackageCloud read access for dependencies | Org-level |
| `PACKAGECLOUD_READ_TOKEN_INTERNAL` | PackageCloud internal read access | Org-level |
| `GITHUB_TOKEN` | Automatic GitHub token for publishing | Job-level (auto) |

---

## 📚 Documentation

- **[DEPLOY.md](./DEPLOY.md)** — Complete deployment and publication guide
- **[docs/](./docs/)** — Additional documentation and specifications

---

## 🏗️ Project Structure

```
pos-mobile-packages/
├── README.md                    # This file
├── DEPLOY.md                    # Deployment guide
├── .github/
│   └── workflows/
│       └── publish-hal-api.yml  # HAL API publication workflow
├── docs/
│   └── specs/                   # Technical specifications
├── Frameworks/                  # Framework integrations
└── catalog-info.yaml            # Backstage catalog metadata
```

---

## 🔗 Related Repositories

- **[pos-android-hal](https://github.com/stone-payments/pos-android-hal)** — HAL source and release management
- **[pos-mobile-app](https://github.com/stone-payments/pos-mobile-app)** — Consumer of hal-api packages (example)

---

## 📞 Support & Troubleshooting

If you encounter issues:

1. Check [`DEPLOY.md`](./DEPLOY.md) troubleshooting section
2. Verify secrets are configured in organization settings
3. Ensure your GitHub token has `packages:read` permission
4. Review the workflow logs in the **Actions** tab

For more help, contact the POS mobile team.

---

**Last Updated:** 2026-08-19
