# Merge-Check Design Spec

## Summary

Add an `ocr merge-check` subcommand that analyzes an MR/Change and determines whether it should be merged into each of the specified target branches. Input can be a Gerrit Change URL, a GitLab MR URL, or a local branch name. All data fetching is done via git (no platform-specific REST APIs).

## Motivation

When an MR is submitted, developers need to decide whether the changes should be cherry-picked or merged into multiple release/hotfix branches. This decision currently requires manual analysis. Merge-check automates this by having an LLM analyze the diff and produce a structured recommendation per target branch.

## Command Interface

```bash
# Via Gerrit Change URL
ocr merge-check \
  --url "https://gerrit.example.com/c/project/+/12345" \
  --branches "release/1.0,release/2.0,release/3.0" \
  --format json

# Via GitLab MR URL
ocr merge-check \
  --url "https://gitlab.com/group/project/-/merge_requests/1" \
  --branches "release/1.0,release/2.0"

# Via local branch (backward compatible)
ocr merge-check \
  --from feature-branch \
  --branches "release/1.0,release/2.0"
```

### Flags

| Flag | Required | Description |
|------|----------|-------------|
| `--url` | One of `--url` or `--from` | Gerrit Change URL or GitLab MR URL |
| `--from` | One of `--url` or `--from` | Local source branch name |
| `--branches` | Yes | Comma-separated list of target branches to evaluate |
| `--repo` | No | Git repository root (default: current directory) |
| `--format` | No | Output format: `json` (default) |

`--url` and `--from` are mutually exclusive.

## URL Resolution Strategy

### Gerrit Change URL

Format: `https://gerrit.example.com/c/project/+/12345`

Resolution steps:
1. Parse URL to extract: host, project path, change number
2. Run `git ls-remote <host>/<project> refs/changes/*/*/<change_number>*` to discover available ref
3. `git fetch origin refs/changes/34/12345/1` to fetch the commit
4. Use `FETCH_HEAD` as source commit; resolve source branch from commit message or fall back to URL metadata

### GitLab MR URL

Format: `https://gitlab.com/group/project/-/merge_requests/1`

Resolution steps:
1. Parse URL to extract: host, project path, MR iid
2. Run `git ls-remote <host>/<project> merge-requests/<iid>/head` to discover the ref
3. `git fetch origin refs/merge-requests/<iid>/head` to fetch the commit
4. Use `FETCH_HEAD` as source; resolve source branch name from remote HEAD

Note: For both platforms, the remote URL is derived from the git config of `--repo`. If `--url` specifies a host that differs from the configured remote, the user must ensure the remote is configured correctly. A future enhancement could add `--remote` flag.

### Local Branch (`--from`)

Resolution:
1. `git merge-base origin/<target_branch> origin/<from>` to compute merge base
2. `git diff <merge_base>..origin/<from>` for diff
3. For "already merged" optimization: `git merge-base --is-ancestor <from_commit> origin/<target_branch>`

## Core Flow

```
merge-check entry
    |
    +-- Parse input: --url or --from
    |
    +-- git fetch all target branches
    |     git fetch origin refs/heads/<branch1> refs/heads/<branch2> ...
    |
    +-- Resolve source:
    |     if --url: git ls-remote + git fetch refs/... -> source_commit
    |     if --from: source_commit = resolve ref to SHA
    |
    +-- Compute diff (cached, same for all branches):
    |     git merge-base <source> <target_branches[0]> as base
    |     git diff <base>..<source>
    |
    +-- For each target_branch (sequential):
    |     |
    |     +-- If --from mode:
    |     |     git merge-base --is-ancestor $source_commit origin/$branch
    |     |     true  -> already_merged, skip LLM
    |     |     false -> continue
    |     |
    |     |     (Gerrit changes are never "already merged" by definition)
    |     |
    |     +-- LLM single call (no tool loop):
    |           prompt = merge_check_template + diff + target_branch
    |           -> parse JSON output
    |
    +-- Aggregate and output JSON result
```

### Why Sequential?

User chose sequential analysis over concurrent to:
- Reduce concurrent API pressure
- Allow potential future sharing of LLM context between branches

### Diff Caching

All target branches share the same diff (it's the MR's diff against the common ancestor). Cache the diff text and compute it once.

Edge case: different target branches may have different merge-bases with the source. In that case, use the widest diff (diff against the earliest merge-base) to ensure all changes are visible. In practice, for most workflows, all target branches share the same base branch (e.g., `main`), so this is usually the same diff.

## Output Format

