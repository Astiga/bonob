# Handle Missing Album/Artist Metadata Gracefully - Implementation Plan

## Overview

Bonob crashes with HTTP 500 errors when processing tracks that have missing album or artist metadata. The root cause is that the code assumes all tracks have albums and artists, using TypeScript non-null assertions and passing `undefined` values to Subsonic API calls like `getAlbum(credentials, undefined)`.

## Current State Analysis

**The Problem:**
1. `Track.album` and `Track.artist` are typed as required but can be missing at runtime
2. Non-null assertions (`!`) are used when constructing Album/Artist objects from Subsonic responses
3. When albumId is `undefined`, code still constructs an Album object with `id: undefined`
4. This undefined ID propagates to `getAlbum()` calls, causing HTTP 500 errors
5. The SMAPI artist conversion assumes artist.id always exists

**Example from ticket:**
- Database shows playlist entries with empty `album` and `artist` fields
- Bonob calls `/rest/getAlbum` with `{ id: undefined }`
- Subsonic returns HTTP 500 for missing required parameter

**Key Discoveries:**
- SMAPI layer already handles optional album/artist IDs with conditional checks (smapi.ts:358, 360, 363)
- `asTrackFromSearchResult()` uses `|| ''` pattern but doesn't call getAlbum (subsonic.ts:328-329)
- `ArtistSummary.id` is currently `string | undefined` but should be required if artist exists

## Desired End State

**Type System:**
- If an album exists, it must have an ID: `AlbumSummary.id: string` (required)
- If an artist exists, it must have an ID: `ArtistSummary.id: string` (required)
- Tracks may not have albums: `Track.album: AlbumSummary | undefined`
- Tracks may not have artists: `Track.artist: ArtistSummary | undefined`

**Behavior:**
- Tracks without album/artist IDs have `album: undefined` / `artist: undefined`
- No HTTP 500 errors when processing incomplete metadata
- No API calls to `getAlbum()` or `getArtist()` when IDs don't exist
- SMAPI responses correctly omit albumId/artistId when undefined

**Verification:**
- Playlist with entries lacking album IDs can be browsed in Sonos without errors
- Test suite validates handling of tracks without albums/artists

## What We're NOT Doing

- Not filtering out tracks with missing metadata
- Not changing the Subsonic API response format
- Not adding validation to reject incomplete metadata
- Not modifying how cover art or other optional fields are handled
- Not creating "unknown" albums/artists with placeholder IDs

## Implementation Approach

Test-driven approach: Write failing tests first, then update type definitions, then fix compilation errors. The type system will guide us to all locations that need updating.

---

## Phase 1: Write Failing Tests

### Overview
Create comprehensive test cases that express the desired behavior for tracks without albums/artists.

### Changes Required:

#### 1. Playlist Test - Entries Without Album/Artist IDs
**File**: `tests/subsonic.test.ts`
**Location**: After existing playlist tests (around line 4604)
**Changes**: Add new test case

```typescript
describe("getting a playlist with entries missing album/artist IDs", () => {
  it("should handle entries without albumId or artistId", async () => {
    // Mock Subsonic response with track that has no album/artist/albumId/artistId
    // Expect: track.album = undefined, track.artist = undefined
  });
});
```

#### 2. getTrack Test - Track Without Album ID
**File**: `tests/subsonic.test.ts`
**Location**: In getTrack test section
**Changes**: Add test verifying no getAlbum call when albumId is missing

```typescript
describe("getTrack with missing album ID", () => {
  it("should not call getAlbum when albumId is missing", async () => {
    // Mock getSong response with no albumId
    // Verify getAlbum is NOT called
    // Expect: track.album = undefined
  });
});
```

#### 3. Similar Songs Test - Songs Without Album IDs
**File**: `tests/subsonic.test.ts`
**Location**: In similarSongs test section
**Changes**: Add test for songs without albums

