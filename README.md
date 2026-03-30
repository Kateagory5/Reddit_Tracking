# Reddit_Tracking
Python script to track mentions on reddit
Reddit Mention Monitor
A lightweight, read-only Python script for monitoring Reddit posts and comments that mention specific keywords across a defined list of subreddits. Results are saved to a local CSV file for internal review.
Purpose
This script is used internally to identify Reddit discussions where subject matter experts at our company can participate and provide helpful, relevant responses to community members asking questions about network security, NAC, Zero Trust, RADIUS, ZTNA, TACACS, and related topics.
The goal is genuine community participation — connecting Redditors who have questions with people who can help answer them.
What It Does

Searches a defined list of subreddits for keyword matches in posts and comments
Tracks the last run timestamp so each run only returns new results since the previous run
Saves results to a local CSV file (never transmitted or shared externally)
Runs locally on a single private machine on demand

What It Does NOT Do

❌ Does not post, comment, vote, or interact with Reddit in any way
❌ Does not run automatically or continuously (manual execution only)
❌ Does not store or transmit Reddit data to any external service
❌ Does not access any non-public data
❌ Does not act on behalf of any user other than the authenticated account owner

Technical Details

Language: Python 3
API access: Read-only, via PRAW (Python Reddit API Wrapper)
App type: Script (personal use)
Authentication: OAuth2 via Reddit script-type app credentials
Rate limiting: Handled automatically by PRAW in compliance with Reddit's API guidelines
Data storage: Local CSV file only — no database, no cloud storage, no third-party services

Setup
1. Install dependency
bashpip install praw
2. Set environment variables
Credentials are stored as environment variables and never hardcoded in the script:
bashexport REDDIT_CLIENT_ID="your_client_id"
export REDDIT_CLIENT_SECRET="your_client_secret"
export REDDIT_USERNAME="your_reddit_username"
3. Run
bashpython reddit_monitor.py
Output
Results are appended to a local CSV file with the following columns:
ColumnDescriptiontypepost or commentsubredditSubreddit where the match was foundkeywordThe keyword that matchedtitle_or_bodyPost title or comment text (truncated to 300 chars)urlDirect link to the post or commentauthorReddit username of the authorscoreUpvote score at time of retrievalcreatedTimestamp of the original post/commentrun_dateTimestamp of when the script was run
Compliance
This script is intended for use in full compliance with Reddit's API Terms of Service and Responsible Builder Policy.
