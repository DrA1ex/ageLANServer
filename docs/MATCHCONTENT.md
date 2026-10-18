# AoE2 DE MatchContent protocol investigation

Base: upstream `1.16.0`, commit `6f20aa0688469d6b8f37e4a0c2a8b7e347b44a92`.
No new pull request is associated with this branch.

## Evidence and scope

The locally available AoE2 DE executable reports build 185872
(101.103.54800.0). Its SHA-256 is
`124744bf4ae0d3856238e00d61fe321d47fcd3c0d7dfcd175c6c76aa9fb458ff`.
The archived EXE and installed EXE have the same digest. Some networking
functions contain encrypted bytes on disk. The conclusions below come from
read-only analysis of loaded code in CrossOver Preview, not just string searches.
No game binary, memory dump, private log, savegame or certificate is distributed
in this repository.

Addresses below are virtual addresses for this executable's preferred image base
`0x140000000`. They are analysis landmarks, not a promise about other builds.

* Upload at `0x14108c160` posts multipart `fileContent` and decimal `matchId`.
  At `0x14108c8f6` it parses the response as JSON. At `0x14108c99f` onward it
  requires `isError`, reads it as a boolean and negates it; a missing field fails.
  Thus an empty HTTP 200 response is not a successful upload for this client.
* Metadata retrieval at `0x14108b690` performs a GET, checks the transport result,
  and parses the body as JSON (`0x14108bb00`). It does not treat the body as the
  saved game.
* The caller at `0x140dfe430` checks `fileList` and its entries' `fileName` and
  `fileSize` before invoking the next stage. This ties these fields to the
  multiplayer custom-file transfer, rather than an unrelated JSON feature.
* Download processing at `0x14108ae10` checks `fileList`; for each entry it requires
  string `fileName`, `filePath`, and `fileCRC`. `filePath` is passed to the download
  routine at `0x14175f370`. `fileCRC` is converted from a string in base 16 at
  `0x14108b2c4`. It is compared to the existing local file's 32-bit checksum to
  decide whether that file can be reused.
* The checksum path reaches `0x140a88a30`, which complements its initial and final
  accumulators and uses the CRC-32/IEEE table at `0x143daa0d0` (entries begin
  `00000000 77073096 ee0e612c 990951ba`). Initial checksum is zero; the standard
  `123456789` vector produces `cbf43926`. This is not the MD5 previously placed in
  HTTP headers.

The corresponding minimal success responses are:

```json
{"isError":false}
```

and

```json
{"isError":false,"fileList":[{"fileName":"TEST.aoe2mpgame","fileSize":9,"filePath":"https://api.ageofempires.com/api/MatchContent/123/file/25f9e794323b453885f5181f1b624d0b","fileCRC":"cbf43926"}]}
```

The URL path is a local server implementation choice. The client consumes the
absolute URL in `filePath`; the original service's storage URL layout has not
been captured. This implementation keeps payloads on the same host and port,
uses HTTPS when the metadata request uses TLS, and includes the content digest
to reject URLs for an overwritten file. Filename sanitization still protects
server disk paths while the metadata preserves the client's Unicode basename.
The file remains streamed to/from disk and its CRC and MD5 are calculated while
uploading. Open and overwrite are synchronized so metadata and payload refer
to the same version.

Communication logging previously read the whole upload before the handler
could extend its deadline. It now observes MatchContent uploads as they stream,
and retains at most 4097 response bytes while hashing the complete response.
The extra byte distinguishes a body exceeding the 4096-byte log limit.

## Previous logs

The original `2026-10-17T19-10-22` server session contains successful storage of
1261799 bytes followed by POST 200 with no body, then GET 200 with raw bytes.
One attempt used match id 385042612; another used 513374789 with the host and
client reversed. These are separate attempts. Match 720660433 also appears in
the game log `2026.10.18-0026.21/MainLog.txt`, followed by `Posting 1 files failed`.
Game and server wall-clock labels differ; matching ids and relative event order
are stronger correlation than the displayed timestamps alone.
The earlier `{isError:false}` upload change was only a hypothesis. Loaded-code
analysis now supports it, and also identifies the missing metadata stage.

## Validation limits

Tests cover HTTP/HTTPS upload → JSON metadata → separate file download, exact
bytes, length, independent CRC and MD5 vectors, Unicode names, missing/invalid
ids, quotas, replacement, stale URLs, concurrent open/overwrite and bounded
logging. Existing router tests continue to cover all registered games, but the
actual client contract was analyzed only for the AoE2 DE build above.

A complete end-to-end claim requires two game clients to restore the same saved
multiplayer match and resume play. Automated HTTP tests, a game reaching its
menu, or even a successful host upload do not establish that result.


Validation commands on macOS arm64 with Go 1.27.2:

* `go test -race ./server/internal/routes/router ./server/internal/models`: passed.
* `go test -race ./server/...`: one existing DNS test failed. The same
  `TestResolveHostsViaCommonHostOrIp` failure was reproduced in an unmodified
  archive of the base commit (the local resolver resolves the invented hostname).
* `go test -race ./server/... -skip '^TestResolveHostsViaCommonHostOrIp$'`: passed.
* `go vet ./server/internal/routes/router ./server/internal/routes/matchcontent ./server/internal/models`: passed.
* `go vet ./server/...`: existing unkeyed struct literal diagnostic in
  `server/internal/models/athens/routes/playfab/items.go:27`; this file is unchanged.
* `CGO_ENABLED=0 GOOS=darwin GOARCH=arm64 go build ./server`: passed.
* `CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build ./server`: passed.
* Universal binary architecture and ad-hoc code-signature verification: passed.
* Installer/rollback checked against temporary installations with a missing,
  disabled and partially configured CustomFiles section. Config preservation,
  byte-exact rollback and active-session rejection passed.
