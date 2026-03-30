#!/usr/bin/env python3
"""
Reddit Keyword Monitor
Searches specified subreddits for keywords in posts and comments,
then appends results to a CSV file.

Setup:
    pip install praw

Usage:
    python reddit_monitor.py
"""

import csv
import praw
import time
import json
import os
import sys
from datetime import datetime, timezone

# ─────────────────────────────────────────────
# CONFIGURATION — edit these before running
# ─────────────────────────────────────────────

# Reddit API credentials
# DO NOT hardcode these — set them as environment variables instead:
#
#   export REDDIT_CLIENT_ID="your_id"
#   export REDDIT_CLIENT_SECRET="your_secret"
#   export REDDIT_USERNAME="your_reddit_username"
#
# Tip: Add these lines to your ~/.zshrc or ~/.bash_profile so they
# persist across terminal sessions without ever touching this file.
REDDIT_CLIENT_ID     = os.environ.get("REDDIT_CLIENT_ID")
REDDIT_CLIENT_SECRET = os.environ.get("REDDIT_CLIENT_SECRET")
REDDIT_USER_AGENT    = f"reddit-monitor/1.0 by {os.environ.get('REDDIT_USERNAME', 'unknown')}"

# Keywords to search for (case-insensitive)
KEYWORDS = [
    "portnox",
    "network access control",
    "NAC solution",
    "zero trust NAC",
    "RADIUS cloud",
    "802.1X",
    "cloud RADIUS",
    "network segmentation",
    "IoT security",
    "device compliance",
    "certificate authentication",
    "ZTNA",
    "zero trust network access",
    "TACACS",
    "TACACS+",
    "network device administration",
    "privileged access",
]

# Subreddits to search (without r/)
SUBREDDITS = [
    "sysadmin",
    "netsec",
    "networking",
    "cybersecurity",
    "msp",
    "netsecstudents",
    "homelab",
    "healthcare_it",
    "pci_dss",
    "itpro",
    "zerotrust",
    "fortinet",
    "paloaltonetworks",
    "Cisco",
    "ccna",
    "ccnp",
]

# File to store the last-run timestamp (created automatically)
STATE_FILE = os.path.expanduser("~/.reddit_monitor_state.json")

# How far back to look on the VERY FIRST run (in days), before any state exists
FIRST_RUN_LOOKBACK_DAYS = 7

# Where to save results — edit this path to wherever you'd like the file
# Each run APPENDS new rows (so you build up a history over time)
CSV_OUTPUT = os.path.expanduser("~/Desktop/reddit_monitor_results.csv")

# ─────────────────────────────────────────────
# SCRIPT — no need to edit below this line
# ─────────────────────────────────────────────

def validate_config():
    """Fail fast if credentials are missing."""
    missing = [k for k in ("REDDIT_CLIENT_ID", "REDDIT_CLIENT_SECRET", "REDDIT_USERNAME")
               if not os.environ.get(k)]
    if missing:
        print(f"[ERROR] Missing environment variable(s): {', '.join(missing)}")
        print("Set them in your shell before running — see config comments above.")
        sys.exit(1)


def sanitize_for_csv(value):
    """
    Prevent CSV injection: strip leading formula characters that Excel/Sheets
    would execute if the cell starts with =, +, -, or @
    """
    s = str(value).strip()
    if s and s[0] in ("=", "+", "-", "@", "\t", "\r"):
        s = "'" + s
    return s



def load_last_run():
    """Return the timestamp of the last run, or None if first run."""
    try:
        with open(STATE_FILE, "r") as f:
            data = json.load(f)
        ts = data.get("last_run")
        if not isinstance(ts, (int, float)) or ts <= 0 or ts > time.time() + 60:
            print("[WARN] State file contains invalid timestamp — starting fresh.")
            return None
        return ts
    except (FileNotFoundError, json.JSONDecodeError):
        return None


def save_last_run(ts):
    """Save the current run timestamp to the state file."""
    with open(STATE_FILE, "w") as f:
        json.dump({"last_run": ts}, f)


def init_reddit():
    return praw.Reddit(
        client_id=REDDIT_CLIENT_ID,
        client_secret=REDDIT_CLIENT_SECRET,
        user_agent=REDDIT_USER_AGENT,
    )


def matches_keyword(text):
    """Return the first matching keyword, or None."""
    text_lower = text.lower()
    for kw in KEYWORDS:
        if kw.lower() in text_lower:
            return kw
    return None


def is_recent(created_utc, cutoff_ts):
    return created_utc >= cutoff_ts


