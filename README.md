# GitHub Actions Release Workflow Samples 

Demonstrating GitHub Actions release workflows, with patterns for version bumping, tagging, and creating GitHub releases.

## Workflows

This repository demonstrates different approaches to release automation in GitHub Actions. Currently, it focuses on **automatic workflows** that trigger on release events without requiring manual action execution. Future additions will include **manual workflows** with `workflow_dispatch` inputs for creating releases on demand.

### Published Release 

**Trigger:** Published Release  
**File:** [.github/workflows/published-release.yml](.github/workflows/published-release.yml)

This workflow demonstrates using the GitHub Release UI as the primary interface for creating a release. The Release UI is the natural place to perform releases because it provides a structured interface that helps prevent common mistakes like forgetting to bump versions, deploy artifacts, attach release assets, or perform other critical release tasks. By triggering automation from the Release UI rather than manual workflow dispatch, you ensure releases follow a consistent, auditable process.

When a release is published in GitHub, this workflow:

1. Delete any existing release and tag
2. Check out the target branch with full Git history
3. Prepend a new changelog entry with release details
4. Commit the changelog update
5. Create an annotated tag pointing to the new commit
6. Recreate the GitHub release pointing to the new tag

**Notes:** 
- This workflow deletes and recreates releases, so it's incompatible with immutable releases. 
- Guards against workflow loops by checking `github.event.sender.type != 'Bot'` to prevent the workflow from triggering itself when it recreates the release.
- The assets attached to the original release are not preserved; additional steps would be needed to reattach them.

### Draft Release

**Trigger:** Draft Release Created _(hypothetical)_  
**File:** [.github/workflows/draft-release.yml](.github/workflows/draft-release.yml)

This workflow demonstrates using the GitHub Release UI as the primary interface for preparing a draft release.

Users would need to:
- Only create draft releases (never published releases directly)
- Wait for the workflow to complete before reviewing and publishing

When a draft release is created in GitHub, this workflow would:

1. Check out the target branch with full Git history
2. Prepend a new changelog entry with release details
3. Commit the changelog update
4. Create an annotated tag pointing to the new commit
5. Add build artifacts to the draft release
6. Update the draft release to point to the new tag

**Notes:** - 
- GitHub does not emit `release` events for draft creation; the `created` action only fires when a release is published. 
- This hypothetical workflow would be compatible with immutable releases since it doesn't need to delete releases. 
- The workflow could optionally be extended to auto-publish after successful completion

**Why this would be useful if supported:**
- Enables pre-publish validation and automation while keeping releases immutable until publication.
- Lets you attach and verify assets, notes, and metadata before making the release public.
- Aligns with the natural Release UI flow while reducing manual steps and mistakes.

**References:**
- GitHub Actions events — Release: https://docs.github.com/actions/using-workflows/events-that-trigger-workflows#release
- GitHub Community discussion: https://github.com/orgs/community/discussions/7118

### Manual Draft Release

**Trigger:** Manual `workflow_dispatch`
**File:** [.github/workflows/manual-draft-release.yml](.github/workflows/manual-draft-release.yml)

This workflow provides a practical alternative to the hypothetical draft-release trigger above. Because GitHub does not emit events when a release enters the draft state, a manual workflow dispatch is used to orchestrate the same automation while keeping the draft in place until you choose to publish.

When you run the workflow from the Actions tab you supply the draft release title (for example `v1.2.3`). The workflow then:

1. Uses the GitHub CLI to locate the matching draft release and validate that a tag name is configured
2. Checks out the target commitish and prepends a changelog entry (creating `CHANGELOG.md` if needed)
3. Commits the changelog update with the GitHub App credentials and pushes back to the default branch
4. Creates an annotated tag for the release, builds placeholder artifacts, and uploads them to the draft release
5. Re-points the draft release at the new tag so you can review and publish it when ready

**Notes:**
- The draft release title must exactly match the input supplied via `workflow_dispatch`
- Extend the build step (`Build artifacts`) to generate your real release assets before uploading
- Keeps the release in draft, allowing you to perform final checks prior to publication

## Shared Key Implementation Details

Applies to the Published and Draft workflows.

- **GitHub App tokens**: Uses app tokens for elevated permissions to bypass branch/tag protection rules when pushing commits and creating tags
- **Ecosystem integration**: Documents where build system tooling (Maven, npm, Gradle, etc.) would integrate for tasks like version bumping, artifact building, and deployment

## Setup

### 1. Create a GitHub App

The workflows use a GitHub App to obtain elevated permissions for bypassing branch/tag protections.

1. **Create the App** (Organization or Repository level):
   - Go to **Settings** → **Developer settings** → **GitHub Apps** → **New GitHub App**
   - Or for organizations: **Organization Settings** → **GitHub Apps** → **New GitHub App**

2. **Configure the App**:
   - **Name**: Choose a unique descriptive name (e.g., "Release Automation Bot")
   - **Homepage URL**: Your repository URL or organization URL
   - **Webhook**: Uncheck "Active" (not needed for this use case)
   
3. **Set Permissions** (Repository permissions):
   - **Contents**: Read and write (for pushing commits and tags)
   - **Metadata**: Read-only (automatically set)

4. **Generate a private key**:
   - Scroll to "Private keys" section → **Generate a private key**
   - Save the downloaded `.pem` file securely

5. **Note the App ID**:
   - Found at the top of the app settings page

6. **Install the App**:
   - Go to **Install App** (left sidebar)
   - Install on your organization or select specific repositories
   - Choose the repository where these workflows will run

### 2. Configure Repository Secrets and Variables

Add the following to your repository (or organization for shared access):

**Repository Settings** → **Secrets and variables** → **Actions**

**Secrets** (use the "New repository secret" button):
- `APP_PRIVATE_KEY`: Paste the entire contents of the `.pem` file (including `-----BEGIN RSA PRIVATE KEY-----` and `-----END RSA PRIVATE KEY-----`)

**Variables** (switch to "Variables" tab, use "New repository variable"):
- `APP_ID`: The numeric App ID from the app settings page

### 3. Verify Ruleset Bypass Permissions

If you have branch or tag rulesets enabled, ensure the GitHub App is allowed to bypass them:

1. **Check branch rulesets**:
   - Go to **Settings** → **Rules** → **Rulesets** (or **Organization Settings** → **Rules** → **Rulesets**)
   - Open any ruleset targeting `main` or your default branch
   - Under **Bypass list**, verify your GitHub App is included
   - If not, click **Add bypass** → **Apps** → select your App

2. **Check tag rulesets** (if protecting tags):
   - Open any ruleset targeting tags (e.g., pattern `refs/tags/*`)
   - Verify your GitHub App is in the **Bypass list**
   - If not, add it the same way

Without bypass permissions, the workflow will fail when trying to push commits or create tags on protected branches.

## Notes

This is a **demonstration repository**. For production use, consider:
- Adding GPG or SSH tag signing for verification
- Implementing additional validation and error handling
- Integrating with your build system's release tooling
- Rotating the App private key periodically
