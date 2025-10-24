def detect_language(text):
    try:
        lang = detect(text)
        return lang if lang != "en" else "none"  
    except:
        return "unknown"

def clean_name(name):
    return re.sub(r'[^a-zA-Z0-9\s/()\-]', '', name).strip()

def safe_artist_lookup(sp, artist_id, retries=5):
    """Fetch artists, api limits apply."""
    for attempt in range(retries):
        try:
            return sp.artist(artist_id)
        except spotipy.exceptions.SpotifyException as e:
            print("rate limit")
            time.sleep(5)
    return {"genres": ["unknown"]}

if os.path.exists(MEMORY_PATH):
    with open(MEMORY_PATH, "r") as f:
        memory = json.load(f)
else:
    memory = {"processed_ids": [], "playlists": {}}

processed_ids = set(memory["processed_ids"])
playlists_cache = memory["playlists"]

artist_cache = {}

all_tracks = []
results = sp.current_user_saved_tracks(limit=50)

while results:
    for item in results["items"]:
        track = item["track"]
        if not track or not track["id"]:
            continue
        if track["id"] in processed_ids:
            continue

        artist = track["artists"][0]
        artist_id = artist["id"]

        if artist_id in artist_cache:
            genres = artist_cache[artist_id]
        else:
            artist_info = safe_artist_lookup(sp, artist_id)
            genres = artist_info.get("genres", ["unknown"])
            artist_cache[artist_id] = genres

        lang = detect_language(track["name"] + " " + artist["name"])

        all_tracks.append({
            "id": track["id"],
            "name": track["name"],
            "artist": artist["name"],
            "genres": genres,
            "language": lang
        })

    if results["next"]:
        results = sp.next(results)
        time.sleep(0.5) 
    else:
        break

print(f" {len(all_tracks)} new liked songs to classify")

genre_groups = {}
for song in all_tracks:
    for genre in song["genres"]:
        genre_clean = genre.title()
        genre_groups.setdefault(genre_clean, []).append(song["id"])

lang_groups = {}
for song in all_tracks:
    lang = song["language"]
    if lang not in ["none", "unknown"]:
        lang_key = lang.upper()
        lang_groups.setdefault(lang_key, []).append(song["id"])

def update_playlists(groups, prefix=""):
    for name, track_ids in tqdm(groups.items(), desc=f"Updating {prefix or 'playlists'}"):
        playlist_name = clean_name(f"{prefix}{name}")
        if len(playlist_name) > 100:
            playlist_name = playlist_name[:97] + "..."

        if playlist_name not in playlists_cache:
            playlist = sp.user_playlist_create(
                user=user_id,
                name=playlist_name,
                public=False,
                description=f"Auto-generated {prefix.lower()}playlist: {len(track_ids)} tracks"
            )
            playlists_cache[playlist_name] = playlist["id"]
        else:
            playlist = {"id": playlists_cache[playlist_name]}

        for i in range(0, len(track_ids), 800):
            batch = track_ids[i:i+800]
            for attempt in range(3):  # retry loop
                try:
                    sp.playlist_add_items(playlist["id"], batch)
                    time.sleep(0.4)  
                    break
                except spotipy.exceptions.SpotifyException:
                    print("rate limit.")
                    time.sleep(5)

        print(f"Updated playlist: {playlist_name} ({len(track_ids)} songs)")

update_playlists(genre_groups)
update_playlists(lang_groups, prefix="Country: ")

processed_ids.update([t["id"] for t in all_tracks])
memory["processed_ids"] = list(processed_ids)
memory["playlists"] = playlists_cache

with open(MEMORY_PATH, "w") as f:
    json.dump(memory, f, indent=2)
