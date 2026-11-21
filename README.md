# VibeFinder

VibeFinder recommends songs from a 20-track classroom catalog and explains how each track scored against a listener's preferences. Project 4 adds a reliability mode that filters weak matches, checks the recommendations, and returns fewer tracks when the catalog cannot fill the request.

## Base project

The original Module 3 project, [Music Recommender Simulation (VibeFinder)](https://github.com/Somama12/ai110-module3show-musicrecommendersimulation-starter), ranked tracks using weighted comparisons of genre, mood, energy, and other attributes. It provided pop, lofi, and EDM profiles, displayed the contribution of each feature, and let users vary the ranking weights and artist repetition penalty.

The Module 3 implementation was built from the [CodePath starter](https://github.com/codepath/ai110-module3show-musicrecommendersimulation-starter). It preserved the starter's 10 song rows and object interfaces, then added 10 illustrative tracks and five scoring attributes.

[Setup](#getting-started) · [Architecture](#architecture) · [Demo](#end-to-end-demo) · [Testing](#evaluation-results) · [Model card](model_card.md) · [Portfolio](PORTFOLIO.md)

## Getting Started

Use Python 3.10 or newer. The CLI and evaluation harness use the standard library. Install the requirements to run the pytest suite.

1. Clone the Project 4 repository.

   ```bash
   git clone https://github.com/Somama12/vibefinder-applied-ai-system.git
   cd vibefinder-applied-ai-system
   ```

2. Set up the environment and install the test dependency.

   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   python -m pip install -r requirements.txt
   ```

   On Windows PowerShell, use `py -m venv .venv` and activate with `.venv\Scripts\Activate.ps1`.

3. Run reliable mode and the automated checks.

   ```bash
   python -m src.main --profile pop --reliable --k 3
   python -m pytest -q
   python -m src.evaluate
   ```

Save custom preferences as JSON, for example `{"genre":"pop","mood":"happy","energy":0.8}`, and pass the file with `--prefs path/to/profile.json`. It overrides the named profile. Run `python -m src.main --help` for all options.

## How The System Works

Real recommenders use interaction data such as listens, saves, skips, and watch history to estimate preferences. Spotify describes using both a person's listening habits and those of people with similar tastes ([Spotify Help](https://support.spotify.com/sc-en/article/find-playlists/)). YouTube lists watch history, likes, dislikes, and feedback among its recommendation signals ([YouTube Help](https://support.google.com/youtube/answer/16089387?hl=en)). Collaborative filtering finds patterns across users' behavior; content-based filtering compares the attributes of items with a user's preferences. Real services can combine multiple approaches. This project uses a hand-written content-based rule: it does not train a model, access a streaming account, or use other listeners' data.

There are three separate parts: **input data** describes songs, **user preferences** specify desired attributes, and **ranking/selection** scores candidates and chooses the best available matches. Genre and mood describe categories; energy, tempo, acousticness and the other numeric features describe a song's character. A taste profile supplies target values, so a low-energy listener is rewarded for closeness to low energy, rather than receiving the most energetic song every time.

### Data and preferences

`data/songs.csv` contains 20 illustrative tracks. These are classroom data, not measured Spotify audio features or verified commercial releases. The original columns are `id`, `title`, `artist`, `genre`, `mood`, `energy`, `tempo_bpm`, `valence`, `danceability`, and `acousticness`. Five added attributes are `instrumentalness`, `popularity`, `release_year`, `duration_seconds`, and `live_performance`. All five are used in scoring when a profile supplies them. Their values are synthetic.

Dictionary profiles require `genre`, `mood`, and `energy`; other feature targets are optional. Missing optional targets do not contribute points. `src/profiles.py` provides pop/happy, lofi/chill, and EDM/intense examples. The starter `UserProfile` object retains `favorite_genre`, `favorite_mood`, `target_energy`, and `likes_acoustic`; the object adapter maps these into the same dictionary scorer.

### Algorithm recipe

For each supplied numeric feature:

```text
similarity = 1 - abs(song_value - user_target) / (maximum - minimum)
contribution = feature_weight * similarity
base_score = sum(all contributions, including category matches)
selection_score = base_score - 0.6 * previously_selected_songs_by_same_artist
```

All numeric inputs must be finite and inside the documented range; similarity therefore stays between 0 and 1. Genre/mood matching is case-insensitive exact matching. It does not treat “indie pop” as identical to “pop.”

| Feature | Valid range | Genre-first weight | Mood-first weight | Energy-first weight |
| --- | --- | ---: | ---: | ---: |
| Genre match | Nonempty text | 2 | 1 | 1 |
| Mood match | Nonempty text | 1 | 2 | 1 |
| Energy | 0–1 | 1 | 1 | 2 |
| Tempo | 40–220 BPM | 0.5 | 0.5 | 0.5 |
| Valence, danceability, acousticness | 0–1 each | 0.25 each | 0.25 each | 0.25 each |
| Instrumentalness | 0–1 | 0.25 | 0.25 | 0.25 |
| Popularity | 0–100 | 0.25 | 0.25 | 0.25 |
| Release year | 1950–2030 | 0.25 | 0.25 | 0.25 |
| Duration | 60–600 seconds | 0.25 | 0.25 | 0.25 |
| Live performance | Song: 0 or 1; target: 0–1 | 0.25 | 0.25 | 0.25 |

The maximum base score for a complete profile is 6.5 in each mode. Popularity is a target similarity, not an automatic reward for fame. A low popularity target supports discovery of less-popular synthetic tracks.

`score_song()` returns a number and feature-level reasons. `recommend_songs()` ranks the whole list and returns `(song, selection_score, explanation)` tuples. With artist diversity enabled, it selects the best candidate, updates artist counts, and re-ranks the remaining candidates. Ties use ascending song ID. Scores are kept at full precision for ranking and rounded only for display. `sorted()` creates a new list, whereas `.sort()` modifies the list in place; the catalog is not reordered or mutated.

The Strategy pattern separates weight configurations from scoring. `RankingStrategy` objects in `STRATEGIES` share the same scoring procedure, making modes easy to compare without copying the algorithm. The CLI's default artist penalty is 0.6; the function's default is 0 for straightforward baseline use.

## Module 3 Baseline Examples

<details>
<summary>Original pop, lofi, and EDM runs</summary>

These are captured outputs from the implemented CLI, including the actual score reasons. Each experiment requests five recommendations.

### POP profile

```bash
python -m src.main --profile pop
```

```text
Loaded songs: 20
Profile: pop | mode: genre-first
#   Song / Artist                                 Score  Reasons
-------------------------------------------------------------------------------------------------------------------
1   Golden Hour Ride / Daybreak Club              6.466  genre match (pop) (+2.000); mood match (happy) (+1.000);
                                                         energy 0.79 vs target 0.8 (+0.990); tempo_bpm 121 vs
                                                         target 120 (+0.497); valence 0.86 vs target 0.85 (+0.247);
                                                         danceability 0.81 vs target 0.8 (+0.247); acousticness
                                                         0.21 vs target 0.2 (+0.247); instrumentalness 0.04 vs
                                                         target 0.05 (+0.247); popularity 76 vs target 80 (+0.240);
                                                         release_year 2022 vs target 2022 (+0.250);
                                                         duration_seconds 208 vs target 210 (+0.249);
                                                         live_performance 0 vs target 0 (+0.250); artist repeat
                                                         penalty (-0.000); base 6.466

2   Sunrise City / Neon Echo                      6.459  genre match (pop) (+2.000); mood match (happy) (+1.000);
                                                         energy 0.82 vs target 0.8 (+0.980); tempo_bpm 118 vs
                                                         target 120 (+0.494); valence 0.84 vs target 0.85 (+0.247);
                                                         danceability 0.79 vs target 0.8 (+0.247); acousticness
                                                         0.18 vs target 0.2 (+0.245); instrumentalness 0.05 vs
                                                         target 0.05 (+0.250); popularity 82 vs target 80 (+0.245);
                                                         release_year 2022 vs target 2022 (+0.250);
                                                         duration_seconds 210 vs target 210 (+0.250);
                                                         live_performance 0 vs target 0 (+0.250); artist repeat
                                                         penalty (-0.000); base 6.459

3   Gym Hero / Max Pulse                          5.231  genre match (pop) (+2.000); mood mismatch (intense)
                                                         (+0.000); energy 0.93 vs target 0.8 (+0.870); tempo_bpm
                                                         132 vs target 120 (+0.467); valence 0.77 vs target 0.85
                                                         (+0.230); danceability 0.88 vs target 0.8 (+0.230);
                                                         acousticness 0.05 vs target 0.2 (+0.212); instrumentalness
                                                         0.02 vs target 0.05 (+0.242); popularity 85 vs target 80
                                                         (+0.237); release_year 2023 vs target 2022 (+0.247);
                                                         duration_seconds 200 vs target 210 (+0.245);
                                                         live_performance 0 vs target 0 (+0.250); artist repeat
                                                         penalty (-0.000); base 5.231

4   Rooftop Lights / Indigo Parade                4.328  genre mismatch (indie pop) (+0.000); mood match (happy)
                                                         (+1.000); energy 0.76 vs target 0.8 (+0.960); tempo_bpm
                                                         124 vs target 120 (+0.489); valence 0.81 vs target 0.85
                                                         (+0.240); danceability 0.82 vs target 0.8 (+0.245);
                                                         acousticness 0.35 vs target 0.2 (+0.213); instrumentalness
                                                         0.1 vs target 0.05 (+0.237); popularity 60 vs target 80
                                                         (+0.200); release_year 2021 vs target 2022 (+0.247);
                                                         duration_seconds 215 vs target 210 (+0.248);
                                                         live_performance 0 vs target 0 (+0.250); artist repeat
                                                         penalty (-0.000); base 4.328

5   City Notebook / Verse Atlas                   3.183  genre mismatch (hip-hop) (+0.000); mood mismatch (focused)
                                                         (+0.000); energy 0.7 vs target 0.8 (+0.900); tempo_bpm 94
                                                         vs target 120 (+0.428); valence 0.55 vs target 0.85
                                                         (+0.175); danceability 0.85 vs target 0.8 (+0.238);
                                                         acousticness 0.2 vs target 0.2 (+0.250); instrumentalness
                                                         0.02 vs target 0.05 (+0.242); popularity 61 vs target 80
                                                         (+0.203); release_year 2022 vs target 2022 (+0.250);
                                                         duration_seconds 205 vs target 210 (+0.248);
                                                         live_performance 0 vs target 0 (+0.250); artist repeat
                                                         penalty (-0.000); base 3.183
```

### LOFI profile

```bash
python -m src.main --profile lofi
```

```text
Loaded songs: 20
Profile: lofi | mode: genre-first
#   Song / Artist                                 Score  Reasons
-------------------------------------------------------------------------------------------------------------------
1   Paper Moon Loop / Quiet Current               6.466  genre match (lofi) (+2.000); mood match (chill) (+1.000);
                                                         energy 0.36 vs target 0.35 (+0.990); tempo_bpm 74 vs
                                                         target 75 (+0.497); valence 0.54 vs target 0.55 (+0.247);
                                                         danceability 0.51 vs target 0.5 (+0.247); acousticness
                                                         0.84 vs target 0.85 (+0.247); instrumentalness 0.93 vs
                                                         target 0.9 (+0.242); popularity 28 vs target 30 (+0.245);
                                                         release_year 2020 vs target 2020 (+0.250);
                                                         duration_seconds 182 vs target 180 (+0.249);
                                                         live_performance 0 vs target 0 (+0.250); artist repeat
                                                         penalty (-0.000); base 6.466

2   Library Rain / Paper Lanterns                 6.412  genre match (lofi) (+2.000); mood match (chill) (+1.000);
                                                         energy 0.35 vs target 0.35 (+1.000); tempo_bpm 72 vs
                                                         target 75 (+0.492); valence 0.6 vs target 0.55 (+0.238);
                                                         danceability 0.58 vs target 0.5 (+0.230); acousticness
                                                         0.86 vs target 0.85 (+0.247); instrumentalness 0.98 vs
                                                         target 0.9 (+0.230); popularity 24 vs target 30 (+0.235);
                                                         release_year 2021 vs target 2020 (+0.247);
                                                         duration_seconds 195 vs target 180 (+0.243);
                                                         live_performance 0 vs target 0 (+0.250); artist repeat
                                                         penalty (-0.000); base 6.412

3   Midnight Coding / LoRoom                      6.337  genre match (lofi) (+2.000); mood match (chill) (+1.000);
                                                         energy 0.42 vs target 0.35 (+0.930); tempo_bpm 78 vs
                                                         target 75 (+0.492); valence 0.56 vs target 0.55 (+0.247);
                                                         danceability 0.62 vs target 0.5 (+0.220); acousticness
                                                         0.71 vs target 0.85 (+0.215); instrumentalness 0.95 vs
                                                         target 0.9 (+0.238); popularity 32 vs target 30 (+0.245);
                                                         release_year 2020 vs target 2020 (+0.250);
                                                         duration_seconds 180 vs target 180 (+0.250);
                                                         live_performance 0 vs target 0 (+0.250); artist repeat
                                                         penalty (-0.000); base 6.337

4   Focus Flow / LoRoom                           4.758  genre match (lofi) (+2.000); mood mismatch (focused)
                                                         (+0.000); energy 0.4 vs target 0.35 (+0.950); tempo_bpm 80
                                                         vs target 75 (+0.486); valence 0.59 vs target 0.55
                                                         (+0.240); danceability 0.6 vs target 0.5 (+0.225);
                                                         acousticness 0.78 vs target 0.85 (+0.233);
                                                         instrumentalness 0.96 vs target 0.9 (+0.235); popularity
                                                         30 vs target 30 (+0.250); release_year 2022 vs target 2020
                                                         (+0.244); duration_seconds 170 vs target 180 (+0.245);
                                                         live_performance 0 vs target 0 (+0.250); artist repeat
                                                         penalty (-0.600); base 5.358

5   Sunday Strings / Cedar Lane                   4.290  genre mismatch (acoustic) (+0.000); mood match (chill)
                                                         (+1.000); energy 0.3 vs target 0.35 (+0.950); tempo_bpm 74
                                                         vs target 75 (+0.497); valence 0.58 vs target 0.55
                                                         (+0.243); danceability 0.4 vs target 0.5 (+0.225);
                                                         acousticness 0.98 vs target 0.85 (+0.217);
                                                         instrumentalness 0.6 vs target 0.9 (+0.175); popularity 25
                                                         vs target 30 (+0.237); release_year 2020 vs target 2020
                                                         (+0.250); duration_seconds 190 vs target 180 (+0.245);
                                                         live_performance 0 vs target 0 (+0.250); artist repeat
                                                         penalty (-0.000); base 4.290
```

### EDM profile

```bash
python -m src.main --profile edm
```

```text
Loaded songs: 20
Profile: edm | mode: genre-first
#   Song / Artist                                 Score  Reasons
-------------------------------------------------------------------------------------------------------------------
1   Electric Horizon / Pulse Arcade               6.470  genre match (edm) (+2.000); mood match (intense) (+1.000);
                                                         energy 0.96 vs target 0.95 (+0.990); tempo_bpm 145 vs
                                                         target 145 (+0.500); valence 0.65 vs target 0.65 (+0.250);
                                                         danceability 0.96 vs target 0.95 (+0.247); acousticness
                                                         0.03 vs target 0.05 (+0.245); instrumentalness 0.72 vs
                                                         target 0.7 (+0.245); popularity 68 vs target 65 (+0.242);
                                                         release_year 2023 vs target 2023 (+0.250);
                                                         duration_seconds 240 vs target 240 (+0.250);
                                                         live_performance 1 vs target 1 (+0.250); artist repeat
                                                         penalty (-0.000); base 6.470

2   Bassline Orbit / Signal Bloom                 6.138  genre match (edm) (+2.000); mood match (intense) (+1.000);
                                                         energy 0.92 vs target 0.95 (+0.970); tempo_bpm 150 vs
                                                         target 145 (+0.486); valence 0.62 vs target 0.65 (+0.242);
                                                         danceability 0.91 vs target 0.95 (+0.240); acousticness
                                                         0.06 vs target 0.05 (+0.247); instrumentalness 0.8 vs
                                                         target 0.7 (+0.225); popularity 58 vs target 65 (+0.232);
                                                         release_year 2024 vs target 2023 (+0.247);
                                                         duration_seconds 235 vs target 240 (+0.248);
                                                         live_performance 0 vs target 1 (+0.000); artist repeat
                                                         penalty (-0.000); base 6.138

3   Festival Afterglow / Pulse Arcade             5.842  genre match (edm) (+2.000); mood match (intense) (+1.000);
                                                         energy 0.93 vs target 0.95 (+0.980); tempo_bpm 142 vs
                                                         target 145 (+0.492); valence 0.68 vs target 0.65 (+0.242);
                                                         danceability 0.94 vs target 0.95 (+0.247); acousticness
                                                         0.04 vs target 0.05 (+0.247); instrumentalness 0.75 vs
                                                         target 0.7 (+0.237); popularity 66 vs target 65 (+0.247);
                                                         release_year 2023 vs target 2023 (+0.250);
                                                         duration_seconds 245 vs target 240 (+0.248);
                                                         live_performance 1 vs target 1 (+0.250); artist repeat
                                                         penalty (-0.600); base 6.442

4   Storm Runner / Voltline                       4.120  genre mismatch (rock) (+0.000); mood match (intense)
                                                         (+1.000); energy 0.91 vs target 0.95 (+0.960); tempo_bpm
                                                         152 vs target 145 (+0.481); valence 0.48 vs target 0.65
                                                         (+0.207); danceability 0.66 vs target 0.95 (+0.178);
                                                         acousticness 0.1 vs target 0.05 (+0.237); instrumentalness
                                                         0.08 vs target 0.7 (+0.095); popularity 72 vs target 65
                                                         (+0.232); release_year 2018 vs target 2023 (+0.234);
                                                         duration_seconds 230 vs target 240 (+0.245);
                                                         live_performance 1 vs target 1 (+0.250); artist repeat
                                                         penalty (-0.000); base 4.120

5   Gym Hero / Max Pulse                          3.908  genre mismatch (pop) (+0.000); mood match (intense)
                                                         (+1.000); energy 0.93 vs target 0.95 (+0.980); tempo_bpm
                                                         132 vs target 145 (+0.464); valence 0.77 vs target 0.65
                                                         (+0.220); danceability 0.88 vs target 0.95 (+0.233);
                                                         acousticness 0.05 vs target 0.05 (+0.250);
                                                         instrumentalness 0.02 vs target 0.7 (+0.080); popularity
                                                         85 vs target 65 (+0.200); release_year 2023 vs target 2023
                                                         (+0.250); duration_seconds 200 vs target 240 (+0.231);
                                                         live_performance 0 vs target 1 (+0.000); artist repeat
                                                         penalty (-0.000); base 3.908
```

</details>

## Ranking Experiments

All comparisons use the same catalog and a 0.6 artist penalty unless stated otherwise.

| Experiment | Before | After | Interpretation |
| --- | --- | --- | --- |
| Pop: genre-first → mood-first | Gym Hero is third (5.231) | Rooftop Lights is third (5.328) | The happy mood becomes more important than exact pop membership. |
| Pop: halve genre weight and double energy weight | Gym Hero is third (5.231) | Rooftop Lights is third (5.288) | Energy 0.76 is closer to the target 0.8 than 0.93. This is the `energy-first` mode. |
| EDM: artist penalty 0 → 0.6 | Electric Horizon, Festival Afterglow | Electric Horizon, Bassline Orbit | Top-two artist count rises from one to two. Festival Afterglow moves third and loses 0.6 points. |

The modes change an explicit design tradeoff; a changed list alone does not prove greater accuracy. The evaluation checks expected behavior, not real listener satisfaction.

## Project 4: Applied AI System Extension

The base is the Project 3 weighted recommender above. The substantial extension is an integrated **reliability mechanism**, one of the Project 4 feature options. It is a deterministic pipeline around a simulated AI recommender, not an LLM, RAG system, fine-tune, or autonomous runtime agent.

With `--reliable`, the system validates the request and catalog, checks whether the requested genre exists, removes songs whose normalized base score is below `--min-match` (default 0.70), ranks eligible songs with the artist penalty, and audits membership, duplicate IDs, scores, order, and explanations. It returns `ok`, `partial`, or `abstained`. It never pads the list with weak songs merely to reach k. An optional JSON trace records factual processing events, counts, selected IDs and scores.

The match fraction is `base_score / maximum_score_for_supplied_features`. It is a heuristic, **not a calibrated probability or confidence estimate**. The gate is applied before diversity; the displayed selection score can be lower after an artist penalty. The coverage check requires the requested genre to exist somewhere in the catalog, but eligible recommendations can include other genres.

### Architecture

The [architecture diagram](diagrams/architecture.mmd) shows how the CLI loads preferences and the song catalog, then selects the baseline or reliable workflow. The reliable workflow checks catalog coverage, applies the match threshold before ranking, and audits the ranked output. The evaluation harness calls both workflows directly and reports which behavior checks pass.

```mermaid
flowchart TD
    subgraph CLI["Command-line app · src.main"]
        Input["CLI options and built-in or JSON preferences"] --> Read["Parse options and read preferences"]
        Read --> Load["Load and validate catalog CSV"]
        Catalog["Default or custom catalog CSV"] --> Load
        Load --> Mode{"Reliable mode enabled?"}
        Report["Print status and message"]
        Report --> Trace["Write JSON audit events if requested"]
        Trace --> Table["Print recommendation table when rows exist"]
        Error["Print error and exit with code 2"]
    end

    subgraph Engine["Recommendation engine"]
        Baseline["Validate request, score all songs, and rank with artist repeat penalty"]
        Validate["Validate preferences, options, and catalog"]
        Validate --> Coverage{"Requested genre exists in the catalog?"}
        Coverage -->|No| Abstain["Return abstained with zero rows"]
        Coverage -->|Yes| Gate["Score all songs and keep matches at or above the threshold"]
        Gate --> Rank["Score and rank eligible songs with the same ranking function"]
        Rank --> Audit["Audit catalog membership, count, unique IDs, scores, order, threshold, and explanations"]
        Audit --> Outcome["Return ok, partial, or abstained based on row count"]
    end

    Mode -->|No| Baseline
    Mode -->|Yes| Validate
    Baseline --> Table
    Outcome --> Report
    Abstain --> Report
    Read -->|Invalid input| Error
    Load -->|Invalid or unreadable catalog| Error
    Baseline -->|Validation fails| Error
    Validate -->|Validation fails| Error
    Audit -->|Audit fails| Error
    Trace -->|Write fails| Error

    subgraph Evaluation["Separate evaluation harness · src.evaluate"]
        Cases["Load default catalog and run nine predefined scenarios"]
        Cases --> Results["Check results or expected errors and print pass/fail summary"]
    end
    Cases -. calls baseline ranking .-> Baseline
    Cases -. calls reliable pipeline .-> Validate
```

## End-to-end demo

The following outputs were captured by running the CLI against the committed catalog.

### Full result for three pop recommendations

The built-in pop profile requests happy tracks with an energy target of 0.8. All three requested slots have eligible matches.

```bash
python -m src.main --profile pop --reliable --k 3
```

```text
Loaded songs: 20
Profile: pop | mode: genre-first
Status: ok | 3 of 3 requested songs meet match >= 0.70.
#   Song / Artist                                 Score  Reasons
-------------------------------------------------------------------------------------------------------------------
1   Golden Hour Ride / Daybreak Club              6.466  genre match (pop) (+2.000); mood match (happy) (+1.000);
                                                         energy 0.79 vs target 0.8 (+0.990); tempo_bpm 121 vs
                                                         target 120 (+0.497); valence 0.86 vs target 0.85 (+0.247);
                                                         danceability 0.81 vs target 0.8 (+0.247); acousticness
                                                         0.21 vs target 0.2 (+0.247); instrumentalness 0.04 vs
                                                         target 0.05 (+0.247); popularity 76 vs target 80 (+0.240);
                                                         release_year 2022 vs target 2022 (+0.250);
                                                         duration_seconds 208 vs target 210 (+0.249);
                                                         live_performance 0 vs target 0 (+0.250); artist repeat
                                                         penalty (-0.000); base 6.466

2   Sunrise City / Neon Echo                      6.459  genre match (pop) (+2.000); mood match (happy) (+1.000);
                                                         energy 0.82 vs target 0.8 (+0.980); tempo_bpm 118 vs
                                                         target 120 (+0.494); valence 0.84 vs target 0.85 (+0.247);
                                                         danceability 0.79 vs target 0.8 (+0.247); acousticness
                                                         0.18 vs target 0.2 (+0.245); instrumentalness 0.05 vs
                                                         target 0.05 (+0.250); popularity 82 vs target 80 (+0.245);
                                                         release_year 2022 vs target 2022 (+0.250);
                                                         duration_seconds 210 vs target 210 (+0.250);
                                                         live_performance 0 vs target 0 (+0.250); artist repeat
                                                         penalty (-0.000); base 6.459

3   Gym Hero / Max Pulse                          5.231  genre match (pop) (+2.000); mood mismatch (intense)
                                                         (+0.000); energy 0.93 vs target 0.8 (+0.870); tempo_bpm
                                                         132 vs target 120 (+0.467); valence 0.77 vs target 0.85
                                                         (+0.230); danceability 0.88 vs target 0.8 (+0.230);
                                                         acousticness 0.05 vs target 0.2 (+0.212); instrumentalness
                                                         0.02 vs target 0.05 (+0.242); popularity 85 vs target 80
                                                         (+0.237); release_year 2023 vs target 2022 (+0.247);
                                                         duration_seconds 200 vs target 210 (+0.245);
                                                         live_performance 0 vs target 0 (+0.250); artist repeat
                                                         penalty (-0.000); base 5.231
```

### Partial result for twenty pop recommendations

The same profile requests 20 tracks. The match gate withholds the 17 tracks below 0.70.

```bash
python -m src.main --profile pop --reliable --k 20
```

```text
Loaded songs: 20
Profile: pop | mode: genre-first
Status: partial | 3 of 20 requested songs meet match >= 0.70.
#   Song / Artist                                 Score  Reasons
-------------------------------------------------------------------------------------------------------------------
1   Golden Hour Ride / Daybreak Club              6.466  genre match (pop) (+2.000); mood match (happy) (+1.000);
                                                         energy 0.79 vs target 0.8 (+0.990); tempo_bpm 121 vs
                                                         target 120 (+0.497); valence 0.86 vs target 0.85 (+0.247);
                                                         danceability 0.81 vs target 0.8 (+0.247); acousticness
                                                         0.21 vs target 0.2 (+0.247); instrumentalness 0.04 vs
                                                         target 0.05 (+0.247); popularity 76 vs target 80 (+0.240);
                                                         release_year 2022 vs target 2022 (+0.250);
                                                         duration_seconds 208 vs target 210 (+0.249);
                                                         live_performance 0 vs target 0 (+0.250); artist repeat
                                                         penalty (-0.000); base 6.466

2   Sunrise City / Neon Echo                      6.459  genre match (pop) (+2.000); mood match (happy) (+1.000);
                                                         energy 0.82 vs target 0.8 (+0.980); tempo_bpm 118 vs
                                                         target 120 (+0.494); valence 0.84 vs target 0.85 (+0.247);
                                                         danceability 0.79 vs target 0.8 (+0.247); acousticness
                                                         0.18 vs target 0.2 (+0.245); instrumentalness 0.05 vs
                                                         target 0.05 (+0.250); popularity 82 vs target 80 (+0.245);
                                                         release_year 2022 vs target 2022 (+0.250);
                                                         duration_seconds 210 vs target 210 (+0.250);
                                                         live_performance 0 vs target 0 (+0.250); artist repeat
                                                         penalty (-0.000); base 6.459

3   Gym Hero / Max Pulse                          5.231  genre match (pop) (+2.000); mood mismatch (intense)
                                                         (+0.000); energy 0.93 vs target 0.8 (+0.870); tempo_bpm
                                                         132 vs target 120 (+0.467); valence 0.77 vs target 0.85
                                                         (+0.230); danceability 0.88 vs target 0.8 (+0.230);
                                                         acousticness 0.05 vs target 0.2 (+0.212); instrumentalness
                                                         0.02 vs target 0.05 (+0.242); popularity 85 vs target 80
                                                         (+0.237); release_year 2023 vs target 2022 (+0.247);
                                                         duration_seconds 200 vs target 210 (+0.245);
                                                         live_performance 0 vs target 0 (+0.250); artist repeat
                                                         penalty (-0.000); base 5.231
```

### Abstention for an unsupported genre

The [custom input](examples/unsupported.json) is `{"genre":"opera","mood":"dramatic","energy":0.5}`. The baseline returns five ranked tracks for this input. Reliable mode checks genre coverage and returns zero.

```bash
python -m src.main --prefs examples/unsupported.json --reliable
```

```text
Loaded songs: 20
Profile: unsupported.json | mode: genre-first
Status: abstained | Requested genre is not represented in the catalog.
```

### Invalid input guardrail

The [invalid profile](examples/invalid.json) sets energy to 1.4. Validation rejects it with exit code 2.

```bash
python -u -m src.main --prefs examples/invalid.json --reliable
```

```text
Loaded songs: 20
Profile: invalid.json | mode: genre-first
Error: energy must be between 0 and 1
```

To write a JSON audit log, add `--trace path/to/trace.json` to a reliable-mode command. For example, `python -m src.main --profile pop --reliable --k 3 --trace examples/pop_trace.json` reproduces the [pop trace](examples/pop_trace.json) with validation outcomes, candidate counts, and selected IDs and scores.

## Evaluation results

```bash
python -m src.evaluate
```

```text
PASS | pop: top genre and three reliable results
PASS | lofi: top genre and three reliable results
PASS | edm: top genre and three reliable results
PASS | unsupported genre: abstain
PASS | strict threshold: abstain
PASS | partial result: no weak padding
PASS | invalid energy: rejected
PASS | artist diversity: increases top-two unique artists
PASS | mode switch: changes top-three selection

Evaluation: 9/9 passed
```

The pytest suite passed **35 tests**. It covers the original object interface, malformed CSV, input bounds, deterministic ranking, and deliberately altered titles, scores, and explanations. The evaluation harness checks nine predefined scenarios and prints a pass/fail summary.

On the classroom catalog, the coverage check caught the baseline's behavior of returning tracks for an unavailable genre. The match threshold also produced a clear partial result when only three tracks qualified. The tests measure implementation behavior on these examples. Listener satisfaction would require feedback from real users.

## Limitations and Risks

The catalog is small, synthetic, English-labeled, and uneven across genres. Genre-first scoring can create a filter bubble, and exact labels miss relationships such as pop versus indie pop. Numeric mood features are subjective, popularity targets can still reinforce popularity bias, and the system has no lyrics, language, listening history, or real user feedback. Artist diversity improves exposure variety but does not demonstrate demographic fairness. The reliability threshold is hand-chosen, and strict coverage checks can withhold useful cross-genre discoveries.

## Reflection

My biggest takeaway is that the scoring rules shape what looks like a good recommendation. Gym Hero ranks third for the happy-pop profile even though its mood is intense, because the genre match carries more weight. Switching modes changes that result. The artist-diversity experiment makes another tradeoff visible: a slightly lower-scoring song can give a different artist a place near the top. These examples help explain why a recommendation score is not the same thing as knowing what someone will enjoy.

The [model card](model_card.md#9-personal-reflection) records my collaboration with AI and the tradeoffs I would revisit. The [AI collaboration log](ai_interactions.md) documents the development and verification work.

## Files and rubric evidence

| Requirement | Evidence |
| --- | --- |
| Project 3 explanation, data, scoring, ranking | This README; `data/songs.csv`; `src/recommender.py` |
| Project 4 execution evidence | [Full, partial, and abstained runs](#end-to-end-demo) |
| Model card | `model_card.md` |
| Five additional attributes and AI workflow | CSV, scoring weights, `ai_interactions.md` |
| Diversity, modes, formatted table | `recommend_songs`, `RankingStrategy`, `format_table` |
| Project 4 base, substantial extension, demo | Project 4 section; `src/reliability.py`; CLI examples |
| Architecture source | [Mermaid source](diagrams/architecture.mmd) |
| Guardrails and evaluation bonus | `tests/test_extended.py`; `src/evaluate.py` |
| AI collaboration and design reflection | `ai_interactions.md`; `model_card.md` |

The optional evaluation-harness stretch is implemented in `src/evaluate.py`.
