# FAQ Governance and Technical Implementation Guide

This document defines the architectural guidelines, content design principles, and automated verification workflows required to manage Frequently Asked Questions (FAQ) documentation. These policies ensure a high-quality user experience, rigorous data governance, clear separation of duties, and reliable audit trails.

!!! tip "See also"
    For the human PR review workflow and guidance on handling out-of-scope content,
    see [FAQ PR Review Guide](faq-reviews.md).

## 📖 Concept: The FAQ Governance Philosophy

!!! info
    In enterprise documentation management, FAQs present a unique challenge. While highly valuable for rapid self-service, unstructured FAQs quickly degrade into unmaintainable content dumps that obscure critical answers and contradict official product documentation.

### Why FAQ Governance Matters

To maintain a trusted single source of truth, FAQ management must satisfy four core pillars:

1. **User Experience:** FAQ content must be concise, predictable, and rapidly scannable.
2. **Governance & Management:** Topics must be structured within designated page boundaries to prevent sprawl.
3. **Separation of Duties:** Content writers propose changes; subject matter experts (SMEs) and docs architects certify structural and technical accuracy.
4. **Auditability:** Every FAQ page must maintain explicit metadata trace paths (such as `lastCertified` records) to verify factual relevancy over time.

### Topic Classification Model

All technical documentation must adhere to strict topic-based classification. FAQ pages are designated as **Reference** topics. They must point directly to, rather than duplicate, detailed procedural or conceptual pages:

```text
[Scannable QA Pair (Reference)]
              │
              └──► (Link to) ──► [Deep-Dive Guide (Task/Concept)]
```

- **Concepts:** Explain *how* and *why* a system behaves a certain way.
- **Tasks / How-Tos:** Provide step-by-step instructions to achieve a specific goal.
- **Troubleshooting:** Diagnose errors, describe failure modes, and provide mitigation paths.
- **FAQs (Reference):** Provide rapid, direct answers (capped at 2–3 sentences) to highly isolated questions. If an FAQ answer must explain a concept or detail a procedure, **it is misclassified** and must be extracted to its own dedicated page.

## 🛠️ How-To: Check Enterprise and Org-Level Settings for Actions/App Support

Before implementing automated enforcement, you must identify whether your GitHub Enterprise Cloud (GHEC) organization permits repository-level CI/CD automation. In GHEC, policies cascade down from the **Enterprise** to the **Organization**, and finally to the **Repository**. Policies enforced at the Enterprise level cannot be bypassed at the Org or Repo levels.

```text
┌────────────────────────────────────────────────────────┐
│                   GHEC Policy Cascade                  │
│                                                        │
│  Enterprise Policies  ──► Sets maximum allowed limits  │
│          │                                             │
│          ▼                                             │
│  Organization Rules   ──► Can restrict further         │
│          │                                             │
│          ▼                                             │
│  Repository Settings  ──► Local configurations         │
└────────────────────────────────────────────────────────┘
```

You can determine your permission status using the **GitHub Web UI** or directly from the **Terminal** via the GitHub CLI or API.

### Option A: Web UI Procedure

#### Step 1: Detect Enterprise-Enforced Policies

!!! warning
    If a policy is enforced by the Enterprise, the setting toggles in your Organization and Repository will be grayed out, accompanied by a shield icon and the text: *"This setting is managed by your enterprise."*

1. Navigate to your Organization's main page on GitHub.
2. Click **Settings** in the top navigation bar of the organization.
3. In the left sidebar, click **Actions** > **General**.
4. Check for enterprise lock banners. If Actions are disabled here by the Enterprise, **Method 1** and **Method 3** are completely unavailable. You must use **Method 2 (Alternative A)** or **Method 3 (Alternative B)**.

#### Step 2: Verify GitHub Actions Availability (Repo Level)

If the Enterprise permits organization-level control, verify your local repository settings:

