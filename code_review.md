# Code Review: app.py vs. playlist_logic.py

> Iterative inconsistency review — no code changes applied.

---

## Issue 1 — `random_choice_or_none` never returns `None` (crash risk)

**File:** [playlist_logic.py:192-196](playlist_logic.py#L192-L196)
**Severity:** High — runtime crash

The function is named `random_choice_or_none` and its return type is `Optional[Song]`, strongly
implying it should return `None` when the list is empty. Instead it calls `random.choice(songs)`
unconditionally, which raises `IndexError` on an empty list.

`app.py` at [app.py:308](app.py#L308) guards against this with:

```python
if pick is None:
    st.warning("No songs available for this mode.")
    return
```

That guard will **never trigger** — the app will crash instead whenever the selected mode has
zero songs.

**Recommended fix:** Add an early return inside `random_choice_or_none`:
```python
if not songs:
    return None
```

---

## Issue 2 — `compute_playlist_stats`: `total` is `len(hype)` instead of `len(all_songs)`

**File:** [playlist_logic.py:119-120](playlist_logic.py#L119-L120)
**Severity:** High — stat is always wrong

```python
total = len(hype)                              # Bug: should be len(all_songs)
hype_ratio = len(hype) / total if total > 0 else 0.0
```

`hype_ratio` divides hype count by hype count, so it is **always 1.0** whenever there is at
least one hype song — regardless of how many chill or mixed songs exist.

`app.py` at [app.py:335](app.py#L335) displays this as a meaningful "Hype ratio" metric, which
will always read `1.00`.

**Recommended fix:** Change `total = len(hype)` to `total = len(all_songs)`.

---

## Issue 3 — `compute_playlist_stats`: `avg_energy` sums only hype songs

**File:** [playlist_logic.py:123-125](playlist_logic.py#L123-L125)
**Severity:** High — stat is always wrong

```python
total_energy = sum(song.get("energy", 0) for song in hype)   # Bug: should be all_songs
avg_energy = total_energy / len(all_songs)
```

The numerator sums energy from `hype` only, while the denominator divides by `len(all_songs)`.
The result is skewed downward for any library that has chill or mixed songs.

`app.py` at [app.py:336](app.py#L336) displays this as "Average energy" across all playlists.

**Recommended fix:** Change the generator to iterate over `all_songs`.

---

## Issue 4 — `search_songs`: substring check is inverted

**File:** [playlist_logic.py:171](playlist_logic.py#L171)
**Severity:** High — search almost never works

```python
if value and value in q:   # Bug: checks if the field IS INSIDE the query string
```

This checks whether the song's field value is a substring of the search query — the reverse of
what the user intends. Typing `"AC/DC"` would only match if "ac/dc" happened to be a substring
of "AC/DC", which it isn't after lowercasing differences are accounted for.

`app.py` at [app.py:280](app.py#L280) passes the user's text input as `query`:

```python
filtered = search_songs(songs, query, field="artist")
```

A user typing an artist name will almost never see results.

**Recommended fix:** Flip the check to `q in value`.

---

## Issue 5 — `classify_song`: chill keyword check inspects `title` instead of `genre`

**File:** [playlist_logic.py:74](playlist_logic.py#L74)
**Severity:** Medium — inconsistent classification logic

```python
is_hype_keyword = any(k in genre for k in hype_keywords)   # checks genre ✓
is_chill_keyword = any(k in title for k in chill_keywords)  # checks title ✗
```

`chill_keywords = ["lofi", "ambient", "sleep"]` are genre names, yet they are matched against
the song's `title`. A song with `genre = "ambient"` and `title = "Forest Walk"` will **not**
be caught by this check. A song with `genre = "rock"` and `title = "Sleep Mode"` (lowercase
"sleep") **would** be incorrectly caught.

This is internally inconsistent and is likely why some songs with ambient/lofi genres land in
Mixed instead of Chill when their energy is in the 4–6 range.

**Recommended fix:** Change `title` to `genre` on line 74.

---

## Issue 6 — `profile_sidebar`: sliders use `st.sidebar` inside column context managers

**File:** [app.py:197-211](app.py#L197-L211)
**Severity:** Medium — layout bug

```python
col1, col2 = st.sidebar.columns(2)
with col1:
    profile["hype_min_energy"] = st.sidebar.slider(...)   # uses sidebar, not col1
with col2:
    profile["chill_max_energy"] = st.sidebar.slider(...)  # uses sidebar, not col2
```

The columns are created, but then `st.sidebar.slider()` is called instead of `col1.slider()` /
`col2.slider()`. Streamlit ignores the column context when the widget is attached to a different
container. Both sliders render outside the columns in the main sidebar flow.

**Recommended fix:** Replace `st.sidebar.slider(...)` with `col1.slider(...)` and
`col2.slider(...)` respectively inside their context managers.

---

## Issue 7 — `profile_sidebar`: `favorite_genre` selectbox hardcodes `index=0`

**File:** [app.py:213-217](app.py#L213-L217)
**Severity:** Medium — profile value is silently overwritten every render

```python
profile["favorite_genre"] = st.sidebar.selectbox(
    "Favorite genre",
    options=["rock", "lofi", "pop", "jazz", "electronic", "ambient", "other"],
    index=0,           # always "rock", ignores saved profile value
)
```

Every Streamlit re-render sets `favorite_genre` back to `"rock"` (index 0) regardless of what
the user selected. The profile value is effectively never preserved across interactions.

**Recommended fix:** Compute the index from the saved value:
```python
options = ["rock", "lofi", "pop", "jazz", "electronic", "ambient", "other"]
saved = profile.get("favorite_genre", "rock")
idx = options.index(saved) if saved in options else 0
profile["favorite_genre"] = st.sidebar.selectbox("Favorite genre", options=options, index=idx)
```

---

## Issue 8 — `lucky_pick`: "any" mode excludes Mixed songs

**File:** [playlist_logic.py:185-188](playlist_logic.py#L185-L188)
**Severity:** Low–Medium — silent data exclusion

```python
else:
    songs = playlists.get("Hype", []) + playlists.get("Chill", [])
```

When `mode = "any"`, Mixed playlist songs are silently excluded. Depending on the profile
thresholds, a large portion of the library may sit in Mixed and never be reachable via lucky
pick.

`app.py` at [app.py:300-307](app.py#L300-L307) offers `"any"` as a mode option with no
indication that Mixed songs are excluded.

**Recommended fix:** Include Mixed: `playlists.get("Hype", []) + playlists.get("Chill", []) + playlists.get("Mixed", [])`.

---

## Issue 9 — `normalize_artist` lowercases artist names, causing display inconsistency

**File:** [playlist_logic.py:22-26](playlist_logic.py#L22-L26) vs. [app.py:290-292](app.py#L290-L292)
**Severity:** Low — cosmetic but confusing

`normalize_artist` lowercases the artist name. `normalize_title` does **not** lowercase the
title. After `normalize_song` runs (called in `build_playlists` and `add_song_sidebar`), every
song's artist is stored in lowercase. The UI then renders:

```
- **Bohemian Rhapsody** by queen (genre rock, energy 8, mood Hype) [classic, opera]
```

instead of `Queen`. This affects all artists in every playlist tab, the lucky pick result, and
the history view.

**Recommended fix:** Remove the `.lower()` call from `normalize_artist` and handle
case-insensitive comparison at the point of comparison (e.g., in `search_songs` and
`most_common_artist`).

---

## Issue 10 — `merge_playlists` is called with an empty dict (no-op)

**File:** [app.py:395](app.py#L395)
**Severity:** Low — dead code / unfinished feature

```python
merged_playlists = merge_playlists(base_playlists, {})
```

Merging with `{}` produces a result identical to `base_playlists`. The `merge_playlists`
function is designed to combine two playlist maps (e.g., a local library with a curated set),
but the second argument is never populated.

This is either dead code or an incomplete feature stub. The variable `merged_playlists` is used
throughout `main()`, so removing or replacing this call would require attention.

**Recommended fix:** Either pass a meaningful second playlist, or replace the line with
`merged_playlists = base_playlists` and remove `merge_playlists` from the import if unused.

---

## Summary Table

| # | Location | Description | Severity |
|---|----------|-------------|----------|
| 1 | `playlist_logic.py:196` | `random_choice_or_none` crashes on empty list instead of returning `None` | **High** |
| 2 | `playlist_logic.py:119` | `hype_ratio` divides hype by hype (always 1.0) | **High** |
| 3 | `playlist_logic.py:124` | `avg_energy` sums hype energy only, divided by all songs | **High** |
| 4 | `playlist_logic.py:171` | Search check is inverted (`value in q` vs. `q in value`) | **High** |
| 5 | `playlist_logic.py:74` | Chill keyword matched against `title` instead of `genre` | **Medium** |
| 6 | `app.py:199,206` | Sliders use `st.sidebar` inside column contexts instead of `col1`/`col2` | **Medium** |
| 7 | `app.py:213` | `favorite_genre` selectbox hardcodes `index=0`, resetting on every render | **Medium** |
| 8 | `playlist_logic.py:187` | "Any" mode in lucky pick excludes Mixed playlist | **Medium** |
| 9 | `playlist_logic.py:26` | Artist lowercased at storage; displays as lowercase in all UI views | **Low** |
| 10 | `app.py:395` | `merge_playlists` called with `{}` — no-op, likely unfinished feature | **Low** |
