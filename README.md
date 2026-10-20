# Marked Markdown Curator filter

A zsh filter that replaces [Marked](https://markedapp.com)’s input with a [Markdown Curator](https://github.com/voostindie/markdown-curator) export to vanilla Markdown when the current document belongs to a configured curator. The export is created live through the HTTP interface of the Markdown Curator.

If you don’t know what the Markdown Curator is, then you’re in the wrong place. This is not for you.

“Vanilla Markdown” here means that all wikilinks — those things between double brackets — are replaced with normal text. Marked in turn will give you a clean HTML version that you can copy and paste anywhere you like, without those weird double brackets or links that can’t be followed. The algorithm for replacing links with text is as follows:

- `[[Document]]` becomes `Document`.
- `[[Document|Alias]]` becomes `Alias`.
- `[[Document#Section]]` becomes `Document > Section`.
- Aliases take precedence over anchors: `[[Document#Section|Alias]]` becomes `Alias`.

## Setup

1. Edit `curators` beside `markdown-curator-filter`. Each line contains a curator name, whitespace, and an absolute vault directory. Paths may contain spaces; do not quote them. Blank lines and lines starting with `#` are ignored. Curator names may contain letters, numbers, `.`, `_`, `~`, and `-`.
2. Configure Marked's custom preprocessor to run the absolute path to `markdown-curator-filter`. Keep `curators` beside the script.
3. Run Markdown Curator on `localhost:7767`.

The filter checks `MARKED_PATH` first. If it is missing or empty, input passes through unchanged. Missing configuration or an unmatched document also passes input through unchanged.

Curator paths match directory boundaries: `/vault/notes` matches
`/vault/notes/Document.md`, but not `/vault/notes-other/Document.md`. Trailing slashes are ignored. The longest matching directory wins; ties use the first entry. Matching is case-sensitive and uses the supplied paths without resolving symlinks.

The final filename loses its last extension and preserves its case. For example, `/vault/notes/My Document.md` becomes `My Document`. The filter sends a GET to `http://localhost:7767/curators/<curator>/queries/export`, sets `Accept: text/markdown`, and encodes the `document` query with curl's `--data-urlencode`.

A valid response means curl succeeded, HTTP status is 2xx, and the body contains non-whitespace content. Connection errors, HTTP errors, redirects, empty bodies, and timeouts fall back to the original input. Connection timeout is 2 seconds; total timeout is 10 seconds. 

Successful exports and fallback input preserve their original bytes, including trailing newlines. Successful exports do not read standard input.

## Export endpoint configuration

The `export` endpoint lives at `http://localhost:7767/curators/<curator>/queries/export`. This is a hardcoded URL, except for the `<curator>` part.

The export endpoint supports two additional query parameters that are *not* used in this script:

- `add-title`: adds a level 1 header at the top of the output with the title of the document, unless the document itself contains one already.
- `strip-emojis-from-links`: removes emoji characters from internal links (that are transformed into text).

This script doesn’t configure either setting, meaning that both settings are active. Don’t like that? Add a `--data-url-encode “<setting>=false"` to the call to curl in the script, replacing `<setting>` with the appropriate setting name.

## Verification

Run `zsh ./test-markdown-curator-filter`. Tests use a zsh fake curl executable and temporary configuration; no running server or additional test framework is required.

