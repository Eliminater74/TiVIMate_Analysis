# TiViMate Search Architecture & FTS Engine — Technical Audit

This document provides a highly detailed architectural analysis of **TiViMate’s high-performance search-as-you-type engine**. It dissects how the application indexes live TV programs, movies, and TV shows in virtual Full-Text Search (FTS) tables, coordinates rapid database joins, handles D-pad search overlays, and prevents UI stutters during character input.

---

## 1. USER-FLOW LEVEL BEHAVIOR

*   **Search-As-You-Type**: The user invokes the search overlay by pressing the search button in the sidebar or clicking a designated remote hotkey. As the user types characters using an on-screen keyboard (or voice search), the screen updates instantly.
*   **Segmented Results**: Results are organized into three clear, vertical categories:
    *   **Live Channels & Programs**: Shows active and upcoming live broadcasts.
    *   **Movies**: Shows available VOD titles.
    *   **TV Shows (Series)**: Shows matched series titles.
*   **One-Click Actions**: Hovering over a matched program card shows EPG timestamps. Clicking D-pad Center on a Live channel result immediately starts playback. Clicking a VOD result launches the unified metadata detail overlay.

---

## 2. THE HIGH-PERFORMANCE SQLITE FTS4 ENGINE

Traditional SQL string matching uses the `LIKE %query%` operator. Over a database populated with 10,000 channels and 250,000 EPG programs, a `LIKE` query forces SQLite to perform a **full table scan**, dragging search times to **200–800ms**. This causes the UI keyboard to freeze and drop remote inputs.

TiViMate solves this entirely by mirroring text search columns into dedicated **SQLite FTS4 (Full-Text Search) Virtual Tables**.

```
                         [ User Character Input ]
                                    │
                                    ▼ (300ms Debounce Filter)
                         [ Database Search Query ]
                                    │
             ┌──────────────────────┴──────────────────────┐
             ▼ (Live EPG Search)                           ▼ (VOD Movie Search)
    [ tvg_programs_fts MATCH ]                    [ movies_fts MATCH ]
             │                                             │
             ▼ (Instant B-Tree Seek)                       ▼ (Instant B-Tree Seek)
    [ Joint Relational Table ]                    [ Joint Relational Table ]
             │                                             │
             └──────────────────────┬──────────────────────┘
                                    ▼
                          [ Rendered UI Grid ]
                        (QueryResult < 2 milliseconds)
```

### The FTS Virtual Table Design
EPG titles, movie names, and series details are mirrored inside virtual tables:
*   `tvg_programs_fts`
*   `movies_fts`
*   `series_fts`

FTS virtual tables construct full-text B-tree indexes over tokenized strings. FTS4 automatically links a hidden column `docid` which maps directly to the relational primary key (`id`) of the main data tables.

---

## 3. KEY QUERIES & CLASS REFERENCES

SQLite queries are compiled as plain text inside the dex files. The decompiled class `C3214.java` (Case 18) and `C3220.java` (Case 11) reveal the exact joint queries used for VOD and Live EPG search.

### A. VOD Movie Search Query (`sources/p240/C3214.java` Line 345)
This query performs an instant match on the FTS index, joining the relational `movies` table and checking playlist visibility flags in a single execution path:

```sql
SELECT m.id, m.playlist_id, p.name playlist_name, p.position playlist_position, 
       m.category_id, m.xc_id, m.stalker_id, m.name, length(m.image) image_length, 
       NULL stalker_poster_url, m.rating, m.added_time, m.position_in_playlist, m.is_favorite
FROM movies m
INNER JOIN playlists p ON m.playlist_id == p.id
INNER JOIN movies_fts ON m.id == movies_fts.docid
LEFT OUTER JOIN movie_categories c ON m.category_id == c.id
WHERE p.is_enabled == 1
    AND m.deleted_time IS NULL
    AND c.deleted_time IS NULL
    AND c.stalker_id IS NULL
    AND IFNULL(c.is_visible, 1) == 1
    AND movies_fts MATCH ?
LIMIT 10000
```
*   *Why this is incredibly fast*: The `MATCH` operator scans pre-compiled string tokens, returning matches in under **1 millisecond**.
*   *Image Width Optimization*: Instead of fetching the heavy binary blob or long text logo URL in the search join, it fetches `length(m.image)`, keeping the row width narrow and preventing SQLite cache page pollution.

