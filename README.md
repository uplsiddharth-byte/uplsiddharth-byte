"""
Detects which repo you've been most actively pushing to recently and
rewrites the <!--CURRENT_PROJECT:START--> ... <!--CURRENT_PROJECT:END-->
block in README.md accordingly.

Required env vars:
  GH_USERNAME   - your GitHub username (e.g. uplsiddharth-byte)
  GH_TOKEN      - a PAT with repo read access (so private-repo pushes count too)

Optional env vars:
  GH_PROFILE_REPO      - defaults to "<username>/<username>"
  LOOKBACK_DAYS         - defaults to 10
  EXTRA_EXCLUDE_REPOS   - comma-separated "owner/repo" list to always ignore
                          (e.g. coursework or notes repos you don't want shown)
"""

import os
import re
import sys
from datetime import datetime, timedelta, timezone
from collections import Counter

import requests

USERNAME = os.environ["GH_USERNAME"]
TOKEN = os.environ["GH_TOKEN"]
PROFILE_REPO = os.environ.get("GH_PROFILE_REPO", f"{USERNAME}/{USERNAME}")
LOOKBACK_DAYS = int(os.environ.get("LOOKBACK_DAYS", "10"))

EXCLUDE_REPOS = {PROFILE_REPO.lower()}
EXCLUDE_REPOS |= {
    r.strip().lower()
    for r in os.environ.get("EXTRA_EXCLUDE_REPOS", "").split(",")
    if r.strip()
}

HEADERS = {
    "Authorization": f"token {TOKEN}",
    "Accept": "application/vnd.github+json",
}


def get_recent_push_events():
    """Pull the user's recent events, stop once we're past the lookback window."""
    events = []
    cutoff = datetime.now(timezone.utc) - timedelta(days=LOOKBACK_DAYS)
    page = 1
    while page <= 3:  # events API caps at ~300 events / 10 pages anyway
        resp = requests.get(
            f"https://api.github.com/users/{USERNAME}/events",
            headers=HEADERS,
            params={"per_page": 100, "page": page},
            timeout=30,
        )
        resp.raise_for_status()
        batch = resp.json()
        if not batch:
            break
        stop = False
        for e in batch:
            created = datetime.strptime(
                e["created_at"], "%Y-%m-%dT%H:%M:%SZ"
            ).replace(tzinfo=timezone.utc)
            if created < cutoff:
                stop = True
                break
            if e["type"] == "PushEvent":
                events.append(e)
        if stop:
            break
        page += 1
    return events


def pick_current_project(events):
    """Rank repos by commit volume in the window, tie-break by recency."""
    counts = Counter()
    latest_ts = {}
    for e in events:
        repo = e["repo"]["name"]  # "owner/repo"
        if repo.lower() in EXCLUDE_REPOS:
            continue
        n_commits = len(e.get("payload", {}).get("commits", []))
        counts[repo] += max(n_commits, 1)
        ts = e["created_at"]
        if repo not in latest_ts or ts > latest_ts[repo]:
            latest_ts[repo] = ts
    if not counts:
        return None
    ranked = sorted(
        counts.items(), key=lambda kv: (kv[1], latest_ts[kv[0]]), reverse=True
    )
    return ranked[0][0]


def get_repo_details(full_name):
    resp = requests.get(
        f"https://api.github.com/repos/{full_name}", headers=HEADERS, timeout=30
    )
    resp.raise_for_status()
    data = resp.json()
    lang_resp = requests.get(data["languages_url"], headers=HEADERS, timeout=30)
    languages = list(lang_resp.json().keys())[:4] if lang_resp.ok else []
    return {
        "name": data["name"],
        "url": data["html_url"],
        "description": data.get("description") or "No description set yet.",
        "languages": languages,
        "pushed_at": data["pushed_at"][:10],
    }


def build_block(repo):
    if repo is None:
        return f"🚧 No push activity in the last {LOOKBACK_DAYS} days."
    stack = " · ".join(repo["languages"]) if repo["languages"] else "—"
    return (
        f"🚧 **Currently working on:** [{repo['name']}]({repo['url']})  \n"
        f"{repo['description']}  \n"
        f"Stack: {stack} · last pushed {repo['pushed_at']}"
    )


def update_readme(block, path="README.md"):
    with open(path, "r", encoding="utf-8") as f:
        content = f.read()
    pattern = re.compile(
        r"<!--CURRENT_PROJECT:START-->.*<!--CURRENT_PROJECT:END-->", re.DOTALL
    )
    if not pattern.search(content):
        print("Markers not found in README.md — add them first, see setup notes.")
        sys.exit(1)
    replacement = f"<!--CURRENT_PROJECT:START-->\n{block}\n<!--CURRENT_PROJECT:END-->"
    new_content = pattern.sub(replacement, content)
    if new_content != content:
        with open(path, "w", encoding="utf-8") as f:
            f.write(new_content)
        print("README updated:\n", block)
    else:
        print("No change needed.")


if __name__ == "__main__":
    events = get_recent_push_events()
    repo_full_name = pick_current_project(events)
    repo = get_repo_details(repo_full_name) if repo_full_name else None
    update_readme(build_block(repo))