```json
{
  "source": {
    "type": "branch | url",
    "ref": "feature-branch | https://gerrit.example.com/c/project/+/12345",
    "commit": "abc1234"
  },
  "branches": [
    {
      "branch": "release/1.0",
      "status": "already_merged",
      "recommendation": null,
      "confidence": null,
      "reason": "Commit abc1234 is already reachable from release/1.0"
    },
    {
      "branch": "release/2.0",
      "status": "needs_analysis",
      "recommendation": "merge",
      "confidence": "high",
      "reason": "Bug fix for null pointer exception that exists in release/2.0",
      "affected_areas": ["UserService.go:45", "config_loader.go:12"],
      "risk": "Low conflict risk, changes only touch shared module"
    },
    {
      "branch": "release/3.0",
      "status": "needs_analysis",
      "recommendation": "skip",
      "confidence": "medium",
      "reason": "New feature API does not exist in release/3.0, merging provides no value",
      "affected_areas": [],
      "risk": "Merging may introduce dead code or unnecessary dependencies"
    }
  ],
  "summary": "2 of 3 branches analyzed (1 already merged). Recommend merging into release/2.0."
}
```

### Status Values

| Status | Meaning |
|--------|---------|
| `already_merged` | Source commit is already reachable from this branch |
| `needs_analysis` | Source commit is not in branch, LLM analysis performed |
| `error` | Failed to analyze (fetch error, LLM error, etc.) |

### Recommendation Values

| Value | Meaning |
|-------|---------|
| `merge` | Changes are relevant and should be merged |
| `skip` | Changes are not relevant or would cause issues |
| `uncertain` | Insufficient information to make a recommendation |

### Confidence Values

| Value | Meaning |
|-------|---------|
| `high` | Clear signal based on shared code paths or bug fixes |
| `medium` | Some relevance but ambiguous |
| `low` | Mostly guesswork |

## Template Design

New file: `internal/config/template/merge_check_template.json`

The template is a single-shot LLM call (no plan phase, no tool loop). It contains:

- **system message**: Role definition, scoring criteria, output format
- **user message**: Placeholders `{{source_branch}}`, `{{target_branch}}`, `{{diff}}`, `{{change_files}}`

Scoring criteria for the LLM:
1. Does the target branch contain the same bug that this MR fixes?
2. Does this MR modify code paths that the target branch also executes?
3. Would merging provide tangible value to the target branch's users?
4. Would merging introduce unnecessary code, dependencies, or conflicts?

The LLM is instructed to output strict JSON, no markdown wrapping.

## Code Structure

### New Files

```
cmd/opencodereview/
  merge_check_cmd.go               # Subcommand entry point, flag parsing, orchestration

internal/mergecheck/
  merge_check.go                    # Core logic: iterate branches, call LLM, aggregate
  merge_check_test.go               # Unit tests
  url.go                            # URL parsing: extract host, project, change ID
  url_test.go                        # URL parsing tests

internal/config/template/
  merge_check_template.json          # Embedded template (go:embed)
```

### Modified Files

```
cmd/opencodereview/main.go           # Register merge-check subcommand
cmd/opencodereview/flags.go           # (no change, new flags in merge_check_cmd.go)
```

### Reused Modules (no changes)

```
internal/diff/                        # ParseDiffText, Provider for local mode
internal/llm/                         # LLMClient, ChatRequest
internal/config/template/             # Template loading pattern (reference only, new template separate)
```

## "Already Merged" Optimization

For `--from` mode (local branch):
- Run `git merge-base --is-ancestor $source_commit origin/$target_branch`
- If exit code 0: mark as `already_merged`, skip LLM call entirely
- If exit code 1: needs analysis

For `--url` mode (Gerrit Change):
- Gerrit changes (open/not-merged) are never in any branch by definition
- Skip the `is-ancestor` check entirely
- Always run LLM analysis

For `--url` mode (GitLab MR):
- Same as `--from`: check `is-ancestor` after fetching the MR commit
- This handles the case where the MR was already merged into some target branches

## Error Handling

| Error | Behavior |
|-------|----------|
| Invalid URL format | Fail fast with usage message |
| `git ls-remote` fails (no such change) | Report error, exit |
| `git fetch` fails for a branch | Skip that branch, report error in output |
| LLM returns non-JSON | Retry once with stricter prompt; if still fails, mark branch as `error` |
| LLM API error | Mark branch as `error`, continue with next branch |
| No target branches specified | Fail fast with usage message |
| All branches already merged | Output JSON with all `already_merged`, no LLM calls |

## Relationship to Existing `review` Command

| Aspect | `ocr review` | `ocr merge-check` |
|--------|-------------|-------------------|
| Purpose | Find code bugs/issues | Decide merge eligibility per branch |
| Input | --from/--to/--commit | --url/--from + --branches |
| LLM flow | Per-file tool-use loop | Single call per branch |
| Tools used | file_read, code_search, etc. | None (diff only) |
| Output | Code review comments | Merge recommendation JSON |
| Template | task_template.json | merge_check_template.json |

## Open Questions

None. All decisions have been made:
- Input: `--url` or `--from` (mutually exclusive)
- Data source: git only (no REST API calls)
- Analysis scope: MR diff only
- Output: structured JSON
- Branch analysis: sequential
- Already-merged detection: `git merge-base --is-ancestor` where applicable