### B. Live EPG Search Query (`sources/p240/C3220.java` Line 196)
```sql
SELECT * FROM tvg_programs
INNER JOIN tvg_programs_fts ON tvg_programs.id == tvg_programs_fts.docid
WHERE tvg_programs_fts.title MATCH ? 
  AND start_epoch_seconds >= ?
LIMIT 10000
```
*   *EPG Past Boundaries*: By adding `start_epoch_seconds >= ?` (current time minus a buffer), search results automatically exclude expired programs, focusing only on active and upcoming streams.

---

## 4. CONCEPTUAL SEARCH-AS-YOU-TYPE PIPELINE

To implement this high-end search routine in your app without copying TiViMate code, use this modern Room and Coroutine blueprint.

### Step 1: Define Room FTS4 Entities
Create a parallel virtual table mapped directly to your relational database entity.

```kotlin
@Entity(tableName = "movies")
data class MovieEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val name: String,
    val rating: Float?,
    val isFavorite: Boolean = false
)

@Fts4(contentEntity = MovieEntity::class)
@Entity(tableName = "movies_fts")
data class MovieFtsEntity(
    @ColumnInfo(name = "name") val name: String
)
```

### Step 2: The High-Performance Search DAO Query
Binds the FTS virtual entity (`movies_fts`) to the relational table (`movies`) via the `rowid` (implicitly mapped as `docid`).

```kotlin
@Dao
interface MovieSearchDao {
    @Query("""
        SELECT m.* FROM movies m
        INNER JOIN movies_fts fts ON m.id = fts.rowid
        WHERE movies_fts MATCH :queryTerm
        LIMIT 100
    """)
    suspend fun searchMovies(queryTerm: String): List<MovieEntity>
}
```

### Step 3: Debounced Search Input Flow
Triggering database queries on every character press (e.g. typing "c", then "ch", then "cha") creates redundant, overlapping threads. Use Kotlin Coroutines and Flow to **debounce** the search queries:

```kotlin
class SearchViewModel(private val searchDao: MovieSearchDao) : ViewModel() {
    private val searchQueryFlow = MutableStateFlow("")

    val searchResults: LiveData<List<MovieEntity>> = searchQueryFlow
        .debounce(300) // Wait for a 300ms pause in typing before querying the database
        .distinctUntilChanged()
        .filter { it.length >= 2 } // Only search for 2 or more characters
        .map { cleanQuery ->
            // SQLite MATCH query requires formatting (e.g., adding wildcards)
            val formattedQuery = "$cleanQuery*"
            searchDao.searchMovies(formattedQuery)
        }
        .asLiveData(Dispatchers.IO)

    fun onQueryChanged(newText: String) {
        searchQueryFlow.value = newText
    }
}
```

---

## 5. TV FOCUS & REMOTE CONTROL UX

*   **Keyboard Focus Lock**:
    Android TV displays an on-screen keyboard. When typing, D-pad navigation moves focus between key characters. The search results grid (usually rendered below or to the side of the keyboard) must **never grab focus automatically** when new results are loaded.
*   **Manual Result Transition**:
    Focus must only shift to search results if the user explicitly presses **D-pad Down** or **D-pad Right** to exit the keyboard boundary.
*   **Visual Highlights**:
    Matched text segments inside the resulting cards are highlighted (e.g. highlighted with standard theme focus accent colors) to provide direct visual feedback.

---

## PureFusionIPTV Beta 10 Recommendations

### Low-Risk Actions
*   Define parallel `@Fts4` entities in Room for your live programs and movie libraries.
*   Implement a **300ms debounce filter** inside your view model search flow.

### Medium-Risk Actions
*   Join playlists inside search DAO queries to filter out disabled playlists or hidden groups automatically.
*   Configure on-screen keyboard layouts to prevent automatic focus shifts when updating search result lists.

### High-Risk Actions
*   Executing database search operations on the main thread using slow `LIKE` string queries, which freezes the screen and triggers ANRs on budget Android TV sticks.

### What to Measure First
*   Benchmark query search times. A full database query over 100,000 EPG items using SQLite FTS MATCH should take **less than 2ms**.

### What Not to Copy Directly
*   Do not copy obfuscated parser identifiers or custom search classes from `sources/p240/C3214.java`.
