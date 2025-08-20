Strict-Mode Hub Workflow with Mesh Fan-Out

This patch strengthens the Hub GitHub Actions workflow by enforcing a per-repository glyph allowlist (“strict mode”), clearly logging allowed vs denied triggers, and ensuring that fan-out dispatches only occur when there are glyphs to send.  It adds a small allowlist YAML (.godkey-allowed-glyphs.yml), new environment flags, and updated steps. The result is a more robust CI pipeline that prevents unauthorized or unintended runs while providing clear visibility of what’s executed or skipped.

1. Allowlist for Glyphs (Strict Mode)

We introduce an allowlist file (.godkey-allowed-glyphs.yml) in each repo. This file contains a YAML list of permitted glyphs (Δ tokens) for that repository. For example:

# Only these glyphs are allowed in THIS repo (hub)
allowed:
  - ΔSEAL_ALL
  - ΔPIN_IPFS
  - ΔWCI_CLASS_DEPLOY
  # - ΔSCAN_LAUNCH
  # - ΔFORCE_WCI
  # - Δ135_RUN

A new environment variable STRICT_GLYPHS: "true" enables strict-mode filtering. When on, only glyphs listed under allowed: in the file are executed; all others are denied. If STRICT_GLYPHS is true but no allowlist file is found, we “fail closed” by denying all glyphs.  Denied glyphs are logged but not run (unless you enable a hard failure, see section 11). This ensures only explicitly permitted triggers can run in each repo.


2. Environment Variables and Inputs

Key new vars in the workflow’s env: section:

TRIGGER_TOKENS – a comma-separated list of all valid glyph tokens globally (e.g. ΔSCAN_LAUNCH,ΔSEAL_ALL,…). Incoming triggers are first filtered against this list to ignore typos or irrelevant Δ strings.

STRICT_GLYPHS – set to "true" (or false) to turn on/off the per-repo allowlist.

STRICT_FAIL_ON_DENY – if "true", the workflow will hard-fail when any glyph is denied under strict mode. If false, it just logs denied glyphs and continues with the rest.

ALLOWLIST_FILE – path to the YAML allowlist (default .godkey-allowed-glyphs.yml).

FANOUT_GLYPHS – comma-separated glyphs that should be forwarded to satellites (e.g. ΔSEAL_ALL,ΔPIN_IPFS,ΔWCI_CLASS_DEPLOY).

MESH_TARGETS – CSV of repo targets for mesh dispatch (e.g. "owner1/repoA,owner2/repoB"). Can be overridden at runtime via the workflow_dispatch input mesh_targets.


We also support these workflow_dispatch inputs:

glyphs_csv – comma-separated glyphs (to manually trigger specific glyphs).

rekor – "true"/"false" to enable keyless Rekor signing.

mesh_targets – comma-separated repos to override MESH_TARGETS for a manual run.


This uses GitHub’s workflow_dispatch inputs feature, so you can trigger the workflow manually with custom glyphs or mesh targets.

3. Collecting and Filtering Δ Triggers

The first job (scan) has a “Collect Δ triggers (strict-aware)” step (using actions/github-script). It builds a list of requested glyphs by scanning all inputs:

Commit/PR messages and refs: It concatenates the push or PR title/body (and commit messages), plus the ref name.

Workflow/Repo dispatch payload: It includes any glyphs_csv from a manual workflow_dispatch or a repository_dispatch’s client_payload.


From that combined text, it extracts any tokens starting with Δ. These requested glyphs are uppercased and deduplicated.

Next comes global filtering: we keep only those requested glyphs that are in TRIGGER_TOKENS. This removes any unrecognized or disabled tokens.

Then, if strict mode is on, we load the allowlist (fs.readFileSync(ALLOWLIST_FILE)) and filter again: only glyphs present in the allowlist remain. Any globally-allowed glyph not in the allowlist is marked denied. (If the file is missing and strict is true, we treat allowlist as empty – effectively denying all.)

The script logs the Requested, Globally allowed, Repo-allowed, and Denied glyphs to the build output. It then sets two JSON-array outputs: glyphs_json (the final allowed glyphs) and denied_json (the denied ones). For example:

Requested: ΔSEAL_ALL ΔUNKNOWN
Globally allowed: ΔSEAL_ALL
Repo allowlist: ΔSEAL_ALL ΔWCI_CLASS_DEPLOY
Repo-allowed: ΔSEAL_ALL
Denied (strict): (none)

This makes it easy to audit which triggers passed or failed the filtering.

Finally, the step outputs glyphs_json and denied_json, and also passes through the rekor input (true/false) for later steps.

