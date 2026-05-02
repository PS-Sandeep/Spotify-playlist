import requests
from bs4 import BeautifulSoup
import spotipy
from spotipy.oauth2 import SpotifyOAuth
import os
from dotenv import load_dotenv

load_dotenv()

CLIENT_ID = os.getenv("SPOTIFY_CLIENT_ID")
CLIENT_SECRET = os.getenv("SPOTIFY_CLIENT_SECRET")
REDIRECT_URI = os.getenv("SPOTIFY_REDIRECT_URI")

date = input("Enter a date (YYYY-MM-DD): ").strip()
year = date.split("-")[0]

url = f"https://www.billboard.com/charts/hot-100/{date}/"

headers = {
    "User-Agent": "Mozilla/5.0"
}

response = requests.get(url, headers=headers)

if response.status_code != 200:
    print("Failed to fetch Billboard data. Check the date or try again.")
    exit()

# Parse HTML
soup = BeautifulSoup(response.text, "html.parser")

song_list = soup.select("li ul li h3")
song_names = [song.get_text(strip=True) for song in song_list]

if not song_names:
    print("No songs found. Check if the date is valid.")
    exit()

print(f"Top {len(song_names)} songs fetched!")

sp = spotipy.Spotify(
    auth_manager=SpotifyOAuth(
        client_id=CLIENT_ID,
        client_secret=CLIENT_SECRET,
        redirect_uri=REDIRECT_URI,
        scope="playlist-modify-private"
    )
)

user_id = sp.current_user()["id"]
print("Logged in as:", user_id)

song_uris = []

for song in song_names:
    try:
        result = sp.search(
            q=f"track:{song} year:{year}",
            type="track",
            limit=1
        )
        tracks = result["tracks"]["items"]
        if tracks:
            song_uris.append(tracks[0]["uri"])
        else:
            print(f"{song} not found on Spotify")
  except Exception as e:
        print(f"Error searching {song}: {e}")

playlist = sp.user_playlist_create(
    user=user_id,
    name=f"{date} Billboard Hot 100",
    public=False
)

if song_uris:
    sp.playlist_add_items(playlist_id=playlist["id"], items=song_uris)
    print("Playlist created successfully!")
else:
    print("No songs were added.")