```typescript
describe("similarSongs with missing album IDs", () => {
  it("should not call getAlbum for songs without albumId", async () => {
    // Mock similarSongs response with songs missing albumId
    // Verify getAlbum is NOT called
    // Expect: tracks have album = undefined
  });
});
```

#### 4. Top Songs Test - Songs Without Album IDs
**File**: `tests/subsonic.test.ts`
**Location**: In topSongs test section
**Changes**: Add test for songs without albums

```typescript
describe("topSongs with missing album IDs", () => {
  it("should not call getAlbum for songs without albumId", async () => {
    // Mock topSongs response with songs missing albumId
    // Verify getAlbum is NOT called
    // Expect: tracks have album = undefined
  });
});
```

#### 5. SMAPI Track Metadata Test - Missing Album/Artist
**File**: `tests/smapi.test.ts`
**Location**: In track conversion tests
**Changes**: Add test for track with undefined album/artist

```typescript
describe("track metadata with missing album/artist", () => {
  it("should omit albumId and artistId when undefined", () => {
    // Create Track with album = undefined, artist = undefined
    // Expect: SMAPI response has no albumId, artistId fields
  });
});
```

#### 6. SMAPI Artist Conversion Test - Artist Without ID
**File**: `tests/smapi.test.ts`
**Location**: In artist conversion tests
**Changes**: Add test for optional artist handling

```typescript
describe("artist conversion with undefined artist", () => {
  it("should handle undefined artist gracefully", () => {
    // Test that SMAPI layer can handle artist being undefined
    // Determine appropriate fallback behavior
  });
});
```

### Success Criteria:

#### Automated Verification:
- [ ] All new tests compile (may need TypeScript ignores initially)
- [ ] All new tests FAIL with current implementation
- [ ] Tests run: source ~/.nvm/nvm.sh && nvm exec 20 npm test -- tests/subsonic.test.ts -t "missing"
- [ ] Tests run: source ~/.nvm/nvm.sh && nvm exec 20 npm test -- tests/smapi.test.ts -t "missing"

#### Manual Verification:
- [ ] Review test failures to confirm they fail for the right reasons (non-null assertions, type errors)
- [ ] Verify test coverage includes all code paths affected by the change

---

## Phase 2: Update Type Definitions

### Overview
Make album and artist optional on Track type, and ensure ID fields are required on AlbumSummary and ArtistSummary.

### Changes Required:

#### 1. Track Type
**File**: `src/music_service.ts:63-74`
**Changes**: Make album and artist optional

**Current code:**
```typescript
export type Track = {
  id: string;
  name: string;
  encoding: Encoding,
  duration: number;
  number: number | undefined;
  genre: Genre | undefined;
  coverArt: BUrn | undefined;
  album: AlbumSummary;      // Currently required
  artist: ArtistSummary;    // Currently required
  rating: Rating;
};
```

**New code:**
```typescript
export type Track = {
  id: string;
  name: string;
  encoding: Encoding,
  duration: number;
  number: number | undefined;
  genre: Genre | undefined;
  coverArt: BUrn | undefined;
  album: AlbumSummary | undefined;    // Now optional
  artist: ArtistSummary | undefined;  // Now optional
  rating: Rating;
};
```

#### 2. ArtistSummary Type
**File**: `src/music_service.ts:18-22`
**Changes**: Make id required (currently optional)

**Current code:**
```typescript
export type ArtistSummary = {
  id: string | undefined;    // Currently optional
  name: string;
  image: BUrn | undefined;
};
```

**New code:**
```typescript
export type ArtistSummary = {
  id: string;                // Now required
  name: string;
  image: BUrn | undefined;
};
```

**Reasoning**: If an artist exists, it must have an ID. If no ID exists, don't create an artist object.

#### 3. AlbumSummary Type (No Change Needed)
**File**: `src/music_service.ts:31-40`
**Current state**: `id: string` (already required)

**Verification**: Confirm AlbumSummary.id is already typed as required `string`, not `string | undefined`.

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compilation FAILS with errors at all locations that assume album/artist exist
- [ ] Error count is reasonable (expect 20-40 errors across subsonic.ts, smapi.ts, tests)

