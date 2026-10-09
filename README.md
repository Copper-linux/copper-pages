# copper-pages

The package repository that **ingot**, Copper Linux's own package manager,
fetches from.

This is a GitHub **Pages** repo, and it holds only small JSON metadata files —
never binaries. The binaries live on this repository's **Releases**; each JSON
page points at the real payload URL plus the sha256 that payload must match.
That indirection is the whole design: Pages is the index, Releases are the
warehouse.

Default repo URL (used by ingot unless `INGOT_REPO` or `/etc/ingot.conf`
overrides it):

    https://copper-linux.github.io/copper-pages

## Layout

    iso/copper/pkg/index.json              name -> category, one entry per line
    iso/copper/pkg/<category>/<name>       one JSON page per package (no extension)

What ingot fetches, exactly:

1. `<repo-url>/iso/copper/pkg/index.json`
   — maps a package name to its category
2. `<repo-url>/iso/copper/pkg/<category>/<name>`
   — the package's JSON page

`.nojekyll` sits at the repo root so Pages serves every file verbatim,
including the extensionless package pages.

## index.json format

One `name -> category` entry per line. ingot finds an entry with a sed line
matcher, so each entry must **start the line** (after optional whitespace) and
its `: "value"` must be on the **same line**:

```json
{
  "libpcap": "libs",
  "nmap": "hacking",
  "tcpdump": "network",
  "tree": "tools"
}
```

The category is also the directory the page lives in: `"tcpdump": "network"`
means the page is at `iso/copper/pkg/network/tcpdump`.

## Package page format

One key per line, flat string values, double-quoted, on the same line as the
key. `depends` is the ONLY array and must be on a single line. Pretty-print —
a minified one-line file will break ingot, whose parser is a sed line-matcher,
not a JSON parser.

```json
{
  "name": "tcpdump",
  "version": "4.99.7-1",
  "category": "network",
  "url": "https://github.com/Copper-linux/copper-pages/releases/download/payloads-v1/tcpdump-4.99.7-1.tar.gz",
  "sha256": "6ae1d621c17e1931747ad6fdb31e67e8f7299437b589ed66ec580cc2d2b0e8a2",
  "depends": ["libpcap"]
}
```

Field rules:

| field | required | shape |
| --- | --- | --- |
| `name` | yes | string, same line as the key |
| `version` | yes | string |
| `category` | yes | string, must match `index.json` and the directory |
| `url` | yes | string — the REAL payload tarball (a Release asset) |
| `sha256` | yes | string — 64-char hex of that exact tarball |
| `depends` | no | single-line array `["a","b"]`, empty `[]`, or omit the line |

If `url` or `sha256` is missing, ingot refuses to install — by design. Key
order does not matter; field **names** must match exactly.

## The payload (`url` + `sha256`)

- `url` points at a `.tar.gz` attached to a **GitHub Release** of this repo
  (Pages never holds binaries) and must be reachable with no auth:
  `https://github.com/Copper-linux/copper-pages/releases/download/<tag>/<asset>`
- `sha256` is the SHA-256 of that exact tarball's bytes:

      sha256sum <file> | awk '{print $1}'

  If you replace the asset, the bytes (and the hash) change — recompute it.

- The tarball's contents are extracted **relative to `/`** on the target
  system, so it must contain real filesystem paths only:

      usr/bin/tcpdump
      usr/lib/libpcap.so.1
      usr/share/man/man1/tcpdump.1

- It MUST NOT contain absolute paths (leading `/`) or any `..` path segment;
  ingot scans `tar -tzf` output and refuses the payload outright if it does.
  Build it that way from the start:

      tar -czf hello-1.0-1.tar.gz -C staging usr

## Packages

| package | category | version | depends |
| --- | --- | --- | --- |
| libpcap | libs | 1.11.0-1 | — |
| nmap | hacking | 7.95-1 | libpcap |
| tcpdump | network | 4.99.7-1 | libpcap |
| tree | tools | 2.3.2-1 | — |

All payloads are x86_64 Linux (glibc), built from upstream sources
(libpcap/tcpdump from tcpdump.org releases, nmap from nmap.org 7.95, tree from
[Old-Man-Programmer/tree](https://github.com/Old-Man-Programmer/tree)), stripped,
and staged with `tar -czf out.tar.gz -C <staging> usr`.

`tcpdump` and `nmap` are real demonstrations of the dependency mechanism: both
binaries have `libpcap.so.1` as a dynamic `NEEDED` entry, and their pages carry
`"depends": ["libpcap"]`, so `ingot install tcpdump` (or `nmap`) installs
`libpcap` first.

## Adding a package

1. **Stage and build the tarball.** Contents extract relative to `/`; use
   `usr/...` paths only — no absolute paths, no `..`:

       mkdir -p staging/usr/bin
       cp hello staging/usr/bin/
       tar -czf hello-1.0-1.tar.gz -C staging usr
       tar -tzf hello-1.0-1.tar.gz     # inspect: usr/... only

2. **Upload it to a Release** (Pages is not for binaries):

       gh release create payloads-v1 hello-1.0-1.tar.gz
       # or, if the release already exists:
       gh release upload payloads-v1 hello-1.0-1.tar.gz

3. **Compute the sha256 of the exact asset bytes** (the file you just
   uploaded — hash it locally, don't rebuild after hashing):

       sha256sum hello-1.0-1.tar.gz | awk '{print $1}'

4. **Write the page** at `iso/copper/pkg/<category>/<name>` (one file per
   package, no extension), pretty-printed, one key per line:

       {
         "name": "hello",
         "version": "1.0-1",
         "category": "tools",
         "url": "https://github.com/Copper-linux/copper-pages/releases/download/payloads-v1/hello-1.0-1.tar.gz",
         "sha256": "<64-char hex of the payload tarball>",
         "depends": []
       }

5. **Add the name to `index.json`**, on its own line:

       "hello": "tools",

6. **Commit and push.** GitHub Pages republishes from `main` (root) within a
   minute or so.

## Verifying

Point ingot at the site and install into a scratch root — it never touches
your real filesystem this way (`ingot` is
`iso/rootfs-overlay/usr/bin/ingot` in `Copper-linux/copper`):

    INGOT_REPO="https://copper-linux.github.io/copper-pages" \
    INGOT_ROOT="$PWD/scratch" \
    sh ingot install tcpdump

Success prints:

    ingot: installed libpcap
    ingot: installed tcpdump

and the files appear under `scratch/` (`scratch/usr/bin/tcpdump`,
`scratch/usr/lib/libpcap.so.1`, ...). ingot also records an installed
manifest under `<root>/var/lib/ingot/`, verifies the payload sha256 before
unpacking, and deletes the package JSON from `/tmp` when it is done.