The background daemon starts automatically on first use.

Tip: ccc index auto-initializes if you haven't run ccc init yet, so you can skip straight to indexing.

CLI Reference
Command	Description
ccc init	Initialize a project — creates settings files, adds .cocoindex_code/ to .gitignore
ccc index	Build or update the index (auto-inits if needed). Shows streaming progress.
ccc search <query>	Semantic search across the codebase
ccc grep <pattern> [path]	Structural code search by example (no index needed)
ccc status	Show index stats (chunk count, file count, language breakdown)
ccc mcp	Run as MCP server in stdio mode
ccc doctor	Run diagnostics — checks settings, daemon, model, file matching, and index health
ccc reset	Delete index databases. --all also removes settings. -f skips confirmation.
ccc daemon status	Show daemon version, uptime, and loaded projects
ccc daemon restart	Restart the background daemon
ccc daemon stop	Stop the daemon
Search Options
ccc search database schema                           # basic search
ccc search --lang python --lang markdown schema      # filter by language
ccc search --path 'src/utils/*' query handler        # filter by path
ccc search --offset 10 --limit 5 database schema     # pagination
ccc search --refresh database schema                 # update index first, then search
By default, ccc search scopes results to your current working directory (relative to the project root). Use --path to override.

Structural Search (ccc grep)
ccc grep finds code by structure, not text — you write a by-example pattern and it matches the syntax tree (via cocoindex's code_match), so formatting, whitespace, and intervening tokens don't matter. It runs entirely locally: no index, daemon, or embeddings required.

ccc grep 'def \NAME(\(ARGS*\)):'                      # every Python function def under the cwd
ccc grep 'foo(\(ARGS*\))' src/                        # calls to foo(...) anywhere under src/
ccc grep 'fn \NAME(\(A*\))' --lang rust               # restrict to one language
ccc grep 'class \NAME:' --path 'tests/**'            # restrict to a path glob
ccc grep 'TODO(\(A*\))' path/to/file.py               # a single file
Metavariables use the \ sigil: \NAME captures one node, \(NAME*\) a run of siblings, \_/\* match anonymously. The pattern is matched per language, so a single invocation scans every supported source file (others are skipped). Inside an initialized project, ccc grep honors the project's include/exclude patterns and .gitignore; otherwise it scans all supported source files under the path.

Results stream to the terminal file-by-file as each match is found (in completion order, since files are matched in parallel) rather than all at once at the end. Each matching file shows its matched line range; under a TTY the path is colored, line numbers are dimmed, and the unmatched context around a match is dimmed so the match stands out.