#### Manual Verification:
- [ ] Review TypeScript errors to identify all affected code locations
- [ ] Create a list of files/line numbers that need fixes
- [ ] Verify errors appear in expected locations: playlist processing, getTrack, similarSongs, topSongs, SMAPI conversions, test helpers

---

## Phase 3: Fix Subsonic Layer - Playlist Processing

### Overview
Update playlist processing to only create Album/Artist objects when IDs exist.

### Changes Required:

#### 1. Playlist Entry Processing
**File**: `src/subsonic.ts:967-982`
**Changes**: Conditionally create Album/Artist objects

**Current code:**
```typescript
entries: (playlist.entry || []).map((entry) => ({
  ...asTrack(
    {
      id: entry.albumId!,        // Non-null assertion
      name: entry.album!,        // Non-null assertion
      year: entry.year,
      genre: maybeAsGenre(entry.genre),
      artistName: entry.artist,
      artistId: entry.artistId,
      coverArt: coverArtURN(entry.coverArt),
    },
    entry,
    this.customPlayers
  ),
  number: trackNumber++,
})),
```

**New code:**
```typescript
entries: (playlist.entry || []).map((entry) => {
  const album: Album | undefined = entry.albumId
    ? {
        id: entry.albumId,
        name: entry.album || '',
        year: entry.year,
        genre: maybeAsGenre(entry.genre),
        artistName: entry.artist,
        artistId: entry.artistId,
        coverArt: coverArtURN(entry.coverArt),
      }
    : undefined;

  return {
    ...asTrack(album, entry, this.customPlayers),
    number: trackNumber++,
  };
}),
```

**Impact**: Need to update `asTrack()` signature to accept `Album | undefined`.

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compilation succeeds for this section
- [ ] Playlist tests pass: source ~/.nvm/nvm.sh && nvm exec 20 npm test -- tests/subsonic.test.ts -t "playlist"

#### Manual Verification:
- [ ] Code review confirms no non-null assertions remain
- [ ] Verify Album only created when albumId exists

---

## Phase 4: Update asTrack Function Signature

### Overview
Modify `asTrack()` to accept optional Album parameter and create ArtistSummary only when artistId exists.

### Changes Required:

#### 1. asTrack Function
**File**: `src/subsonic.ts:286-315`
**Changes**: Accept optional album, create optional artist

**Current signature:**
```typescript
export const asTrack = (album: Album, song: song, customPlayers: CustomPlayers): Track => ({
```

**New signature:**
```typescript
export const asTrack = (album: Album | undefined, song: song, customPlayers: CustomPlayers): Track => ({
```

**New implementation:**
```typescript
export const asTrack = (album: Album | undefined, song: song, customPlayers: CustomPlayers): Track => {
  const artist: ArtistSummary | undefined = song.artistId
    ? {
        id: song.artistId,
        name: song.artist || '?',
        image: artistImageURN({ artistId: song.artistId }),
      }
    : undefined;

  return {
    id: song.id,
    name: song.title,
    encoding: pipe(
      customPlayers.encodingFor({ mimeType: song.contentType }),
      O.getOrElse(() => ({
        player: DEFAULT_CLIENT_APPLICATION,
        mimeType: song.transcodedContentType || song.contentType
      }))
    ),
    duration: song.duration || 0,
    number: song.track || 0,
    genre: album?.genre || maybeAsGenre(song.genre),
    coverArt: coverArtURN(song.coverArt),
    album,           // Can be undefined
    artist,          // Can be undefined
    rating: {
      love: song.starred !== undefined,
      stars: song.userRating && song.userRating <= 5 && song.userRating >= 0
        ? song.userRating
        : 0,
    },
  };
};
```