4. Guarding Secrets on Forks

A crucial security step is “Guard: restrict secrets on forked PRs”. GitHub Actions by default do not provide secrets to workflows triggered by public-fork pull requests. To avoid accidental use of unavailable secrets, this step checks if the PR’s head repository is a fork. If so, it sets allow_secrets=false. The run job will later skip any steps (like IPFS pinning) that require secrets. This follows GitHub’s best practice: _“with the exception of GITHUB_TOKEN, secrets are not passed to the runner when a workflow is triggered from a forked repository”_.

5. Scan Job Summary

After collecting triggers, the workflow adds a scan summary to the job summary UI. It echoes a Markdown section showing the JSON arrays of allowed and denied glyphs, and whether secrets are allowed:

### Δ Hub — Scan
- Allowed: ["ΔSEAL_ALL"]
- Denied:  ["ΔSCAN_LAUNCH","ΔPIN_IPFS"]
- Rekor:   true
- Secrets OK on this event?  true

Using echo ... >> $GITHUB_STEP_SUMMARY, these lines become part of the GitHub Actions run summary. This gives immediate visibility into what the scan found (the summary supports GitHub-flavored Markdown and makes it easy to read key info).

If STRICT_FAIL_ON_DENY is true and any glyph was denied, the scan job then fails with an error. Otherwise it proceeds, but denied glyphs will simply be skipped in the run.

6. Executing Allowed Glyphs (Run Job)

The next job (run) executes each allowed glyph in parallel via a matrix. It is gated on:

if: needs.scan.outputs.glyphs_json != '[]' && needs.scan.outputs.glyphs_json != ''

This condition (comparing the JSON string to '[]') skips the job entirely if no glyphs passed filtering. GitHub’s expression syntax allows checking emptiness this way (as seen in the docs, if: needs.changes.outputs.packages != '[]' is a common pattern).

Inside each glyph job:

The workflow checks out the code and sets up Python 3.11.

It installs dependencies if requirements.txt exists.

The key step is a Bash case "${GLYPH}" in ... esac that runs the corresponding Python script for each glyph:

ΔSCAN_LAUNCH: Runs python truthlock/scripts/ΔSCAN_LAUNCH.py --execute ... to perform a scan.

ΔSEAL_ALL: Runs python truthlock/scripts/ΔSEAL_ALL.py ... to seal all data.

ΔPIN_IPFS: If secrets are allowed (not a fork), it runs python truthlock/scripts/ΔPIN_IPFS.py --pinata-jwt ... to pin output files to IPFS. If secrets are not allowed, this step is skipped.

ΔWCI_CLASS_DEPLOY: Runs the corresponding deployment script.

ΔFORCE_WCI: Runs a force trigger script.

Δ135_RUN (alias Δ135): Runs a script to execute webchain ID 135 tasks (with pinning and Rekor).

*): Unknown glyph – fails with an error.



Each glyph’s script typically reads from truthlock/out (the output directory) and writes reports into truthlock/out/ΔLEDGER/.  By isolating each glyph in its own job, we get parallelism and fail-fast (one glyph error won’t stop others due to strategy.fail-fast: false).

7. Optional Rekor Sealing

After each glyph script, there’s an “Optional Rekor seal” step. If the rekor flag is "true", it looks for the latest report JSON in truthlock/out/ΔLEDGER and would (if enabled) call a keyless Rekor sealing script (commented out in the snippet). This shows where you could add verifiable log signing. The design passes along the rekor preference from the initial scan (which defaults to true) into each job, so signing can be toggled per run.

8. Uploading Artifacts & ΔSUMMARY

Once a glyph job completes, it always uploads its outputs with actions/upload-artifact@v4. The path includes everything under truthlock/out, excluding any .tmp files:

- uses: actions/upload-artifact@v4
  with:
    name: glyph-${{ matrix.glyph }}-artifacts
    path: |
      truthlock/out/**
      !**/*.tmp