1. Navigate to your repository's main page on GitHub.
2. Click **Settings** in the top navigation bar. *(If you do not see "Settings", you lack repository Administrator/Maintainer permissions.)*
3. In the left sidebar, click the **Actions** dropdown, then select **General**.
4. Review the **Policies** section.

#### Step 3: Inspect Workflow Write Permissions

For automation scripts to write directly to a PR description or alter metadata, the default runner environment token (`GITHUB_TOKEN`) must possess explicit write permissions:

1. Scroll down the **Actions > General** settings page to **Workflow permissions**.
2. Verify that **Read and write permissions** is selected.
3. Ensure that the checkbox **Allow GitHub Actions to create and approve pull requests** is checked.

### Option B: Terminal / Command Line Procedure

If you are managing automation at scale, you can query these GHEC policies programmatically using the GitHub CLI (`gh`) or `curl` with the GitHub REST API.

#### Step 1: Check Organization-Level Actions Permissions

Run this command to check if GitHub Actions are enabled globally for your organization and see if they are restricted.

=== "GitHub CLI"
    ```bash
    gh api orgs/{org_name}/actions/permissions
    ```

=== "cURL"
    ```bash
    curl -H "Authorization: Bearer $GITHUB_TOKEN" \
         -H "Accept: application/vnd.github+json" \
         https://api.github.com/orgs/{org_name}/actions/permissions
    ```

**What to look for in the JSON response:**

- `enabled_repositories`: If this is `"none"`, Actions are completely disabled.
- `allowed_actions`: Determines if you can run third-party actions (`all`, `local_only`, or `selected`).

#### Step 2: Check Repository-Level Actions Permissions

Check if Actions are enabled on your specific repository.

=== "GitHub CLI"
    ```bash
    gh api repos/{owner}/{repo}/actions/permissions
    ```

=== "cURL"
    ```bash
    curl -H "Authorization: Bearer $GITHUB_TOKEN" \
         -H "Accept: application/vnd.github+json" \
         https://api.github.com/repos/{owner}/{repo}/actions/permissions
    ```

**What to look for in the JSON response:**

- `enabled`: Must be `true` for workflows to run.

#### Step 3: Check Default Workflow Permissions (Token Write Scopes)

Check if workflows have read/write access and if they can approve pull requests by default.

=== "GitHub CLI"
    ```bash
    gh api repos/{owner}/{repo}/actions/permissions/workflow
    ```

=== "cURL"
    ```bash
    curl -H "Authorization: Bearer $GITHUB_TOKEN" \
         -H "Accept: application/vnd.github+json" \
         https://api.github.com/repos/{owner}/{repo}/actions/permissions/workflow
    ```

**What to look for in the JSON response:**

```json
{
  "default_workflow_permissions": "write",
  "can_approve_pull_request_reviews": true
}
```

!!! warning
    If `default_workflow_permissions` returns `"read"`, your bot workflows **cannot** append templates or perform automated remediation without a custom personal access token (PAT).

## 🛠️ How-To: Implement FAQ Validation Checks

When changes are proposed to FAQ documentation, the PR review pipeline must enforce structure, style, and metadata updates. Depending on your repository's technical constraints, choose **one** of the three methods below.

```text
              Enforcement Method Selection
              ───────────────────────────
                           │
       Is `.github/workflows/` available & writeable?
                   ┌───────────┴───────────┐
                   ▼                       ▼
                [ YES ]                 [ NO ]
                   │                       │
    Are you enforcing strictly?     Have Org-Level Admin?
       ┌───────────┴───────────┐       ┌───────┴───────┐
       ▼                       ▼       ▼               ▼
   [Method 1]             [Method 3] [Method A]   [Method B]
(Auto-Append Bot)     (Status Blocker) (Org Repo) (Query URL)
```

### Method 1: The Auto-Append Bot (Preferred)

!!! tip
    Use this method if GitHub Actions are enabled and the workflow path and permissions are writeable. This script automatically appends the FAQ validation checklist to any PR altering an FAQ markdown file.

