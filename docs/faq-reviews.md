# FAQ Documentation and Contribution Management

!!! tip "See also"
    For automated enforcement, GitHub Actions workflows, and GHEC permission setup,
    see [FAQ Governance & Automation](faq-governance.md).

This guide establishes the standards and workflows for maintaining high-quality FAQ (Frequently Asked Questions) sections. By enforcing strict separation of concerns, we preserve the integrity of our user experience, ease governance, and ensure clear audit trails.

## FAQ Architecture and Governance Principles

!!! info
    FAQs are highly visible entry points for users, but they easily degrade into "dumping grounds" for content that belongs elsewhere. Managing FAQ contributions actively at the Pull Request (PR) stage secures four key operational pillars.

1. **User Experience (UX):** Users expect FAQs to provide rapid, punchy, and direct answers to highly specific questions. When FAQs are clean, users resolve doubts without searching through extensive manuals.
2. **Governance and Lifecycle Management:** Small, discrete question-and-answer (Q&A) pairs are easier to update, deprecate, and track for accuracy over time.
3. **Separation of Duties:** Enforcing topic types ensures technical writers, product managers, and engineers own and verify the specific content matching their expertise.
4. **Audit and Compliance:** Clean metadata tracking (such as certification dates) ensures documentation meets regulatory or organizational compliance standards.

### The Limits of an FAQ

To maintain readability and cognitive ease, an FAQ topic must respect strict boundaries:

- **Quantities:** An individual FAQ page should contain no more than 10–15 Q&A pairs.
- **Length:** Answers must not exceed two to three sentences.
- **Type Separation:** FAQ answers must never introduce new concepts, troubleshoot complex errors, or outline multi-step procedures. These belong in dedicated **Concept**, **Troubleshooting**, or **How-to** pages respectively.

## Review an FAQ Pull Request

Follow these steps to systematically review incoming Pull Requests (PRs) that propose additions or modifications to FAQ pages.

### Prerequisites

- Access to the project's documentation repository and pull request interface.
- Familiarity with the documentation navigation structure (`mkdocs.yml` or equivalent).

### Step 1: Verify FAQ Page Organization

Ensure the proposed Q&A pairs are located on the correct page and do not exceed structural limits.

1. Check that the Q&A pairs reside within an explicit FAQ document/section.
2. Count the total Q&As on the target page. If the new additions push the total count beyond **10–15 pairs**, flag the PR and request that the page be split.
3. Verify that the Q&As are grouped logically under appropriate subheadings.

### Step 2: Evaluate Individual Q&A Pairs

Audit the format and content of each proposed pair:

1. **The Question:** Verify that the "question" is phrased as an actual, grammatical question ending with a question mark.
2. **The Answer:** Ensure the answer does not exceed **two to three sentences**.
3. **References:** Check all mentioned external resources or internal pages. Ensure they contain live, functional inline hyperlinks. Do not allow plain-text references to other sites.
4. **Uniqueness:** Search the existing documentation to verify that the proposed Q&A pair does not duplicate an existing entry.
5. **Accuracy:** Cross-reference the answer with core product behavior to ensure it does not contradict other documentation pages.

### Step 3: Validate Page Metadata

Ensure tracking metadata is updated correctly to keep the document certified.

1. Locate the front-matter metadata at the top of the modified markdown file.
2. Verify that the `lastCertified` date (or equivalent review timestamp) has been updated to the current date.

### Step 4: Approve or Request Changes

- If all checks pass, approve the PR.
- If any check fails, leave specific, actionable comments on the PR citing the rules in this guide.

## Resolving Out-of-Scope and Bloated FAQ Content

When reviewing FAQ contributions, you will frequently encounter submissions that violate standard FAQ constraints (e.g., answers that are too long, contain procedural steps, or explain complex concepts). Use this guide to help contributors restructure and relocate their content.

### Problem: The FAQ Answer Contains "How-To" Procedures

!!! warning "Problem: How-To Content in an FAQ Answer"
    **Symptoms:** The answer details step-by-step instructions (e.g., "First, click X, then configure Y...") or exceeds three sentences.

**Resolution:**

1. Instruct the contributor to extract the step-by-step procedure into a dedicated **How-to** page.
2. Have them replace the bloated FAQ answer with a brief, high-level summary (1–2 sentences) followed by a direct hyperlink to the new How-to page.

*Example:* "You can configure custom ports during installation. For a complete guide, see [How to Configure Custom Ports](http://docs.google.com/how-to/ports.md)."

### Problem: The FAQ Answer Explains a Foundational "Concept"

!!! warning "Problem: Conceptual Content in an FAQ Answer"
    **Symptoms:** The answer defines technical architecture, explains design philosophies, or provides deep background information.

**Resolution:**

1. Direct the contributor to move the explanatory prose to a **Concept** page.
2. Keep the FAQ question focused, and rewrite the answer to point to the concept page.

*Example:* "Yes, our platform supports clustering. To understand the underlying cluster architecture and node communication, read our [Clustering Concepts guide](http://docs.google.com/concepts/clustering.md)."

### Problem: No Existing Page Fits the Relocated Content

!!! warning "Problem: No Navigation Home for Relocated Content"
    **Symptoms:** You need to reject/relocate FAQ content, but there is no logical home for the new Topic Type in the current navigation.

**Resolution:** Instruct the contributor to create a new page and provide them with the following guidance in your PR review:

1. **Recommend a Category:** Suggest a new or existing navigation section (e.g., `/concepts/` or `/how-to/`).
2. **Propose a Scalable Title:** Suggest a page title with a broad enough scope to accommodate future growth (e.g., use *Authentication Methods* instead of *How to Log In with Okta*).
3. **Assign the Correct Topic Type:** Expressly declare whether the new page must be structured as a *Concept*, *How-to*, or *Troubleshooting* topic.
4. **Provide Navigation Instructions:** Instruct the contributor to update the navigation configuration (e.g., `mkdocs.yml`) to register the new page under the proposed section.

### Problem: The FAQ Section is Too Long (Over 15 Q&As)

!!! warning "Problem: FAQ Page Density Exceeded"
    **Symptoms:** A single FAQ page has become difficult to read or navigate due to sheer volume.

**Resolution:**

1. Request that the contributor split the FAQ page into smaller, highly specialized FAQ sub-topics.
2. Provide clear suggestions for splitting (e.g., splitting a general *Billing FAQ* into *Invoicing FAQ* and *Subscription Plans FAQ*).
