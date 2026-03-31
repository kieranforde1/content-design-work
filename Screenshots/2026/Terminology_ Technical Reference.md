# Summary

# Terminology: Technical Reference

**Owner:** Kieran Forde, Central Content Design 

**Related:** [Terminology: 2026 Content Design Vision](https://docs.google.com/document/u/0/d/1JCxwCp2xIHNivpBJLZeu14EvQ7vfx3USIjNWYVn7HjU/edit), [Technical Requirements: Content Design Tooling](https://docs.google.com/document/u/0/d/1vBvmB2rb-GvBsz4yFfhlQS2kqdbB7QzT7JSN0V2dnXg/edit)

This document contains the technical research, architecture decisions, and implementation detail behind the terminology infrastructure described in the Vision doc. It's intended as a working reference for anyone who needs to understand how the system is built, what's been explored, and what's been parked.

**Linguine** — A proof of concept validating that terminology issues identified by the checker app can be fixed through Linguine's UI, which auto-generates GitHub PRs. This confirms Linguine as a viable fix mechanism for clear-cut terminology errors.

**Storage** — Documents the current data architecture, including the schema design, Supabase setup, and decisions around how scan results are stored and queried.

**Monarch** — Explores how terminology quality metrics could integrate with Monarch's service maturity framework, including a proposed maturity dimension and pilot approach.

**Accessibility** — Early-stage exploration of how the checker app could surface accessibility-related content issues. Currently parked pending progress on core terminology capabilities.

For the strategic context — why this work matters, what we're solving, and where we're headed — see the Vision doc.

# Storage

# Storage Architecture Summary

## The Two-Layer Storage Model

This app uses **two storage layers** working together:

### Layer 1: Browser Storage (IndexedDB)

* **Purpose**: Fast, local cache during scanning  
* **Location**: Lives entirely in the user's browser  
* **Behavior**: Results are written here first during a scan because it's instant and doesn't require network calls  
* **Risk**: If the browser tab closes, or storage is blocked, this data is lost

### Layer 2: Cloud Database (Postgres via Supabase)

* **Purpose**: Permanent, shareable storage  
* **Location**: Remote server  
* **Behavior**: A background service gradually pushes IndexedDB data to the cloud in batches  
* **Benefit**: Data survives browser sessions and can be accessed from anywhere

## How Data Flows During a Scan

```
Files uploaded
    ↓
Detection engine finds issues
    ↓
Issues cached to IndexedDB (instant)
    ↓
BackgroundSyncService pushes batches to cloud (every 3 seconds)
    ↓
When scan finishes, final "flush" pushes all remaining items
    ↓
Verification step confirms cloud has the data
    ↓
Scan marked "completed" only if verified
```

## Critical Lessons Learned

### 1\. Never Trust "Completed" Without Verification

**Problem solved**: The app was marking scans as "completed" based on how many issues were *detected*, not how many were *actually saved* to the database.

**Solution**: Before marking complete,  app now runs a `COUNT(*)` query against the database to verify the issues actually exist. If the count is zero but detected issues, the scan is marked `persistence_failed` instead.

### 2\. Preflight Health Checks Are Essential

**Problem solved**: Users could start a multi-hour scan without knowing that storage was broken (IndexedDB blocked, authentication expired, etc.).

**Solution**: Before any scan starts, app now:

* Tests that IndexedDB can write/read/delete a record  
* Tests that the cloud database can accept a test insert  
* Block the scan with a clear error message if either fails

### 3\. Never Delete the Only Copy of Data

**Problem solved**: The sync service was clearing the local cache even when the cloud sync failed, destroying the only copy of results.

**Solution**: The cache is now only cleared when:

* The flush operation reports success  
* The pending count is confirmed to be zero  
* If sync fails, the cache is preserved for retry

### 4\. Timeouts Must Match Reality

**Problem solved**: The finalization step had a 70-second timeout, but large scans need several minutes to push hundreds of thousands of records.

**Solution**: Removed the artificial timeout. The flush now runs to completion (up to 10 minutes for very large scans).

# Database Schema (Key Tables)

`terminology_scans`

* Stores scan metadata: name, status, timestamps, file counts  
* Status values: `in_progress`, `completed`, `partial`, `persistence_failed`  
* The `total_issues_found` field should now reflect the *actual persisted count*

`terminology_issues`

* One row per flagged term  
* Links to scan via `scan_id` foreign key  
* Has unique constraints to prevent duplicates

`writer_terms`

* The glossary of terms to detect  
* Loaded from Supabase on app start, cached locally for scan performance

# Security Model (RLS)

Row Level Security controls who can do what:

| Role | Read | Write | Delete |
| ----- | :---: | :---: | :---: |
| Admin | ✓ | ✓ | ✓ |
| Authenticated | ✓ | ✗ | ✗ |
| Anonymous | ✗ | ✗ | ✗ |

Only `@hubspot.com` emails can sign up (enforced by edge function validation). 

Admin role must be manually assigned in the `user_roles` table.

# 

# Key Services to Understand

| Service | Responsibility |
| ----- | ----- |
| `ScanCacheService` | Manages IndexedDB \- caching, reading, clearing |
| `BackgroundSyncService` | Pushes IndexedDB data to cloud in batches |
| `ScanReportService` | Creates/updates scan records, saves issues to database |
| `useTwoPassFileProcessor` | Orchestrates the entire scan lifecycle |

# 

# If Rebuilding on Internal Infrastructure

Things to replicate:

1. **Local cache layer** \- Some equivalent of IndexedDB for buffering during processing  
2. **Background sync pattern** \- Don't block the main processing loop on network calls  
3. **Two-phase commit** \- Verify persistence before marking complete  
4. **Preflight checks** \- Test storage is working before starting long operations  
5. **Observable state** \- Show pending vs. persisted counts so users know if it's working  
6. **Unique constraints** \- Prevent duplicate issues with database-level enforcement  
7. **Role-based access** \- Separate read-only users from admins who can modify data

# Future Storage

## Elasticsearch database

Alia Curtis is building an Elasticsearch database of all Linguine product strings. It will be updated in real-time with new and changed English strings and translations. 

If we used that index as an input source for the terminology checker app, it could replace the current manual download/upload flow and instead let the app pull strings directly from Elasticsearch on demand, giving us broader, fresher coverage of UI copy with less overhead. 

From a storage perspective, Elasticsearch (fed by Linguine’s primary stores) would become the system of record for raw strings, while the checker’s own two‑layer storage—local browser cache plus Supabase/Postgres—for scans and terminology issues would stay focused on results and audit history rather than holding bulk string content, keeping with HubSpot’s pattern of using Elasticsearch as a secondary, query‑optimized store on top of a more durable primary datastore.

# Monarch

# How to Automatically Create Monarch Tasks 

This guide walks through the complete process of automatically creating **Monarch tasks**, which are used to bulk‑create work items for teams at HubSpot.

## **What is Monarch?**

Monarch is HubSpot’s system for bulk creation of **Mainsail Issues**—work items assigned to teams for infrastructure migrations, reliability concerns, and other systematic changes.

## **The 4‑Step Process to Create Monarch Tasks Automatically**

### **Step 1: Create a Campaign** 

1. Go to go/monarch or [https://private.hubteam.com/monarch](https://private.hubteam.com/monarch)  
2. Click **Create Campaign**  
3. Fill out the **Info** section:  
* **Description:** A detailed description that someone unfamiliar with the technology can understand  
* **Context:** Why this work is needed  
* **Instructions:** Explicit, step‑by‑step guidance—include exact commands, how to interpret output, and what to do with results  
* **Resources:** Links to documentation, queries, or tools that support the work

**Pro tip:** Test the description with 2–3 teams of different types to confirm they can understand it without extra context.

### **Step 2: Choose or Create a Work Producer** 

**Work Producers** are the automated workers that generate work items.

There are two main options:

#### **Option A: Use an Existing Producer (Recommended)**

1. In the campaign, select **Edit producer details**  
2. Browse the dropdown of existing producers  
3. Many common use cases are already covered

#### **Option B: Create a Custom Producer**

If no existing producer fits the use case:

**1\. Generate the producer code:**

```shell
# Run java-gen and select "monarch work producer"
java-gen
```

**2\. Create a new module:**

* Name: `your-repo-monarch-producers`  
* Visibility: `OWNING_TEAM` initially (for testing)

**3\. Implement the producer:**

```java
@Override
public Set<ProducerWorkItem> getWorkItems(YourCustomProducerQuery query) {
  // Logic to find work items
  List<Issue> foundIssues = issueService.findIssues(query.getCriteria());

  return foundIssues.stream()
    .map(issue -> ProducerWorkItem.builder()
      .setKey(issue.getUniqueIdentifier())      // Unique stable ID
      .setTeams(issue.getAssignedTeams())
      .setUrgency(issue.getUrgency())
      .setTitle(issue.getWorkItemTitle())
      .setDetailsMd(formatIssueDetails(issue))
      .setCategory(issue.getGroup())
      .setWorkUnits(calculateWorkUnits(issue))
      .build())
    .collect(toImmutableSet());
}
```

**4\. Configure the producer:**

```java
WorkProducerRunner
  .newBuilder("Your Team")
  .register(
    "your-custom-producer",
    WorkProducerDescription
      .newBuilder(YourCustomProducer.class)
      .setName("Your Custom Producer Name")
      .setQueryClass(YourCustomProducerQuery.class)
      .setVisibility(ProducerVisibility.OWNING_TEAM) // Start with OWNING_TEAM
      .setDocumentationMdResourcePath("your-custom-producer.md")
      .setRefreshable(true)   // Allow on-demand refresh
      .setReassignable(true)  // Allow teams to reassign
      .build()
  );
```

**5\. Test locally:**

```shell
# Run with dry-run to see what work items would be generated
# Add to deployment YAML:
--dryRun --query "{}"
```

**6\. Deploy to QA:**

* Remove the `--dryRun` flag  
* Deploy to QA  
* Verify logs show something like:  
   `[Tq2PullWorkerService] INFO c.h.m.p.base.WorkProducerPullWorker - Registered work producer - ...`

**7\. Set to GLOBAL (optional):**

* Once tested and stable, change visibility to `ProducerVisibility.GLOBAL` if the producer should be reusable by other teams.[^3](https://product.hubteam.com/docs/mainsail/monarch/custom-producer.html)

### **Step 3: Estimate and Schedule the Work** 

**Estimate work units:**

* Define how much effort each work item requires  
* Standard conversion: **3 units ≈ 1 day** for a senior engineer familiar with the codebase  
* Estimates should be optimistic but realistic

**Create a work schedule:**

* Set due dates for the work  
* Choose urgency levels where appropriate (**LOW, MEDIUM, HIGH, CRITICAL**)  
* Work schedules control when work is assigned and when it is due[^4](https://product.hubteam.com/docs/mainsail/monarch/work-units.html)

---

### **Step 4: Get Approval and Launch :white\_tick:**

**Approval process:**

1. **Submit a Mainsail proposal** in Monarch

   * Provide the rationale for the campaign  
   * Select 2–3 reviewers:  
     * Domain expert(s) for the campaign  
     * TL(s) of impacted teams  
2. **Automatic PR creation**

   * Monarch creates a PR for the selected reviewers  
3. **Reviewer approval**

   * Domain experts and TLs review and approve  
4. **Leadership final check**

   * Leadership performs final approval  
5. **Campaign goes live**

   * Work items are automatically created and assigned to teams[^5](https://product.hubteam.com/docs/mainsail/monarch/approval-process.html)

---

## **Key Concepts to Understand**

### **Work item properties**

| Property | Description | Importance |
| ----- | ----- | ----- |
| Key | Unique, stable identifier | Critical for tracking changes across refreshes |
| Title | Human‑readable title | Shown in the UI |
| DetailsMd | Detailed description in Markdown | Explains what needs to be done |
| Teams | List of Janus teams assigned | Indicates who is responsible |
| Category | Grouping for UI display (e.g. repo/service) | Helps with browsing and reporting |
| Work Units | Effort estimate (3 units ≈ 1 day) | Helps teams prioritize and plan |
| Urgency | LOW / MEDIUM / HIGH / CRITICAL | Maps to scheduling and prioritization |

---

## **Best Practices**

### **✅ Test first**

* Use draft campaigns for early experiments  
* Run producers locally with `--dryRun`  
* No work is assigned until the campaign is approved

### **✅ Write strong documentation**

* Clearly explain why the work is needed  
* Provide exact commands and steps  
* Link to supporting resources and queries  
* Embed example output or results where helpful

### **✅ Make keys stable**

* Avoid using line numbers as keys (they change frequently)  
* Avoid using raw file paths when files may move  
* Prefer stable identifiers like method names, class names, logical IDs, or composite keys  
* If instability is unavoidable, consider `setMatchUnstableWork(true)` where appropriate

### **✅ Enable reassignment**

* In most cases, keep `setReassignable(true)`  
* This lets teams route work to the correct team when ownership is unclear  
* Janus hierarchies can be complex; reassignment is often necessary

---

## **Common Mistakes to Avoid :warning:**

### **❌ Unstable keys**

* Keys based on line numbers or brittle file paths cause work items to reset when code changes  
* Fix: use stable identifiers or explicitly configure matching for unstable work

### **❌ Vague instructions**

* Example anti-pattern: “Check the dependency tree”  
* Better: “Run `mvn dependency:tree | grep <pattern>`, look for lines matching `<pattern>`, and update to `<version>`”

### **❌ No testing**

* Launching without `--dryRun` or QA validation can create thousands of incorrect tasks  
* Always test locally and in QA before requesting approval

### **❌ Disallowing reassignment**

* Setting `setReassignable(false)` prevents teams from correcting mistakes in assignment  
* Use only when absolutely certain that routing is always correct

---

## **Getting Help :sos:**

* **\#migrations-tooling-support** – Questions about producers, campaigns, and Monarch patterns  
* **\#mothership** – Help automating the underlying code changes  
* **Monarch UI:** [https://private.hubteam.com/monarch](https://private.hubteam.com/monarch)  
* **Documentation:** [https://product.hubteam.com/docs/mainsail/monarch/](https://product.hubteam.com/docs/mainsail/monarch/)

---

## **Quick Reference: Creating a First Campaign**

1. Go to **go/monarch** and create a campaign  
2. Write a detailed **Info** section with context and instructions  
3. Choose an existing producer or implement a custom one  
4. Test the producer locally with `--dryRun`  
5. Deploy to QA and verify behaviour  
6. Estimate work units (3 units ≈ 1 day)  
7. Create a work schedule with due dates and urgency  
8. Submit a proposal for approval  
9. Get 2–3 reviewers (domain expert \+ TLs)  
10. After leadership approval, let the campaign go live and automatically generate tasks

# Linguine

# Linguine

## **1\. What Linguine is**

Linguine is HubSpot’s internal **localization and string management UI**. For the purposes of the terminology project, it’s effectively:

* A **UI for viewing and editing English UI strings** that live in translation files (e.g. `en.lyaml`).  
* A bridge between **non‑engineers** (designers, content designers) and the codebase:  
  * I can edit copy in Linguine’s web UI.  
  * Linguine then opens a **GitHub PR** with the updated strings, so engineers can review and merge.

It’s already used in the org as a way for designers to make **small copy updates** without touching code, via the “View and edit English strings” view on a project resource.

For terminology, that makes Linguine a good manual path when the string lives in a translation file.

## **2\. Testing making changes in Linguine**

### **2.1 Browse and filter English strings**

Using the [Linguine UI, I navigated to](https://tools.hubteam.com/linguine-ui/dashboard/copyEditor?org=HubSpot&repo=automation-ui-enrollment&path=automation-ui-enrollment/static/lang/en.lyaml&englishSource=GITHUB&filter=automation-ui-enrollment.aiAssistantFeedbackFooter.feedbackForm.help) the first issue from [one of my Code Orange audit reports](https://docs.google.com/spreadsheets/d/1eay5Kgibx_3NLZWP3O0KnDtB19OWCpZcZsRUPCo0iWQ/edit?gid=379834055#gid=379834055). 

![][image1]

Linguine supports:

* Selecting an **org** and **repo** (`org=HubSpot`, `repo=automation-ui-enrollment`).  
* Pointing at a specific **translation file path**: `automation-ui-enrollment/static/lang/en.lyaml`  
* Applying a **filter by key**:  
  * `filter=automation-ui-enrollment.aiAssistantFeedbackFooter.feedbackForm.help`  
    This narrows the UI down to the exact string I care about.

### **2.2 Edit copy directly in the UI**

Within that view, Linguine shows the English source string that maps to the issue in my report:

![][image2]

Issue:

* **Term:** `AI Assistant`  
* **Action needed:** `Replace: Deprecated term that was used prior to the launch of Breeze.`  
* **Full text:** `Your feedback helps to improve HubSpot AI Assistant.`  
* **Suggested change:** `Your feedback helps to improve Breeze.`

### **2.3 Automatically open a GitHub PR**

After saving my edit in Linguine:

* Linguine automatically opened a [**GitHub PR**](https://git.hubteam.com/HubSpot/automation-ui-enrollment/pull/363/commits/7a96458aaa256d276bc9b18c2de6ee720a72bbd1) in the same repo  
  * It creates a **branch and commit** with the updated translation file(s).  
  * Opens a **PR** so that engineers (or code owners) can review and merge the change.

# 3\. What this proves

This experiment shows that:

* I can take a **real terminology issue** from my checker and use **Linguine** to:  
  * Locate the correct translation entry by key.  
  * Edit the English copy safely in a UI.  
* And rely on Linguine to:  
  * Create a GitHub **PR** with the string change.  
  * Fit into existing review/merge processes.

## This is a concrete piece of progress on the **“Fix”** side of my Terminology OS: it validates Linguine as a viable manual/assisted mechanism for shipping small, safe terminology fixes without needing engineers to hand‑edit every string.

### 3.1 What’s still missing

Creating PRs is great, but:

* I’m still relying on teams to look at and address the PRs I open  
* I don’t have a way to get notified when the changes are made (unless I subscribe to every PR and manually update my records if/when I get an update  
* I need to manually create every PR and add a context comment \- it took about a minute to create my test, but there are thousands of issues to fix, so this is not sustainable

I need to find a way to automate this flow to have impact at scale.

Also, the .en translation files I’m fetching (based on a Sourcegraph export) don’t necessarily cover back-end code, which could mean there are gaps in my audit.

# Accessibility

# Accessibility Checking — Feature Analysis

## I got Claude Code to review the app and create a prompt to add WCAG Accessibility checking to the app. 

## This feature is parked for now, but something to consider adding in future.

# What your app can realistically check

Your app receives raw UI strings (stringKey/stringValue pairs) from JSON files. It cannot inspect rendered DOM, CSS, contrast ratios, focus order, or interactive behavior. That constrains the WCAG coverage significantly — but what remains is genuinely valuable and underserved by most tooling.

Targeting WCAG 2.1 AA is the right call for HubSpot. That's the level required for legal compliance in most jurisdictions (ADA, EN 301 549\) and is what HubSpot's accessibility commitments would be graded against.

---

### **Checks that are feasible from string content alone**

| WCAG Criterion | Level | What to detect |
| :---- | :---- | :---- |
| 2.4.4 Link Purpose | AA | Vague link text: "click here", "read more", "learn more", "here", "more info", "this link" |
| 1.3.3 Sensory Characteristics | A | Direction-only instructions: "see the box on the right", "click below", "shown above", "in the left panel" |
| 1.4.1 Use of Color | A | Color-as-sole-indicator: "the red button", "click the green", "blue link", "yellow warning text" |
| 2.4.6 Headings and Labels | AA | Vague labels: single-word labels with no context ("submit", "go", "ok" in isolation) |
| 3.3.1 Error Identification | A | Overly generic error messages: "An error occurred", "Something went wrong", "Try again" with no description |
| 3.3.2 Labels or Instructions | A | Placeholder-only patterns: very short strings that appear to serve as both label and placeholder |
| 1.3.1 / Reading | AA | ALL CAPS strings — screen readers often spell these out letter by letter |
| 3.1.5 Reading Level | AAA | Flesch-Kincaid grade level on help text, alerts, and tooltips exceeding \~grade 9 |
| Alt text patterns | A | Strings containing "image of", "photo of", "picture of", "icon of" — bad alt text conventions |
| Device-specific instructions | A | Mouse-only language in instructional text: "right-click to", "hover over", "drag and drop to" without keyboard equivalent |

---

### **Where this fits in your existing architecture**

Your app is well-architected for this extension. The key integration points are:

New service: AccessibilityCheckService.ts

* Mirrors the shape of WriterCSVService.ts but is rule-based, not dictionary-based  
* Each check maps to a WCAG criterion with a severity level  
* Uses the existing contextType from ContextDetectionService.ts to scope rules (e.g., color-only checks are higher priority in button context)

Extend TwoPassDetectionResult

* Add wcagCriterion?: string (e.g., "2.4.4")  
* Add wcagLevel?: 'A' | 'AA' | 'AAA'  
* Add termType variants: 'accessibility-issue'  
* Add new patternType values like "WCAG 2.4.4 \- Link Purpose"

Add a third detection pass

* After the two Writer passes, run an accessibility pass  
* Scoped by contextType: link-purpose checks only fire on strings classified as links/CTAs; sensory checks fire everywhere  
* Results flow through the same pipeline into StringData and Supabase

Minimal Supabase schema change

* Add wcag\_criterion, wcag\_level columns to terminology\_issues — or store in the existing patternType/notes fields if you want zero schema change initially

---

### **UI representation**

New summary card: "Accessibility Issues" alongside "Errors to Fix" / "Style Issues" / "Needs Review"

* Sub-counts by WCAG level (A vs AA)  
* Clicking it filters the table to accessibility issues

Table additions

* New termType badge style for accessibility issues (distinct color from terminology issues — e.g., purple vs. the current red/orange/yellow)  
* Show WCAG criterion reference inline (e.g., "WCAG 2.4.4") with a tooltip explaining the rule  
* Link to the WCAG criterion documentation page

Filter additions

* New filter option: "Issue type" — Terminology | Accessibility | Both  
* Within accessibility, filter by WCAG level (A / AA / AAA)

Settings Panel addition

* Toggle to enable/disable accessibility checking  
* Optionally: configure WCAG target level (A, AA, AAA) — so teams can scope to what they've committed to

---

### **Trade-offs and where this will and won't succeed**

Where it will succeed

* Link text (WCAG 2.4.4): "Click here" and "Read more" are extremely common failures in large codebases. Your string detection will catch these reliably. This is where you'll get the most signal.  
* Color-only instructions (WCAG 1.4.1): These are usually verbatim patterns ("the red button") that string matching handles well.  
* ALL CAPS detection: Near-zero false positive rate. If a string is fully uppercase, it's always worth flagging for screen reader behavior.  
* Sensory characteristics (WCAG 1.3.3): Directional instructions like "see the form on the right" are structurally detectable with high confidence.  
* Vague error messages: "Something went wrong" is always a problem regardless of context.

Where it will struggle

* False positives on link text: "Read more about our privacy policy" is fine contextually. "Read more" alone is not. Your app sees the string value, not the surrounding DOM. You'll flag both. Teams need to expect this and treat the check as a prompt to verify, not a definitive failure.  
* Context dependency: "Submit" as a button label might be fine if there's a visible form heading that provides context. Your app won't know. This is the fundamental limitation — WCAG compliance is architectural, and string analysis is only one layer.  
* Coverage gap: Realistically, string-level analysis can surface maybe 20–30% of practical WCAG AA issues. The remaining failures live in contrast ratios, keyboard navigation, focus management, ARIA usage — none of which are accessible to your app. Setting that expectation clearly in the UI matters.  
* Reading level checks: Flesch-Kincaid on short UI strings is noisy. "Save changes" scores as Grade 2\. "Authenticate via OAuth provider" scores as Grade 14\. Neither data point is very actionable on its own. This check is only useful on strings you've already classified as help text or descriptions — and even then, treat it as a low-confidence signal.  
* AAA criteria: WCAG AAA is aspirational. Including it is fine as an opt-in, but it shouldn't be on by default — it'll create noise that undermines trust in the higher-priority findings.

Recommended scope for v1  
Start with checks for WCAG A and AA only. Specifically prioritize: link purpose (2.4.4), sensory characteristics (1.3.3), use of color (1.4.1), and ALL CAPS detection. These have the best signal-to-noise ratio and lowest false positive rate from pure string analysis. Reading level and AAA can come later.

```
I want to add a new feature to this terminology app: accessibility compliance checking 
based on WCAG 2.1 guidelines. This should be a new detection pass that runs alongside 
the existing two-pass Writer/heuristic detection system. The target level is WCAG 2.1 AA.

## New Service: AccessibilityCheckService

Create a new service at `src/services/AccessibilityCheckService.ts`. This service should:

- Define a set of rule-based accessibility checks, each mapped to a WCAG criterion
- Accept a string value and a contextType (from the existing ContextDetectionService 
  classifications: 'buttons' | 'labels' | 'titles' | 'help' | 'alerts' | 'errors' | 'general')
- Return an array of AccessibilityIssue results

Each check should cover:

1. **WCAG 2.4.4 (AA) - Link Purpose**: Flag strings that are exactly or primarily 
   "click here", "read more", "learn more", "here", "more", "more info", "this link", 
   "this page", "link". Scope this check to context types: general, help, labels.

2. **WCAG 1.3.3 (A) - Sensory Characteristics**: Flag strings containing directional 
   or positional instructions used alone: "on the right", "on the left", "above", 
   "below", "click below", "shown above", "in the top", "in the bottom". These 
   instructions rely on visual layout which isn't available to screen reader users.

3. **WCAG 1.4.1 (A) - Use of Color**: Flag strings that reference color as the sole 
   means of conveying information, e.g., "the red button", "click the green", 
   "yellow warning", "blue link". Use pattern matching for color words (red, green, 
   blue, yellow, orange, purple, grey, gray, pink) adjacent to UI element nouns 
   (button, link, text, indicator, icon, dot, badge).

4. **WCAG 3.3.1 (A) - Error Identification**: Flag generic error strings that provide 
   no description. Patterns: "an error occurred", "something went wrong", "error", 
   "failed", "try again" — when the full string value is fewer than 6 words and 
   contains no specific context. Scope to context type: errors, alerts.

5. **ALL CAPS detection (WCAG 1.3.1)**: Flag any string where the entire value 
   (excluding whitespace and punctuation) is uppercase AND the string is longer than 
   2 characters. Screen readers often read these letter-by-letter. 

6. **WCAG 1.3.3 - Device-specific mouse instructions**: Flag strings containing 
   "right-click", "right click", "hover over", "mouse over" when used as the primary 
   instruction without a keyboard alternative. Scope to context types: help, general.

For each issue, return:
- `wcagCriterion: string` — e.g., "2.4.4"
- `wcagLevel: 'A' | 'AA' | 'AAA'`
- `wcagName: string` — e.g., "Link Purpose"
- `description: string` — a short plain-English description of why this is flagged
- `confidence: 'high' | 'medium' | 'low'`

## Extend the detection data model

In the relevant types file (wherever StringData and TwoPassDetectionResult are defined):
- Add `termType` variant: `'accessibility-issue'`
- Add optional fields to StringData: `wcagCriterion?: string`, `wcagLevel?: 'A' | 'AA' | 'AAA'`, `wcagName?: string`
- Add `patternType` values like `"WCAG 2.4.4 - Link Purpose"`, `"WCAG 1.3.3 - Sensory Characteristics"` etc.

## Integrate into the detection pipeline

In the detection engine (detectionEngine.ts or TwoPassDetectionService.ts), add a 
third pass after the existing two passes:
- Run AccessibilityCheckService on each string value + contextType
- Map results to the existing StringData shape using the new wcag fields
- Apply the same deduplication logic (one issue per stringKey+criterion)
- Accessibility issues should use confidence mapping: WCAG Level A → 'high', 
  Level AA → 'medium', Level AAA → 'low'
- Accessibility issues should use priority mapping: Level A → 'CRITICAL', 
  Level AA → 'REVIEW', Level AAA → 'OK'

## Settings toggle

In SettingsPanel.tsx, add a toggle to enable/disable accessibility checking. 
Default: enabled. Store this setting in localStorage with key 
`accessibilityCheckEnabled`. When disabled, skip the third pass entirely.

Also add a WCAG target level selector (A / AA / AAA) — default AA. Only run checks 
at or below the selected level. Store in localStorage as `wcagTargetLevel`.

## UI changes

### Summary card
Add a new summary card "Accessibility Issues" in CombinedTermSummary.tsx (or wherever 
the existing summary cards live). Show:
- Total count of accessibility issues
- Sub-count by level: "X Level A, Y Level AA"
- Use a distinct color (purple/violet tones work well, distinct from the existing 
  red/orange/yellow terminology cards)
- Clicking it should filter the results table to show only accessibility issues

### Table display
In EnhancedTermResultsTable.tsx:
- Add a distinct badge style for `termType === 'accessibility-issue'` — use a 
  purple/violet badge labeled "Accessibility"
- In the row or expanded detail, show the WCAG criterion reference: 
  e.g., "WCAG 2.4.4 – Link Purpose" as a small label
- Add a tooltip on the criterion label that explains the rule in plain English
- Add a link icon next to the criterion that links to 
  `https://www.w3.org/WAI/WCAG21/Understanding/[slug]` for the relevant criterion

### Filters
In TermFilters.tsx, add a new filter group "Issue Type" with options:
- All (default)
- Terminology only
- Accessibility only

Within the accessibility filter, optionally add sub-filters for WCAG level (A / AA).

## Important notes

- Accessibility checks should NOT replace or interfere with the existing Writer/heuristic 
  detection. They run as an additional pass and produce additional results in the same table.
- The accessibility pass should be skipped if the user has no strings (i.e., only run 
  when there are strings to process).
- Keep false positive awareness in mind: add a small disclaimer in the UI near the 
  accessibility summary card (e.g., a help icon tooltip that says: "These checks 
  identify potential WCAG issues in string content. Some results may require 
  manual verification — context not visible in string values alone may affect whether 
  an item is a true failure.")
- Do not implement reading level / Flesch-Kincaid scoring in this first version — 
  save that for a follow-up.
- Do not implement WCAG AAA checks in this first version.

```

[image1]: <data:image/png;base64,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>

[image2]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAZsAAAEJCAYAAABCNoqwAAAtg0lEQVR4Xu2d+bMdxZXn/ZeA5Iie/mFiHDGOmLZ/opmJ8AzT024m3J42bncThsbGuG2DAWOzGTdhFmNjdgxo39+mFQGWwWIVqxBCICEhBAiBENo3JMDAnXey6lSdPHlqu1X5Fun7jfhEVZ6su7736nOzMnX1hb/5yn/rAQAAADH5wl/919N6p963POMUxanTfE7JWObw+6ndgOllLDXaBjNSrJrJkpSiumCmwKrVZrHPrDaMGG3B7GqmCHSf1z+H2sMeU+bknKqQfWb/XJ9T5wz1Tp07NLpvQ33cX3acY55msFvmSwaas0Bg1TwW+SxswkKfRTlTxb6uOwaovSBhIGGqgutZ32DOlIH5vSmDCVPrMKSZZzOcIvcD5vqMjNZGaFuXOT6LR2uLaduE2b2pS2b3vrhkjiCpSaj2xaU5UzNmOWTfF11bsKwBy2k7M2F5TVZoZtTnfs10D1M2sm3JJt+SNGLJxkJJpS/ZECQOq6aYkLLRpJKpKRspGks2Xn9k2UyZY0ijXyplU1SvSalsrJpiHGWTyaRMNFo2qXCayoYkksuG6oZkastG1Fg2tYVDsiG0VKqQsmG0VEoIZJPU6skmEY4vGyUcLZQyxl02vnC+8J++fHogm9KRDdVT2cQd2VgoqUi5WLVCSBxWzUDLpi/hLPYJhNEEkopuCwyBaMpkw/35tkPZKOG4kU0qHGvkwiMbOcIpRMtm7mDCRJGNlItVK5NNI+GEsqlLIiMhm0U1ZCOk40Y23ugmkmwKhUOSYdIaS8SNdurQUjaLa8pGCCfrK5BNUutDNo5UNn0Lx5BKEYFolGz+6st/GwgmYVmKbhtMS7FqAUsFZX0p0wWqJuXiHTd9iZLPEleTaKmcorD7F2f4fdTm+uJALqcIdF/IiBITtRVaMLUYTtBycbWO0IKZncikGwaVfKjdkHmConrGgMcUgdknBJL10b4Wjqst8pgiWSD6aF9Lx7HQRwumlAU+Si6FsGRcO5WKq/G+QSqdkHkpcj9Ey2dKxlxRp/25rpahL6kNJzUmv9Qm5NIEQzCFkETkvsksn/RyWj1m+ixrwHLazshZPioHBdUYVxNymbp8em/qipy8j/ZDRmVzuhBKn7JhuVi1ACmTsr5QLlYtFA1BQtHtCtlQvYVsvD4lECmbauGMGG1FIJI6DKeM7mvZZCLqAC2bToQzmNBWNlIuVq1P2Xj9UjZEU9movlqykcLRbZNUNE1kw8Jh2TjhUM2QjIYlk+3Xk00ul1A2ed2QDdVKZGMJZ8rwKLTVYrFoIhtNIBpDNo2E00I2DiGbZTVkI4ST1DuRDQvGqhnUlg3B0iiqKwpkUzyyyWt5fxeyycVSJJv2IxuLEZ9AJHUZTraTTjYESWFIYAijLh3LRvZ1LhuqBbKRwkn3G8lGCMe4dFYPlg1hCKZUNiwZuW9QOLLRo55msvH60stnJJuphBZLEf3KxhROhWx0u1Q2swyhlKFlQ5fSKmSTCifrC2RjC8fJpnTORqGl4y8e8OWh52gK+3lOxqrVmbMpg+dkvBpJQvSVzdkUEczLlKHlUUEwTyMhQeh2A/RltErKZGLUSgjndHyh6HkYVxeCaTKfo2XSar5GQvMzSi6dUDVnU0TVnI2ExCL3TZRU1PyMnLPR8zZaMnrORl9O03M4LJmpmWiseRwFz81k+ywS0acFIyQTwkJJ2zxnY0qmBDdXw2i5lGPN2TB6dZonnmC+Rs3ZMHqOppRwRFMLmq9h6eRzNv3KJu2flLKxMIRSRiCUMrRQrJogEEwZSiZVBDKpQkjFiWUkE0eZbAoXCIyrbDoQjpMNw7JI61ogTYgmG1HrQza0Qq3WAoGsrmWTC6dKNiwckk0iE5JNhXAC2bBcZJtrdWTDwhFtEs5kkA3VmspGt01YNjNDqRSRyYb2J5JsWC5Wra1sWC5WrSvZ6HaAFopVEwRCKUOIpA6BTKpQsqEaC6RoxFNApWyUcLI+QyjjJpuMgZwuZCOFU3jpzKCJbDSBaELZuJqSjIUvIi2bRDhVstGX1cpXqglYNplwOpZNVkvRYimCZWNeOiumS9nQpbJANlIwsWSjmfyysWoGE1I2Vj0lEEoVQiZBWxHIpAolFScb+jc4qm7IRRPKhpCyGe5QNr5wgqXRgTz6YSChdFl0A6RsPOFQ3RCNKRstF6vWRDhJLRu9iEtrenTTlWyStpZNiXAC2Vg0kY1FS9k0EE6Xsikc3bBktHwKIeHMbCWcZIHAvcsTSCC871im5ELtPiHBZO2lCYGA1CKAKqYLvBpN+Kt+Y0VaKTMUWa18gUC2SEAQ9Cu5+KvVRoz+EQ9vkUCNxQOnZPvDQZ+3QMBaKGDVajPko0YyzRg02gI1kmnFPNoO+MhJ/6bMV8wLFwGUoUc6yZLohY6kRtsErmf9Wi4ZC1J0WxGMdCwSueSr1GpijGZqM6QJFwnIZdC8X7RQwK1GE3hyGclXqlmr1aaMzM6YSugFAUWwYBbPStCLBCwCwagFAmYtZVkRcpHAzGRJtFgYUM10hy8Xf1k0USEbKZqWwvFkQ5Ac0nrnsrEwhFJGoWyImLKx+tvJphwhFEssVq02Qz5qJNOOQR8tjDaYsqG6IZI6pJLhEUwoG+pPpKJFY8qG6qlw/JFOE9loDNHUls2CZETTSjYNhVNDNm5ZdIFs9Einrmx0nWXjJNOZbKhuiKaWcKyaIBANI2QjhZP++5s66JFMuWwsupJNAMlhmULJpA5aNoXCMYRShZZNVpsAshF4NSUTGtXkIxuLVCYaLZu+hDPko2XTarQz6KOF0ZoBnzayEcLJRjaecKgeSsaTjRBOHNkUCEdLpZRUNn0Lx5BKETVkk0mHZVMinLqysYQTjmzmhGIpZZYSToVspHDcvhQK9ynJNJTNt9c8mMumpnAmsGxGKZAN5T/PWRmKxWJcZJOLZiLIpvORTVTZEKkwWsmGGMxRczTtGfBxsiEMkdSBRzVONoSWDRGKxpNNKpyszxNN2qdl01Y4gVDKaCmbAUMqRdSQDf/7G082qXD6HdkUyYYk48umT+FksqkQjicbKRjZ5lod2bBwkv1bN7/kzsOWbFzdko0SjimbU+9bkaHl0mSBgOaiR190T0xmxitvhAsBRqHcvf71bP/CR9emCwCKoIn+ZP+ri1b1Pk/v/w8vvx4uBrAIFghU4QsmWCCga41QgtELAKxabZRcZo/W0sn/ou9GK6Pul3TmCwByavUb352WLwLwFxA0IsoCgRRvgQChJ//L+hRugYBqB4SSyRYCyH0TJZdgcYBP4Rd1qsUBell01u8tEkgWCtQmWCAgFwnIfYNggYBeKJC2eQEA1YJFAQX0/b1po4gl0c2/Fbp88QARLg6ooniRAEfXbtvyUs8tFAhGM+WLBzqVjW4XyYZGMWWy0avRKEWy+exz1oyfQC6aQCZVpEKxhGPVGrE4RArDqtUmlUwmm3b4X+IZQTaElk1Kp7KZa0ijX7Rs5mmJyH7dV4M6smHJaPkEtJCNq+UjmVqykcJx+4ZUighEIwUj9w0CyTCpWGSbZdOXcAyhVFHxb3DML+ocC9kYwrktHeFkohltZ6vSgtHMGMrGHWPIJq/lIrniqfWZGPYc+8ht5ciGZSPztcV/DqRDOWfVM6FMUmQuefxFr/7zJ9dlfa/uPeikQtl64HAmGU4gGuqflLJZnBKKpA7hN0ZL2SRLoitlUtVfKps+hRPIhrZSGtxnyKSKQDa0lcLQxxhCKaNT2SjhGIIpl00unLJ/8FkoG084FfIJRCOFo9t1ZWPQSjZ9CMcb2YTCmUiykcLJRJPJRgsnrRuiyWQTzNOUztkQcu6lqL6sd+HqRDZvHDiSwX2cb97/ZLZ/16hsaNRDSeZflvX2Hf/YtWn734cf8eZmbnphY3LsdHvOhnPRY2uzfZ574fxu7Savj0PfFE3yoAy/vr1izsZHi6lqzqYO/lyN7JP1kVAubSCZFM3lNIaEMezTes5GMFfR9aKBeRJjTqYE/dU2ch5HrlLL+uvO2aRtiTdXky0gEHKpnMNZ4KNGM9XIeZtEOsE8jWaQ52z0vA33GXM25rxNPn+j5RPM5wRzNpI5oXyMeZxCtFyC+ZkKgtHOLBsSjJ7DCWgyZ2ORzNFw5PyMOWcjRjquTtv+FwhosVg1XzYyVP/KglXJ/rR8cQDFl43fZ61Wo0tyrq9ENnKRAGVwy9uZVP561gpvpCP7Xtl7oPfXs5e7fSeMcZaNLxfdFmhhtKFT2YwyrrKhuq41IBUNjVq0TKoolA1RSzZUz4XjCUiPblgunmwWNZCNEo4azdRDy4YwJKOF48lHykbW6sjGF01z2Yi+ySKbusIJZGLDIx3KrZvXOenwogGWSi3ZCOE0l01Wqy8bXf/yvIeSekvZEJTzHn42qBfJZu6mNzOh/Je5Kz3ZzHst75PxZJMKp2vZWKvQZF/eP6L6qS3QwmjNsI8WSFO0bGIJp+sl0V2MbAgtm7TWVDbFI5tkdMPLoD0BsWwqhZOKZixlo6kjGykcJZ/OZMPCcaMdQyxFjIVsLALRKNlo4ei2gEKCSdokjnyVWqlsUuF0KBsLWzYyfCmN84sn87mbfmWjc+jjTzK5UJZsfSfb15fYlr2xQ/TlQuFkwuhUNksCofTPiE8gi7YM+6h5msaMm2y4zxBJHTzZEFIYVq1r2eTCqZZNKhdPNot82dQVjjeHo6VSgZrDCYRSRiCaGrIR7RNLNgQLRe4bBKLRwkn3K2RD0TWSDc/hZP8Op0g2qo+Ek/5/NrQwIF0gUCITc4GAQPdZq9GcbEbFIedqNu496LZaNnKlGm2lgFx/ukhAr0crWiDw98se9epnLFmd9ZF0stt5CwOSthaIXVPUXkCgBVJBsAigDC2PonqKsSigGC0TUXMy8eFFAUULBJi835eJXCCgBaMXBOjLanZ/IpJkZZq/OEAKJlgQUIkWzKAglEst5AIBTzJUC6WTC8ZCi6UETzZaOqJNMlGiKV+tRosAePFAsiBAyyVYMCCpWiAgoUUAcj8glE+wIKCUMtFU0Md3p8ml0FoqTb87TQslWzTACwHkvokczdDiACK/lBYsEPBWoxmyKRNKaZ9cqeaWRitZqFVqeT3fSopk4y+HZnTbx4kkFYtD9qeyWb1jV96vRVJYN8Ri1QKETIK2IhBKGUompX1aJnWRspHtUDa1VqNVykb0lcqkuD9nUMgmF46TUCCRpgwktF2NFsiG9/uVjRSObteRjUFj2fDKtFw4/cumQjiVsjGEEwiljHiysf4djlyVFsjE1ZvIJhdOMk+T9tWWjRBOJhtbOPbIpugSmkGpbAgtGyGcItlI6USXDQtH9N23Yasa1RBKGmbNQMuGawGpSJrKxqqVCqWsT0ukLvVko6WiZSP7k30lG+rTskmFoyVSJZu8bzAVTiibdqMbYiChC9lI6WT7LBveLyAQjRRMR7LRl9AW1ZRNJpzw8pkWjFdrIhtNIBpDNo2E00I2jlQ2JdIJ4JGNunyWL4tuLpt8ZEMI2VQKh0c3NWTjj2yEcJw08n9/UyiUsr5askmEE8iksWykYHS7BD2y4VqlbIr6BLVlQ7A8dFvRSDbEiKCsT0ukCSQUq5YTjlxCyYT9NWTjRj1FMimWja7Jf4fDEupfNAyJYVBhSKQunmw0hmi0bHQ7kE+/sgmF00w2VCuXTd8jG00gmq5lM9cQShlaNnNCuVg42aTCaSWbXDh+v5BNpXAELBw32tEjG70ooDYkj0QsCalMuiIQTLhAoBbTC2oeahFAFTMEVs1DLRBQBP1KLmXfnVb1vWlBv5ZLJcMhLBCr1oihECkRq1abQZ9MPGLU0jcDPqNS0ZP+fTE/hb+oMyWXDe3rxQM+/uKChYF4qr47jb+881TXXhDKp+/vTSPm5+gFAUUI+QSLA8oY0vgLBErR0hmmmvjeNLVYwNVpmxLIxzE7R8ulDJKJ25+VwHIpQwrGa89U+wZilFPNDB+9KEDRUjYsGcYQRhsmsGxohMIrz8pkkxynZUP1ySqbtBZFNsO+MKLIhtDy6JeBBCcbwhBIEyplk1BfNr5otGyslWksm2JOAtkMWbLJhZPJJhVOJqJS2QjhWKOZIjLZpMLxRjeGaEplo4kjG3M1WueycSOdDulKNhYtZePJxap5tJQN901k2fQlnCFFWtOy6Us6gz5aNp38G5yBhK5kk+HLRv/DTy0UKZ2gXiIbSzhZXyAZQzaNhSNk049wOv5W6DLKZUM1QzapcELJKNnQvpZKLUgeJBnGEI2WjiWgyLIpomDOJkXU63x3mtnnzdkIvPmafN6mEcF8TRnGvEwZwZxNGWI+RtNozqYIJaDGczZlKLnoeRdV4+9Eq/rGaPuLOjv47rRszobI5aK/M61qzsZEf1En4+ZeuE/PydQgmLMhpEB0uwQ5Z8PtJvM2uh2g5BLMy9joL+kkan93Wjpvo+XS/LvT5NyMVauat5FzN2l7hBDiCeZrNCSSuQqq1Ye/M8367rTse9NS9BxO2XenZavU1LxNOW3mbPx5mw5kQ/VQNpmEJqxsiuopgVDKUDKRtSiyWZIia/2SCiWTzWJfHIZs/C/iLOu3ZMMUyKSmbLh/Cokmk00unEaSaSQbwhBKGYFoJCQN3S6hsWxErS/ZLArEEkom/JLOXDa5cKpl4wtH/zscE5aKJx8pG1mrIxsWjpQN17RYSphAstH/9iYUShl9yIaFk+1L2YyKxaFlk9b6kY0nnFQ6QX+XstHtACWTwnpKIJQyUolYsrEIZFLFYsVoTcumb+Eo2VBNCqRgxFMkG58Isgn65MgmlE3f0okpG69G0tDtBjSRjSYQjSEbqhmSKUfLJhFOqWy8vkQ4uWxKhCNlk7U7lI1XS9FiKYJl40Y7DehXNlQzZBOMbIRwXD0QjIZlMyuUSi06lU0inLCvqWy0cHRb0YlsrD4tk7qkMgnaikAmVaQyCWSj+gKRNIHFMrovRzdKNs1GNr5wfNmMlMqkrmySfZIDyyYRTnvZMIMJ0WRjYUiliEA2Wi5WrUw2WjokG0ZLJScf2XQhm0Q4vmwqhKPlE9BENhZjJ5u+RzambGirRCMFo9smUjaElkkdRmXz+lvvKcEsb4YUiG4HGIsAyphWBk3063YDpjfFXwjA/wUBoRcIaLnoBQNhv15AsNiTSNkCgWpGfIKRTBuGfdRIphUkkVYLBBRCPI5OFgikzFO4RQM5JJfk3+vQvl4MII8La66uBOMvDKAabROqFgjkLEzRbUEwyimAL501XTBgjGYKGTTaHhWLBHRbQPKpt0AgIezzFwmECwMKYME0WQ4dCEYsAnBYtRQxysnRiwRmGQsBqkgWCGi5BAsExlc2Vk0QCKYMJZMq0hGM3BaR9J8osllsSKNfhn20MNpgyiata5HUoVI2VNe1mtSQTY4hE0XQH8gmF47f10Q2Fm2EQ9uGwjFGM7WpI5tUME4mhmzkSKcr2fT1ZZ1N/v1NIBstF6tWJhstHZINo6VShpbNzDGQjVXvSjZF9T5l0wxDNqlwgr5AJn3IRggn6A+EUsaIj5ZNq9HOsI8WhlVrxJBPG9lo4XjLogmqGyKpi5aNJ5yknoxwQplUMelk07dwDKkUUVc2LBm5b9KNbBqNbFg4wT/4pLohGks6bl9KhftkTRBIhhGyyYTDNS0Wi0khG6teIJWiuikbqyYIRi7F2CMbqseSDdVjyoZIhdFKNsRwjp6niSYbwpBJFZ5sGBYG9xkiqYMnG0LIwhvZ0DFKJoqg35RNIpywr41sDOFoqZQyOWXT5cgmkc3cUCqVaNlUCMeTjZSL6LOkE0hGC0fKhmtaLDa1ZIMgCIIgMQPZIAiCINED2SAIgiDRA9kgCIIg0QPZIAiCINED2SAIgiDRA9kgCIIg0QPZIAiCINED2SAIgiDRA9kgCIIg0QPZIAiCINED2SAIgiDRA9kgCIIg0QPZIAiCINED2SAIgiDRA9kgCIIg0QPZIAiCINHzhR07d+sagiAIgnQayAZBEASJHsgGQRAEiZ4JI5vTzzxblxAEQZATJFFl87V/PLf3pdPOdNw1Y6Gr0X6X4fv7w6xFnd83giAI0k2iykaf/Fk8xDW/uSPbl8fS9qtnnOXdlo/75Y139A4fOZrVue/eOYPe8bfcM8e1Zy9a2lv16FPBfSEIgiBjm6iyufG2aZ5QKLzPstF1ub3iulvcduWqx7KaJRvi+XUbXPvQ4SNuRMV9cvsvF1zWW/HH1ckNEQRBkDFLVNkMr1jlttveeic48VfJhuZwvv6dC7xjSCKWbOR2+vyR3rkXXukERXz++ee9b513cSYlBEEQZOwTVTZ8gieeWbs+q1Hqyuazzz7z7qdINnJfHi/7ZBtBEAQZu0SVDWX3nn261HeayGLrm9u9dpPbIgiCIN0mumy6iDVSaZo2t0UQBEHaZVLIBkEQBJncgWwQBEGQ6IFsEARBkOiBbBAEQZDoiSqb9/cfBQAAMInYf+S4PpV3kmiy2XvoWPAiAAAATHxiJJps9JMHAAAwOYgRyAYAAIBHjEA2AAAAPGIEsgEAAOARI5ANAAAAjxiBbCYZO/Ycctvtuw4EfWPBu+nj98umbe8GtYnIzn1HgloT2t5+IrBj98G+Xwfd7p0Pxu53lJ6rrlm8tXNfUAMhMTLhZfPk2leDWlvunb8026cv6NT9baD7k/ffNXfMGnLb8y7+VdDXhqWrnghqmnWvvRm8X7pdRdHxdHKq8xw0+mcp0cfW4e3397vta2/tDPqa8MIrW732gqWrvOf25nt7g9uU0fR3ih7j77/zw6BeF/q7u+a39wSvoy70/v30l78N6vI9uP72GUF/v9Bz1TWLfn8vTjZiZNxks3D5n3pv7tzb2/rOB67NW96nkw9tb7xzVvaphT7NL//TU9lxfGIYWPFIVhu8/8/e4zy3YUtv/Za3vfumXzh+PDlCWP3Meu+Pi495dv1m7z6ZN97dne3zJ/6yT070ByxHBvS66LlteH27a/OnSHrOfMyckQe9x9GyoT66PZ+8aOSgP41OX7Q829/27h63nbf4oaxGr/OfvndJ9n4y9DwefPRZr0Y/M96n9/G0r/+r17/57Z29x5/f4B3/8JoXvcfi/fmjJ2Def/WNHd5zoPfqkTXrsn5+3vQzkvclf5ZFJxL5+otq9P49/tyGQDb0+vX7ueKRNdnvJ9de2bqj98fHn8/a/HtEz5s+4ZNs5H0w9B4PP7A6qNPfB+/r10nInx+Pdul3hWv6vZDHW49BrH31jd79o6+N9qVsaF8+NjH8wKPuPeKfC0Gvkz8wSNnw8yuC7lv+nvF9yveF3lv+Hf/Tk2uDnwnLRv5OEXTcY8+9nLX5feHX088HnJOBGBkX2cg/BN63asTdc0ay2rpN27z+y6+/vfejy6/Pat/9yVXBfb6393C2z9KQ9//Nf/tpUNPPie5D//ES//Ofvpft8x/LBZf92vzUyrd/ZWvyv5bKGp2g75g56P5AredBf3y8r2XD9f991g+y/XMuvNptr7zhzuw5ch992r353vlejfj+Jf+R7XNf1XvH+xde9Ru3peO3bN/l9mcO3u+2q558wW35xMO3+cr/Ostt+edLJ3l+DtZj0POW7xt/ALGOlfDrv3Xawt76zW+592RjeilP3t/T6zZ5Nfmz+I+b7/OOpS1JXtb4QwTX6CRNr23mQPI+WLKxnjttr7st+cRv9b/02lvZPr22q35zV2/kwceC127dlo+n++DfI/n7wSdw2mfZ8G1f3Lit941zLiy8b9ryBwHaZ9nQh4gzz/6x99wkdCwLnu+LftbfPv9nwWNYv+MMPVf629P3w6KTz5O3q55Ifjfl6wEJMTLmsqFfaPlpSP8S6H0+Gf3o8hu8+6FPkiQbbsshuXWfhHXCJNnct2Cp90nXur2sEX9++qVS2Vx27S3ZsfJ2sjZ76IHs9nRf8tINn4z142vZ6JEFQ6Md/Zg0ypCXVuTrs2Qj2/q9o608OcnbyfuiNgtd357gkWCRbPhkIZ/3rKGVvUXLHw6OpX066RP03P7HN87xXr8FvSfyPnhEoi+j0TF3pu+9rOnnwNBr4Q8/BMmGnxuLSf+MaYSu7+uKG+4IHktDstGPz8fry3cW198+M3hclo0ccfAxK1c/E9T07en9I8noun4PdD/9HdS9/MdXBAh5GU0+J4nu4+M3vfle7cc8WYiRMZcNUfXLKvdZNvIPk058dDJpIhv6JKxPmASdCOmyyA9/fl3p7fUfBUGftHifP7kWjWz4U6GkTDbyBE3w4zeVja73IxvrvaMtvSZCPw5d8pA1uq31nsp2kWz403Jd2cj7tj5R62N0jUdk+hIRHTO00r/cVfSauCbrVSMbvtyj74t/3mWPVSabZX96srCvrMayIRHpY6yavj2PDHVdo/vp97ruib+ObPRtrD66XNf1HOhkJ0bGRTb0g75txoA3LKbtP//g5274LH8R5GU04q7Zw1l/lWyoRpK65Fc3u5o+YRLyMtrPRkcjdNKn56GPs35x6bov3b/8RFokG+q/dPR5UD8fWyYbvg2dWGlLQqRaE9nwtf55S/6YPWaVbPj++L2jiWnrvZNzJwT9LGcMrHAneBIBjURp7obuj547f1jg29OWrvtzW8qGfgfo/aBLi3y7fmTDNXr9NMohCVrvCT1H+V5TjU+W9Jpo+8ub7s7ujy8v8bF0qYyeJ12i4hqPkLhtyYbeK3pd0xYuy46j32nanzP8QOFro30ajdOWfldZNvI2ZcdzjUc91P79vfPd+0BznlSTl9F+f98Ct6UFInzbm+6e40ZufHv6ebljR++HtnLORj4XDc0bUT9d2uXjmsiGb2PJhubgaF+eM+SWuGXawtLnd7ISI+MiGwAAGE8gmHJiBLIBAJx0QDblxAhkAwAAwCNGIBsAAAAeMQLZAAAA8IgRyAYAAIBHjESTDf5baAAAmJzESDTZUI599JfgRQAAAJi4xEpU2SAIgiAIBbJBEARBogeyQRAEQaIHskEQBEGiB7JBEARBogeyQRAEQaIHskEQBEGiB7JBEARBogeyQRAEQaIHskEQBEGiB7JBEARBogeyQRAEQaIHskEQBEGiB7JBEARBoieqbL502pkZstZP+Hb/9+wfuf3Tzzw769uyfVfv4puHMhAEQZCJleiy0ft6e/a//yJrP/70C16/FJW1PXjoSO/YR594koFwEARBJl7GRDYvvfKaKQvKt867OGsT5110tdv++Be/7j2zdn1wPOWFl17N2nJEI3ln1/7seARBEGR8E1023zznwt73L77Gq8mtlM39qx71+uS+VaPoy2e8//TL27JjEARBkPFNdNnoSHls3/Ge15ay+ZcLLuste/DPlbJ5e+deUzYIgiDIxMm4yebcC690+z/82bVZnWXDbYbbsk9GX0L7YN9hrx9BEAQZ30SVDYIgCIJQIBsEQRAkeiAbBEEQJHogGwRBECR6IBsEQRAkeiAbBEEQJHqiyWbN+m29o8c/7pQXNm539zvWIAiCIO0STTZ7DhwNZNEFR459FNRigyAIgrRLFNl8/vnnwQm7Kw5/eHzMhYMgCIK0S+eyIdF8+umnwQm7Kw4e+RCyQRAEmWSJIpu//OUvwQm7Kw4cPgrZIAiCTLJMOtnsP3TEXUrT9ZggCIIg7RJFNp988klwwu4KyAZBEGTyBbKpAYIgCNIukE0NEARBkHY5YWUzsGpttj/88Dqv3RQEQRCkXU5o2bBghh/JZbPv0Idu/9lX3gpuUwSCIAjSLie0bF7c/I7bLn98QyYb3j60ZmPvoac3BrezQBAEQdrlhJYNbx97cavbvrv7YO/+J14JjqkCQRAEaZcTXja8L+Xz9vv73HbTW+8Ht7NAEARB2uWElU0Z23ftD2plIAiCIO1yUsqmKQiCIEi7QDY1QBAEQdoFsqkBgiAI0i6QTQ0QBEGQdoFsaoAgCIK0SzTZxPpvoSEbBEGQyZdostmw9d3e6+/sDk7c/ULyWrN+W++xFzb3nly31e2PBe/vOaRfIoIgCNIwUWRD/3nasWPHeocPH+4dOHCgt2/fvlbs37+/d+jQod6HH37oRPbZZ5/ph0UQBEEmcKLI5tNPP+199NFHvaNHjzpJHDx4sBV0H0eOHOkdP37ciYweA0EQBJk86Vw2FBp5kBQ+/vhjJwga5bSB7oPkxaMayAZBEGRyJYpsSAYEiYFGOSSeNtB9EBANgiDI5EwU2XBYOl2BIAiCTM5ElQ2CIAiCUCAbBEEQJHogGwRBECR6IBsEQRAkeiAbBEEQJHogGwRBECR6IBsEQRAkeiAbBEEQJHogGwRBECR6osnm5Y2be1867cysTfuyLfPVM87SJQRBEOQESjTZULRsKA8+8oTbX/XoU1mNZDOw9MGsrW9XJCkEQRBkciSqbAaXPdT7u7POd/tSJO9/sMeTSJFsqH7rvXN7l15zU++7P7rc1RAEQZDJl6iyoeiRCe8/lI5wKEWy4dvq+0AQBEEmV8ZFNus2bPLqJJvjxz9y7UOHj3iyOf3Ms90lNxoNIQiCIJMz0WVD8pALAJ5Zu95JZMnKhz3ZUC6++kZTThjVIAiCTO5Elw2CIAiCQDYIgiBI9EA2CIIgSPRANgiCIEj0QDYIgiBI9ESVzQf7D/fWrN8GAABgkhAr0WSzfsuO3tHjHwMAAJhE7NxzUJ/OO0k02ZAh9YsAAAAw8YkRyAYAAIBHjEA2AAAAPGIEsgEAAOARI5ANAAAAjxiBbAAAAHjEyAkjmwNHPux947s/Cep1uOXeub1NW9/qXXfLfUFfXXa8vzuoNYW+3VrXumYsHiMGk/V5l3HZtTcHNQAmAjEyLrKRJ45ZA8t67+/ZHxzTlDYnI5LNi69sPilkc9OdM4PaeKFfr27X6aP6OT+50m33HTwS9FcxY+GSoEYsWJL8Z37c3rb9vey/u9DHEpde89vCviJeeHlTUGtD08cHoIgYGRfZLFj8QG/9ptfdPv+BzB5cHvwxW/t8zBvb38366P/DoRptqf3dH1/h2lfdcHt2DNc+2HfQu09CyobEp5/HPXOGXPuiq5L/b0ff/vCHxz3Z/L9zLzIFSsf+20VXB8+Jn6d8jU89n/y/P2vWvhw8nnW/xOZt2732oaPHzGPlMbrNx7265c2gRvs/vvy64Db0ONvf2xUcS9s/Pvq027dO6vJ42bbuh7aPP7vObZc9tDrrt95ngn8n9P3r+yZ+8etbgtvrY4nf3DEjOK7seF3j9sEjyc/lzpmL3JZ/vx57eq37MMDH6Z/n9bcmH4bo93TvwcOuxqN5PkY/BwD6IUbGRTaE/MOgE/2KVY95fXJbVNP3R9sfXnZtVqNPo2/u2OndhoWk75tlc9t98736rr0HerfeNy94HHl7ekyWDdWtE+DFv7zJbenksHbDa6XPk7Z0MqF9EteV19/WO+Nb38+O55OVhX5dr7/5jtmvj6Pt4WPJ/5Za1G/VivpvmzbfiX31U8+79sNPPOu9Bj6e5MtY9yNr9D7RPl1++tb3Ls7qBH94Kbu9rhGWBO+dO9y7ffoCt//bu2Zl9TLZPPHcut6R0fePXie19c/rmpvuyto7d+9z29/dPdtt+fm8/e6u4H6t502/p7v3Jx+arr9tWtAPQFtiZNxkQ1gngqK6VSu6jUR+UmT0/dCJxbqMRsfwSU3W6JP8P/zrv3t1kg31ffv8S726ZtHSh9wnc/2c+HnyY+jH5C0/vr5ffSxBx7Fcdb88ztrfIiT106tvDI6Towv5fOhky8fx5S0JH6fvT7at51N0LEOjAF0j6PXT85Mn/ztnLOzNG1np9i3ZFD3nItkUHc9tfn+4/e6uPa7NsnngkSddnZ6Xvu/B5auyfTmy4RrNNcrH07cHoB9iZELI5vf3zDXr+g9X16puo/uKarTPspEnTqrTiEuOJIoeR45s6DKhfjxGykb3Wff92htvB8fqdlHf+Zf8qjd9gX8ytR7D2pcyrboNPY6sc9/KR56oLUbZth5H1nallzrf2enPk/ExchRh3Z729x866vYt2UgxSVmXyUbuHzic3LfVL9ssG10vqvE+ZANiEyMTQja8z3CNPslRW15/L/qDqrovXaNLHtymS2csG7psVXZbrvNIxt1+2nxvzub//PMPssscGpaNvl9u6zrXWDrE8+s3Bvcr4eOs1U76sYr217yQzBkRz657NTjuodVPmY+jR3bydfClQf1Yus3zEYy+H3k7q8ajHILmkqgm5+JoboSP1bL5yRXXe22CZPrypq3B4xD0u/nWjve9Gh2jf150SZHbPFqRl9GIv/2Hs4PHpg9i3H/t7/7galWyoUuScm4SgKbEyLjKBtjoE9p4IOdAAAAnFzEC2UxAIBsAwHgSI5ANAAAAjxiBbAAAAHjECGQDAADAI0YgGwAAAB4xAtkAAADwiJFosqGQcAAAAEweNm7bqU/lnSSqbBAEQRCEAtkgCIIg0QPZIAiCINED2SAIgiDRA9kgCIIg0QPZIAiCINED2SAIgiDRA9kgCIIg0QPZIAiCINED2SAIgiDRA9kgCIIg0QPZIAiCINED2SAIgiDRA9kgCIIg0RNdNrMXLe196bQze7v37NNdCIIgyEmSqLK5e+YiJ5rzLrrabZ9+/iV9SGHo+LqhY197fZsuj2uaPH8EQZATPVFlY51w/+6s812doH0Kt/l42f7aP56b7V9x3S29TVu2uf3vX3yN2y5c/IB3W46+T/m4PMqSxzB0fwNLHwzqxEVX3hDcTrefWbs+6EcQBDnZM+aykTV5sqacfubZZp8+eesTOe3Lkc2nn34aPLY+3tryPsvG6vvqGWeZz4fz9e9cENQQBEFO9kSXDZ2ceV9un3ruxaBWJBvOx598ktWIe+cMZm19GY1v9/y6DV7belz5GLRfJptb7pnjRluUlaseC46BbBAEQcJElQ2FxcAn30OHj2Ttw0eOZsdQLNnwKIW49JqbnLzmDa1wt5VC0LKRl7Mo8nE58vayViYb3sr7ksdANgiCIGGiywZBEARBIBsEQRAkeiAbBEEQJHogGwRBECR6IBsEQRAkeiAbBEEQJHqiyualzTt6R49/DAAAYJKwZn2cr/6KJht6wvpFAAAAmPjECGQDAADAI0YgGwAAAB4xAtkAAADwiBHIBgAAgEeMQDYAAAA8YgSyAQAA4BEjkA0AAACPGIFsAAAAeMQIZAMAAMAjRiAbAAAAHjEC2QAAAPCIEcgGAACAR4xANgAAADxiBLIBAADgESOQDQAAAI8YgWwAAAB4xAhkAwAAwCNGIBsAAAAeMRJNNpSXNu8IXgQAAICJCw0UYiSqbBAEQRCEAtkgCIIg0QPZIAiCINED2SAIgiDRA9kgCIIg0QPZIAiCINED2SAIgiDRA9kgCIIg0QPZIAiCINED2SAIgiDRA9kgCIIg0QPZIAiCINED2SAIgiDRA9kgCIIg0QPZIAiCINED2SAIgiDRA9kgCIIg0QPZIAiCINED2SAIgiDRA9kgCIIg0QPZIAiCINED2SAIgiDRA9kgCIIg0QPZIAiCINED2SAIgiDRA9kgCIIg0QPZIAiCINED2SAIgiDRA9kgCIIg0QPZIAiCINED2SAIgiDR8/8B1RLdgEoqNRgAAAAASUVORK5CYII=>