Create a workflow file at `.github/workflows/faq-governance.yml`:

```yaml
name: FAQ Governance & Audit Enforcer

on:
  pull_request:
    types: [opened, edited]
    paths:
      - '**/*FAQ*.md'
      - '**/*faq*.md'

jobs:
  inject-faq-audit-checklist:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - name: Check out repository
        uses: actions/checkout@v4

      - name: Append Governance Checklist
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_URL: ${{ github.event.pull_request.html_url }}
          PR_BODY: ${{ github.event.pull_request.body }}
        run: |
          MARKER="<!-- FAQ_GOVERNANCE_CHECKLIST_MARKER -->"

          # Idempotency check: Don't append if it's already there
          if echo "$PR_BODY" | grep -q "$MARKER"; then
            echo "Governance checklist already present. Skipping."
            exit 0
          fi

          GOVERNANCE_BLOCK=$(cat << 'EOF'

          <!-- FAQ_GOVERNANCE_CHECKLIST_MARKER -->
          ---
          ## 📋 Required FAQ Governance Audit
          *This PR alters FAQ content. Reviewers and contributors must verify the following criteria before merging.*

          ### 🏗️ FAQ Organization & Structure
          - [ ] **Topic Alignment:** All QA pairs belong strictly within this specific page topic.
          - [ ] **Density Check:** This page does **not** exceed 10–15 QA pairs.
          - [ ] **Scope Control:** Answers do not introduce long concepts, troubleshooting guides, or complex procedures.
          - [ ] **Isolation:** No QA pairs are placed outside designated FAQ documents.

          ### ✍️ Content Quality & Grammar
          - [ ] **Grammar Check:** Every "question" is structured as an actual grammatical question.
          - [ ] **Conciseness:** Answers are brief and do not exceed 2 to 3 sentences.
          - [ ] **Hyperlink Integrity:** Answers referring to external/internal resources include direct links.
          - [ ] **Deduplication:** This QA pair does not duplicate content found in other sections or pages.
          - [ ] **Consistency:** The answers do not contradict other live pages or established FAQ topics.

          ### 🔑 Metadata & Navigation
          - [ ] **Audit Trail:** The `lastCertified` front-matter metadata tag has been updated to reflect this review.
          - [ ] **Navigation Sync:** If content was moved or a new page was recommended, `mkdocs.yml` has been updated.
          EOF
          )

          NEW_BODY="$PR_BODY"$'\n'"$GOVERNANCE_BLOCK"
          echo "$NEW_BODY" | gh pr edit "$PR_URL" --body-file -
```

### Method 2: Manual Organization-Level Selection (Alternative A)

!!! info
    Use this method if the local repository's `.github/workflows` path is blocked, but you have Org-level Administrator access.

**Terminal Execution:**

1. **Create the global `.github` repository** under your organization:
   ```bash
   gh repo create {org_name}/.github --public --description "Global organization templates"
   ```

2. **Clone the repository locally:**
   ```bash
   git clone https://github.com/{org_name}/.github.git
   cd .github
   ```

3. **Establish the PR template directory structure:**
   ```bash
   mkdir -p .github/PULL_REQUEST_TEMPLATE
   ```

4. **Write the governance template** to a template markdown file:
   ```bash
   cat << 'EOF' > .github/PULL_REQUEST_TEMPLATE/faq_markdown_template.md
   ## FAQ Governance Validation Required
   - [ ] Verified FAQ answers are under 3 sentences.
   - [ ] No procedural instructions or troubleshooting details are included.
   - [ ] `lastCertified` metadata in front matter has been updated.
   EOF
   ```

5. **Commit and push** the changes to the default branch:
   ```bash
   git add .
   git commit -m "add global faq pr template"
   git push origin main
   ```

### Method 3: The Query Parameter URL Link (Alternative B)

!!! info
    Use this method if both local repository workflows and organization-level repository creation are locked.