**Reasoning**:
- Only create Artist if song.artistId exists
- Only create Album if passed in (caller's responsibility)
- Use optional chaining for album fields (album?.genre)

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compilation succeeds
- [ ] Tests pass: source ~/.nvm/nvm.sh && nvm exec 20 npm test -- tests/subsonic.test.ts

#### Manual Verification:
- [ ] Verify artist only created when artistId exists
- [ ] Check optional chaining used correctly for album fields

---

## Phase 5: Fix Track Retrieval Methods

### Overview
Update getTrack, similarSongs, and topSongs to only call getAlbum when albumId exists.

### Changes Required:

#### 1. getTrack Method
**File**: `src/subsonic.ts:726-735`
**Changes**: Conditionally call getAlbum

**Current code:**
```typescript
getTrack = (credentials: Credentials, id: string) =>
  this.getJSON<GetSongResponse>(credentials, "/rest/getSong", { id })
    .then((it) => it.song)
    .then((song) =>
      this.getAlbum(credentials, song.albumId!).then((album) =>
        asTrack(album, song, this.customPlayers)
      )
    );
```

**New code:**
```typescript
getTrack = (credentials: Credentials, id: string) =>
  this.getJSON<GetSongResponse>(credentials, "/rest/getSong", { id })
    .then((it) => it.song)
    .then((song) => {
      if (song.albumId) {
        return this.getAlbum(credentials, song.albumId).then((album) =>
          asTrack(album, song, this.customPlayers)
        );
      } else {
        return asTrack(undefined, song, this.customPlayers);
      }
    });
```

#### 2. similarSongs Method
**File**: `src/subsonic.ts:1017-1033`
**Changes**: Filter and conditionally call getAlbum

**New code:**
```typescript
similarSongs: async (id: string) =>
  subsonic
    .getJSON<GetSimilarSongsResponse>(
      credentials,
      "/rest/getSimilarSongs2",
      { id, count: 50 }
    )
    .then((it) => it.similarSongs2.song || [])
    .then((songs) =>
      Promise.all(
        songs.map((song) => {
          if (song.albumId) {
            return subsonic
              .getAlbum(credentials, song.albumId)
              .then((album) => asTrack(album, song, this.customPlayers));
          } else {
            return asTrack(undefined, song, this.customPlayers);
          }
        })
      )
    ),
```

#### 3. topSongs Method
**File**: `src/subsonic.ts:1034-1051`
**Changes**: Same pattern as similarSongs

**New code:**
```typescript
topSongs: async (artistId: string) =>
  subsonic.getArtist(credentials, artistId).then(({ name }) =>
    subsonic
      .getJSON<GetTopSongsResponse>(credentials, "/rest/getTopSongs", {
        artist: name,
        count: 50,
      })
      .then((it) => it.topSongs.song || [])
      .then((songs) =>
        Promise.all(
          songs.map((song) => {
            if (song.albumId) {
              return subsonic
                .getAlbum(credentials, song.albumId)
                .then((album) => asTrack(album, song, this.customPlayers));
            } else {
              return asTrack(undefined, song, this.customPlayers);
            }
          })
        )
      )
  ),
```

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compilation succeeds
- [ ] Tests pass: source ~/.nvm/nvm.sh && nvm exec 20 npm test -- tests/subsonic.test.ts -t "getTrack"
- [ ] Tests pass: source ~/.nvm/nvm.sh && nvm exec 20 npm test -- tests/subsonic.test.ts -t "similarSongs"
- [ ] Tests pass: source ~/.nvm/nvm.sh && nvm exec 20 npm test -- tests/subsonic.test.ts -t "topSongs"

#### Manual Verification:
- [ ] Verify getAlbum only called when albumId exists
- [ ] Check that undefined is passed to asTrack when no albumId

---

## Phase 6: Update asTrackFromSearchResult

### Overview
Update search result conversion to pass undefined instead of creating Album with empty string.

### Changes Required:

#### 1. asTrackFromSearchResult Function
**File**: `src/subsonic.ts:325-339`
**Changes**: Conditionally create album

**Current code:**
```typescript
const asTrackFromSearchResult = (song: song, customPlayers: CustomPlayers): Track => {
  const album: Album = {
    id: song.albumId || '',    // Empty string fallback
    name: song.album || '',
    year: song.year,
    genre: maybeAsGenre(song.genre),
    artistId: song.artistId,
    artistName: song.artist,
    coverArt: coverArtURN(song.coverArt),
  };

  return asTrack(album, song, customPlayers);
};
```

**New code:**
```typescript
const asTrackFromSearchResult = (song: song, customPlayers: CustomPlayers): Track => {
  const album: Album | undefined = song.albumId
    ? {
        id: song.albumId,
        name: song.album || '',
        year: song.year,
        genre: maybeAsGenre(song.genre),
        artistId: song.artistId,
        artistName: song.artist,
        coverArt: coverArtURN(song.coverArt),
      }
    : undefined;

  return asTrack(album, song, customPlayers);
};
```

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compilation succeeds
- [ ] Search tests pass: source ~/.nvm/nvm.sh && nvm exec 20 npm test -- tests/subsonic.test.ts -t "search"

#### Manual Verification:
- [ ] Verify album only created when albumId exists

---

## Phase 7: Fix SMAPI Layer

### Overview
Update SMAPI conversion functions to handle undefined albums/artists.

### Changes Required:

#### 1. track Function
**File**: `src/smapi.ts:350-372`
**Changes**: Use optional chaining for album/artist access

**Current code:**
```typescript
export const track = (bonobUrl: URLBuilder, track: Track) => ({
  itemType: "track",
  id: `track:${track.id}`,
  mimeType: sonosifyMimeType(track.encoding.mimeType),
  title: track.name,

  trackMetadata: {
    album: track.album.name,
    albumId: track.album.id ? `album:${track.album.id}` : undefined,
    albumArtist: track.artist.name,
    albumArtistId: track.artist.id ? `artist:${track.artist.id}` : undefined,
    albumArtURI: coverArtURI(bonobUrl, track).href(),
    artist: track.artist.name,
    artistId: track.artist.id ? `artist:${track.artist.id}` : undefined,
    duration: track.duration,
    genre: track.album.genre?.name,
    genreId: track.album.genre?.id,
    trackNumber: track.number,
  },
  // ...
});
```

**New code:**
```typescript
export const track = (bonobUrl: URLBuilder, track: Track) => ({
  itemType: "track",
  id: `track:${track.id}`,
  mimeType: sonosifyMimeType(track.encoding.mimeType),
  title: track.name,

  trackMetadata: {
    album: track.album?.name,
    albumId: track.album?.id ? `album:${track.album.id}` : undefined,
    albumArtist: track.artist?.name,
    albumArtistId: track.artist?.id ? `artist:${track.artist.id}` : undefined,
    albumArtURI: coverArtURI(bonobUrl, track).href(),
    artist: track.artist?.name,
    artistId: track.artist?.id ? `artist:${track.artist.id}` : undefined,
    duration: track.duration,
    genre: track.album?.genre?.name,
    genreId: track.album?.genre?.id,
    trackNumber: track.number,
  },
  // ...
});
```

**Note**: Check if coverArtURI needs updating to handle undefined album/artist.

#### 2. coverArtURI Helper (If Needed)
**File**: `src/smapi.ts` (location TBD)
**Changes**: Handle tracks without album/artist for cover art

May need to update coverArtURI to handle undefined album/artist gracefully.

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compilation succeeds
- [ ] SMAPI tests pass: source ~/.nvm/nvm.sh && nvm exec 20 npm test -- tests/smapi.test.ts

#### Manual Verification:
- [ ] Verify optional chaining used correctly
- [ ] Check SMAPI responses omit undefined fields

---

## Phase 8: Fix Test Helpers

### Overview
Update test helper functions to handle optional albums/artists.

### Changes Required:

#### 1. getPlayListJson Helper
**File**: `tests/subsonic.test.ts:432-470`
**Changes**: Handle optional album/artist

**Current code:**
```typescript
entry: playlist.entries.map((it) => ({
  id: it.id,
  // ...
  album: it.album.name,
  artist: it.artist.name,
  // ...
  albumId: it.album.id,
  artistId: it.artist.id,
  // ...
})),
```

**New code:**
```typescript
entry: playlist.entries.map((it) => ({
  id: it.id,
  // ...
  album: it.album?.name,
  artist: it.artist?.name,
  // ...
  albumId: it.album?.id,
  artistId: it.artist?.id,
  // ...
})),
```

#### 2. Other Test Helpers
**File**: `tests/subsonic.test.ts`, `tests/smapi.test.ts`
**Changes**: Update any other helpers that assume album/artist exist

Search for patterns like:
- `track.album.id`
- `track.artist.id`
- `track.album.name`
- `track.artist.name`

Replace with optional chaining:
- `track.album?.id`
- `track.artist?.id`
- `track.album?.name`
- `track.artist?.name`

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compilation succeeds for all test files
- [ ] All existing tests pass: source ~/.nvm/nvm.sh && nvm exec 20 npm test

#### Manual Verification:
- [ ] Grep for remaining non-optional access patterns
- [ ] Verify test helpers work with both defined and undefined albums/artists

---

## Phase 9: Verify All Tests Pass

### Overview
Run full test suite and verify all new and existing tests pass.

### Success Criteria:

#### Automated Verification:
- [ ] Full test suite passes: source ~/.nvm/nvm.sh && nvm exec 20 npm test
- [ ] No TypeScript compilation errors: source ~/.nvm/nvm.sh && nvm exec 20 npx tsc --noEmit
- [ ] New tests for missing albums/artists pass
- [ ] No regressions in existing tests

#### Manual Verification:
- [ ] Review test output for any warnings
- [ ] Verify test coverage for edge cases
- [ ] Check that all new test cases from Phase 1 now pass

---

## Testing Strategy

### Unit Tests (Added in Phase 1):
- Playlist entries without album/artist IDs
- Single track retrieval without album ID
- Similar songs without album IDs
- Top songs without album IDs
- Search results without album IDs
- SMAPI conversion of tracks without albums/artists
- Test helpers work with optional albums/artists

### Integration Tests:
- End-to-end playlist browsing with mixed metadata (some tracks with albums, some without)
- Verify SMAPI responses have correct structure with optional fields

### Manual Testing Steps:
1. Start bonob server locally
2. Configure with Subsonic account containing playlist with incomplete metadata (like playlist ID 38886 from ticket)
3. Browse the playlist in Sonos app
4. Verify:
   - No HTTP 500 errors in bonob logs
   - Tracks appear in Sonos interface (even those without album/artist)
   - Tracks play correctly
   - Missing album/artist information displayed appropriately
5. Check bonob debug logs to confirm:
   - getAlbum is NOT called for entries without albumId
   - No errors or warnings about undefined values
6. Test similar songs and top songs features with artists that have tracks lacking album IDs

## Performance Considerations

**Positive Impact:**
- Fewer unnecessary API calls to `getAlbum()` when album IDs don't exist
- Faster response times for playlists/searches with incomplete metadata
- Reduced network latency and Subsonic server load

**No Performance Degradation:**
- Conditional checks (`if (song.albumId)`) have negligible overhead
- Optional chaining (`track.album?.name`) is compile-time only, no runtime cost
- No additional database queries or network requests

## Migration Notes

**No Migration Required:**
- This is a bug fix, not a data migration
- Existing tracks with albums/artists continue to work identically
- Only affects behavior for tracks that currently cause HTTP 500 errors

**Backwards Compatibility:**
- SMAPI responses maintain same structure
- Optional fields that were conditionally set remain conditional
- No breaking changes to public API contracts
- Clients already handle optional albumId/artistId fields

## References

- Original ticket: `claude-plans/1109-dont-getalbum-no-album.md`
- SMAPI optional field handling: src/smapi.ts:358, 360, 363
- Type definitions: src/music_service.ts:18-74
- Search optimization pattern: `asTrackFromSearchResult()` at src/subsonic.ts:325-339
