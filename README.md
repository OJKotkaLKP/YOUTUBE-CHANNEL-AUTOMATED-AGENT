# YOUTUBE-CHANNEL-AUTOMATED-AGENT

import os
import requests
import openai
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow
from googleapiclient.discovery import build
from datetime import datetime

# --- CONFIGURATION ---

# Set these as environment variables or directly in code
GITHUB_TOKEN = os.getenv('GITHUB_TOKEN')          # Personal GitHub token
OPENAI_API_KEY = os.getenv('OPENAI_API_KEY')      # OpenAI API Key
CLIENT_SECRETS_FILE = "credentials.json"          # OAuth2 credentials from Google Cloud Console

# YouTube API Scopes
SCOPES = ["https://www.googleapis.com/auth/youtube.upload"]

# --- 1. Fetch GitHub Repo Activity or Content ---

def fetch_github_readme(owner, repo):
    url = f"https://api.github.com/repos/{owner}/{repo}/readme"
    headers = {"Authorization": f"token {GITHUB_TOKEN}", "Accept": "application/vnd.github.v3.raw"}
    resp = requests.get(url, headers=headers)
    if resp.status_code == 200:
        return resp.text
    else:
        print("Failed to fetch README:", resp.text)
        return ""

# --- 2. Generate Video Script with OpenAI ---

def generate_script_from_text(text, prompt="Create a YouTube video script based on this repo:"):
    openai.api_key = OPENAI_API_KEY
    response = openai.Completion.create(
        engine="gpt-3.5-turbo-instruct",
        prompt=f"{prompt}\n\n{text}",
        max_tokens=500,
        temperature=0.7,
    )
    return response["choices"][0]["text"].strip()

# --- 3. (Optional) Create Video with VEED.IO or similar API ---
# You can integrate with VEED.IO API if available, or skip to upload manual videos.

# --- 4. Authenticate and Upload Video to YouTube ---

def youtube_authenticate():
    flow = InstalledAppFlow.from_client_secrets_file(CLIENT_SECRETS_FILE, SCOPES)
    creds = flow.run_local_server(port=0)
    return build("youtube", "v3", credentials=creds)

def upload_video(youtube, video_file, title, description, tags=None, categoryId="22", privacyStatus="unlisted"):
    request_body = {
        "snippet": {
            "title": title,
            "description": description,
            "tags": tags or [],
            "categoryId": categoryId,
        },
        "status": {
            "privacyStatus": privacyStatus
        }
    }
    mediaFile = MediaFileUpload(video_file, mimetype="video/*", resumable=True)
    request = youtube.videos().insert(
        part="snippet,status",
        body=request_body,
        media_body=mediaFile
    )
    response = request.execute()
    print("Video uploaded! Video ID:", response["id"])
    return response["id"]

# --- 5. Fetch YouTube Video Analytics ---

def fetch_video_stats(youtube, video_id):
    response = youtube.videos().list(
        part="statistics",
        id=video_id
    ).execute()
    stats = response["items"][0]["statistics"]
    print("Video Stats:", stats)
    return stats

# --- MAIN AGENT FLOW ---

def main():
    owner = "OJKotkaLKP"
    repo = "YOUTUBE-CHANNEL-AUTOMATED-AGENT"

    # 1. Fetch repo README/content
    readme_text = fetch_github_readme(owner, repo)

    # 2. Generate video script (AI)
    video_script = generate_script_from_text(readme_text)
    print("Generated Video Script:\n", video_script)
    
    # 3. (Optional) Use VEED.IO API to create video from script

    # 4. Upload video to YouTube
    youtube = youtube_authenticate()
    video_file = "path/to/your/generated_or_manual_video.mp4"  # Replace with your video path
    video_id = upload_video(
        youtube,
        video_file=video_file,
        title="Automated Video for GitHub Repo",
        description=video_script,
        tags=["GitHub", "YouTube", "Automation"],
    )

    # 5. Fetch analytics
    fetch_video_stats(youtube, video_id)

if __name__ == "__main__":
    main()