GitHub natively parses query parameters passed via the compare URL. You can bundle your checklist into a pre-formatted link or browser bookmarklet for your writing team.

Construct the URL template as follows:

```text
https://github.com/OWNER/REPO/compare/main...BRANCH_NAME?title=[FAQ]+Topic+Update&body=%23%23+FAQ+Governance+Audit%0A-%20%5B%20%5D+Density%3A+Does+not+exceed+10-15+pairs%0A-%20%5B%20%5D+Metadata%3A+lastCertified+is+updated%0A-%20%5B%20%5D+Navigation%3A+mkdocs.yml+updated
```

When clicked, this automatically populates the PR creation canvas with the required validation checkboxes.

## 🔍 Troubleshooting: Common Workflow Failures

### Issue 1: GitHub CLI fails with "Resource not accessible by integration"

- **Symptoms:** The Action runs, but terminates at the `gh pr edit` step with an HTTP 403 error.
- **Root Cause:** The default `GITHUB_TOKEN` is operating in read-only mode, or the workflow file does not explicitly declare its permission scopes.
- **Mitigation:** Ensure your workflow file contains the explicit permissions block directly beneath the job header:

  ```yaml
  jobs:
    append-validation-list:
      runs-on: ubuntu-latest
      permissions:
        pull-requests: write
  ```

### Issue 2: Workflow is not triggering on FAQ file changes

- **Symptoms:** An FAQ Markdown file is pushed, but the Action does not run.
- **Root Cause:** Path mapping matches are case-sensitive or directory boundaries are failing to capture nested structures.
- **Mitigation:** Always use double asterisks (`**`) to traverse deep nested directories, and explicitly account for case variations in filenames:

  ```yaml
  paths:
    - '**/*FAQ*.md'
    - '**/*faq*.md'
    - '**/*Faq*.md'
  ```

### Issue 3: Workflow fails on Enterprise-enforced read-only tokens

!!! warning
    Enterprise-enforced read-only token scopes cannot be overridden at the repository level.

- **Symptoms:** You have configured `permissions: write` in your workflow, but the job still fails with write permission errors.
- **Root Cause:** The parent enterprise has a global security policy enforcing read-only `GITHUB_TOKEN` scopes across all child orgs/repositories, blocking local job overrides.
- **Mitigation:** You must use **Method 2 (Alternative A)** or **Method 3 (Alternative B)**, or contact your enterprise GHEC administrator to request an exemption or a dedicated GitHub App credentials setup.

## 📋 Reference: Reviewer Verification Checklists

These checklists represent the core criteria for FAQ reviews and must be validated during human peer-review stages before merging any FAQ pull request. For the full step-by-step reviewer workflow, see the [FAQ PR Review Guide](faq-reviews.md).

### FAQ Structural & Organization Rules

- **Topic Boundary:** All question/answer pairs must reside on the specific FAQ page mapped to that system. Do not allow general answers to drift into mismatched technical areas.
- **Density Boundary:** FAQ pages must not exceed 10–15 question/answer pairs. If a topic grows beyond this count, the reviewer must request that the page be split into logical sub-topics.
- **Extraction Rule:** Answers are strictly limited to **two or three sentences**. If an answer introduces conceptual theory, troubleshooting pathways, or step-by-step procedure, the reviewer **must** reject the PR and instruct the contributor to extract that content into a separate Conceptual, Troubleshooting, or Task page.
- **Deduplication:** A question/answer pair must not exist on any other page or section.

### Metadata Compliance Rules

- **Metadata Stamp:** The page's front-matter metadata configuration must include a `lastCertified` stamp. This date must be manually updated to the current date on any PR modifying the page's contents.

  ```yaml
  ---
  title: Integration FAQ
  lastCertified: 2026-06-17
  ---
  ```

- **Navigation Alignment:** If content extraction yields a new documentation page, the PR must include corresponding updates to the `mkdocs.yml` outline configuration file to ensure navigation rendering remains unbroken.