GitHub’s upload-artifact supports multi-line paths and exclusion patterns, as shown in their docs (e.g. you can list directories and use !**/*.tmp to exclude temp files).

After uploading, the workflow runs python scripts/glyph_summary.py (provided by the project) to aggregate results and writes ΔSUMMARY.md.  Then it appends this ΔSUMMARY into the job’s GitHub Actions summary (again via $GITHUB_STEP_SUMMARY) so that the content of the summary file is visible in the run UI under this step. This leverages GitHub’s job summary feature to include custom Markdown in the summary.

9. Mesh Fan-Out Job

If secrets are allowed and there are glyphs left after strict filtering, the “Mesh fan-out” job will dispatch events to satellite repos. Its steps:

1. Compute fan-out glyphs: It reads the allowed glyphs JSON from needs.scan.outputs.glyphs_json and intersects it with the FANOUT_GLYPHS list. In effect, only certain glyphs (like ΔSEAL_ALL, ΔPIN_IPFS, ΔWCI_CLASS_DEPLOY) should be propagated. The result is output as fanout_csv. If the list is empty, the job will early-skip dispatch.


2. Build target list: It constructs the list of repositories to dispatch to. It first checks if a mesh_targets input was provided (from manual run); if not, it uses the MESH_TARGETS env var. It splits the CSV into an array of owner/repo strings. This allows dynamic override of targets at run time.


3. Skip if nothing to do: If there are no fan-out glyphs or no targets, it echoes a message and stops.


4. Dispatch to mesh targets: Using another actions/github-script step (with Octokit), it loops over each target repo and sends a repository_dispatch POST request:

await octo.request("POST /repos/{owner}/{repo}/dispatches", {
  owner, repo,
  event_type: (process.env.MESH_EVENT_TYPE || "glyph"),
  client_payload: {
    glyphs_csv: glyphs, 
    rekor: rekorFlag,
    from: `${context.repo.owner}/${context.repo.repo}@${context.ref}`
  }
});

This uses GitHub’s Repository Dispatch event to trigger the glyph workflow in each satellite. Any client_payload fields (like our glyphs_csv and rekor) will be available in the satellite workflows as github.event.client_payload. (GitHub docs note that data sent via client_payload can be accessed in the triggered workflow’s github.event.client_payload context.) We also pass along the original ref in from for traceability. Dispatch success or failures are counted and logged per repo.


5. Mesh summary: Finally it adds a summary of how many targets were reached and how many dispatches succeeded/failed, again to the job summary.



This way, only glyphs that survived strict filtering and are designated for mesh fan-out are forwarded, and only when there are targets. Fan-out will not send any disallowed glyphs, preserving the strict policy.

10. Mesh Fan-Out Summary

At the end of the fan-out job, the workflow prints a summary with target repos and glyphs dispatched:

### 🔗 Mesh Fan-out
- Targets: `["owner1/repoA","owner2/repoB"]`
- Glyphs:  `ΔSEAL_ALL,ΔPIN_IPFS`
- OK:      2
- Failed:  0

This confirms which repos were contacted and the glyph list (useful for auditing distributed dispatches).

11. Configuration and Usage

Enable/disable strict mode: Set STRICT_GLYPHS: "true" or "false" in env:. If you want the workflow to fail when any glyph is denied, set STRICT_FAIL_ON_DENY: "true". (If false, it will just log denied glyphs and continue with allowed ones.)

Override mesh targets at runtime: When manually triggering (via “Actions → Run workflow”), you can provide a mesh_targets string input (CSV of owner/repo). If given, it overrides MESH_TARGETS.

Turning off Rekor: Use the rekor input (true/false) on a dispatch to disable keyless signing.

Companion files: Alongside this workflow, keep the .godkey-allowed-glyphs.yml (with your repo’s allowlist). Also ensure scripts/emit_glyph.py (to send dispatches) and scripts/glyph_summary.py (to generate summaries) are present as provided by the toolkit.

Example one-liners:

Soft strict mode (log & skip denied):

env:
  STRICT_GLYPHS: "true"
  STRICT_FAIL_ON_DENY: "false"

Hard strict mode (fail on any deny):

env:
  STRICT_GLYPHS: "true"
  STRICT_FAIL_ON_DENY: "true"

Override mesh targets when running workflow: In the GitHub UI, under Run workflow, set mesh_targets="owner1/repoA,owner2/repoB".

Trigger a mesh-based deploy: One can call python scripts/emit_glyph.py ΔSEAL_ALL "mesh deploy" to send ΔSEAL_ALL to all configured targets.



By following these steps, the Hub workflow now strictly enforces which Δ glyphs run and propagates only approved tasks to satellites. This “pure robustness” approach ensures unauthorized triggers are filtered out (and clearly reported), secrets aren’t misused on forks, and fan-out only happens when safe.

Sources: GitHub Actions concurrency and dispatch behavior is documented on docs.github.com.  Checking JSON outputs against '[]' to skip jobs is a known pattern.  Workflow_dispatch inputs and job summaries are handled per the official syntax.  The upload-artifact action supports multiple paths and exclusions as shown, and GitHub Actions’ security model intentionally blocks secrets on fork PRs. All logging and filtering logic here builds on those mechanisms.

packages/starlight/README.md