def search_subreddit(reddit, subreddit_name, cutoff_ts):
    results = {"posts": [], "comments": []}
    sub = reddit.subreddit(subreddit_name)

    # ── Posts ──────────────────────────────────
    try:
        for post in sub.new(limit=200):
            if not is_recent(post.created_utc, cutoff_ts):
                continue
            text = f"{post.title} {post.selftext}"
            kw = matches_keyword(text)
            if kw:
                results["posts"].append({
                    "keyword":   kw,
                    "title":     post.title,
                    "url":       f"https://reddit.com{post.permalink}",
                    "author":    str(post.author),
                    "score":     post.score,
                    "created":   datetime.fromtimestamp(post.created_utc, tz=timezone.utc).strftime("%Y-%m-%d %H:%M UTC"),
                    "subreddit": subreddit_name,
                })
    except praw.exceptions.PRAWException as e:
        print(f"  [!] Reddit API error fetching posts from r/{subreddit_name}: {e}")
    except Exception as e:
        print(f"  [!] Unexpected error fetching posts from r/{subreddit_name}: {type(e).__name__}: {e}")

    # ── Comments ───────────────────────────────
    try:
        for comment in sub.comments(limit=500):
            if not is_recent(comment.created_utc, cutoff_ts):
                continue
            kw = matches_keyword(comment.body)
            if kw:
                results["comments"].append({
                    "keyword":   kw,
                    "body":      comment.body[:300].replace("\n", " ") + ("…" if len(comment.body) > 300 else ""),
                    "url":       f"https://reddit.com{comment.permalink}",
                    "author":    str(comment.author),
                    "score":     comment.score,
                    "created":   datetime.fromtimestamp(comment.created_utc, tz=timezone.utc).strftime("%Y-%m-%d %H:%M UTC"),
                    "subreddit": subreddit_name,
                })
    except praw.exceptions.PRAWException as e:
        print(f"  [!] Reddit API error fetching comments from r/{subreddit_name}: {e}")
    except Exception as e:
        print(f"  [!] Unexpected error fetching comments from r/{subreddit_name}: {type(e).__name__}: {e}")

    return results


CSV_COLUMNS = ["type", "subreddit", "keyword", "title_or_body", "url", "author", "score", "created", "run_date"]


def write_csv(all_results, since_dt):
    run_date    = datetime.now().strftime("%Y-%m-%d %H:%M")
    file_exists = os.path.isfile(CSV_OUTPUT)
    rows        = []

    for subreddit_name, results in all_results.items():
        for p in results["posts"]:
            rows.append({
                "type":          "post",
                "subreddit":     sanitize_for_csv(f"r/{subreddit_name}"),
                "keyword":       sanitize_for_csv(p["keyword"]),
                "title_or_body": sanitize_for_csv(p["title"]),
                "url":           sanitize_for_csv(p["url"]),
                "author":        sanitize_for_csv(f"u/{p['author']}"),
                "score":         p["score"],
                "created":       p["created"],
                "run_date":      run_date,
            })
        for c in results["comments"]:
            rows.append({
                "type":          "comment",
                "subreddit":     sanitize_for_csv(f"r/{subreddit_name}"),
                "keyword":       sanitize_for_csv(c["keyword"]),
                "title_or_body": sanitize_for_csv(c["body"]),
                "url":           sanitize_for_csv(c["url"]),
                "author":        sanitize_for_csv(f"u/{c['author']}"),
                "score":         c["score"],
                "created":       c["created"],
                "run_date":      run_date,
            })

    with open(CSV_OUTPUT, "a", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=CSV_COLUMNS)
        if not file_exists:
            writer.writeheader()   # Only write header on first ever run
        writer.writerows(rows)

    return len(rows)


def main():
    validate_config()
    run_ts = time.time()

    last_run = load_last_run()
    if last_run:
        cutoff_ts = last_run
        since_dt  = datetime.fromtimestamp(cutoff_ts, tz=timezone.utc)
        since_str = since_dt.strftime("%Y-%m-%d %H:%M UTC")
    else:
        cutoff_ts = run_ts - FIRST_RUN_LOOKBACK_DAYS * 24 * 60 * 60
        since_dt  = datetime.fromtimestamp(cutoff_ts, tz=timezone.utc)
        since_str = f"{since_dt.strftime('%Y-%m-%d %H:%M UTC')} (first run — {FIRST_RUN_LOOKBACK_DAYS}-day default)"

    print(f"Reddit Monitor starting — {datetime.now().strftime('%Y-%m-%d %H:%M')}")
    print(f"Subreddits: {', '.join(SUBREDDITS)}")
    print(f"Keywords:   {', '.join(KEYWORDS)}")
    print(f"Since:      {since_str}\n")

    reddit = init_reddit()
    all_results = {}

    for sub in SUBREDDITS:
        print(f"  Scanning r/{sub}...")
        all_results[sub] = search_subreddit(reddit, sub, cutoff_ts)
        posts_found    = len(all_results[sub]["posts"])
        comments_found = len(all_results[sub]["comments"])
        print(f"    → {posts_found} post(s), {comments_found} comment(s)")

    total = sum(len(r["posts"]) + len(r["comments"]) for r in all_results.values())
    print(f"\nTotal hits: {total}")

    # Save the run timestamp BEFORE sending email so a send failure doesn't reset state
    save_last_run(run_ts)

    if total == 0:
        print("Nothing found — CSV not updated.")
        return

    rows_written = write_csv(all_results, since_dt)
    print(f"✅ {rows_written} row(s) written to: {CSV_OUTPUT}")


if __name__ == "__main__":
    main()
