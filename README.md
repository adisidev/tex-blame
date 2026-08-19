# tex-blame

`git blame` for LaTeX papers, rendered as a compiled PDF.

Point it at a paper's git history and it compiles one PDF in which every author's contributions appear in their own colour — base text black, additions coloured by author, deletions struck through in the colour of whoever removed them, and answered TODO scaffolds struck through by their answerer instead of silently vanishing.

Built on [`latexdiff`](https://github.com/ftilmann/latexdiff), which does all the actual diffing; `tex-blame` orchestrates it across a chain of revisions, keeps the attribution honest, and gets the result through `latexmk` in one go.

## Modes

```sh
# blame everything after a given commit (before it: black)
tex-blame --since BASE_SHA --main docs/src/manuscript.tex

# blame the entire history — base is the empty tree, every word gets an author
tex-blame --all --main docs/src/manuscript.tex

# explicit chain of revisions (first one shown black)
tex-blame --git BASE REV1 REV2 --main manuscript.tex --labels alice,bob

# no git at all: a chain of directories
tex-blame --dirs old/ theirs/ mine/ --main manuscript.tex
```

In `--since`/`--all` mode the first-parent commit chain is walked, consecutive commits by the same author are squashed into one round (`--per-commit` to keep them separate), and each author is assigned a colour from a built-in palette. A legend box is injected after `\maketitle`.

## Requirements

- `latexdiff` (tested with 1.4.0)
- `latexmk` + a TeX distribution
- Python 3.8+, `git` for the git modes

No Python dependencies; `tex-blame` is a single file. Put it on your `PATH`:

```sh
ln -s "$(pwd)/tex-blame" ~/.local/bin/tex-blame
```

## Options

| flag | meaning |
|---|---|
| `--since BASE` | blame every commit after `BASE`; earlier text stays black |
| `--all` | blame the whole history (base = empty tree) |
| `--git BASE R1 [R2 ...]` | explicit revision chain; `BASE` shown black |
| `--dirs D0 D1 [D2 ...]` | explicit directory chain |
| `--head REF` | newest revision for `--since`/`--all` (default `HEAD`) |
| `--repo PATH` | git repository root (default `.`) |
| `--main PATH` | main `.tex` file, relative to the project root (required) |
| `--labels a,b,...` | legend labels, one per round (default: author names) |
| `--colors x,y,...` | colour names, one per round (default: built-in palette per author) |
| `--base-label TEXT` | legend label for black text |
| `--per-commit` | one round per commit instead of squashing author runs |
| `--scaffold CMD` | scaffold macro (repeatable; default `todo`, `decide`) |
| `--no-scaffold` | disable scaffold handling |
| `--opaque-env ENV` | extra environment to treat as an opaque block (repeatable) |
| `--no-legend` | skip the legend box |
| `--no-pdf` | write the merged `.tex` and stop |
| `--open` | open the PDF when done (macOS) |
| `--keep` | keep the temporary work directory |
| `--latexdiff-arg ARG` | pass an extra argument to every `latexdiff` call |
| `-o PATH` | output PDF path (default `<main>-blame.pdf`) |

## Scaffold macros: seeing who answered a TODO

Papers often carry scaffold markers — `\todo{...}`, `\decide{...}` — that a later round *answers* by deleting the marker and writing real prose. Plain `latexdiff` hides a deleted macro call inside an invisible `%DIFDELCMD` comment, so answered todos silently disappear from a diff.

`tex-blame` declares scaffold macros as latexdiff *text commands*: their argument is diffed like prose, so an answered todo stays visible, struck through in the colour of the round that answered it, right next to the prose that replaced it. Edits *within* a todo diff word-by-word the same way. Scaffold macros must take a single-paragraph argument.

## How it works

1. The revision chain is built (explicitly, or from the first-parent git history) and the `\input`/`\include` graph is walked from the main file of the newest revision.
2. **Per-file attribution.** A file changed by only one round gets a plain two-way diff (base vs newest) attributed wholly to that round — chained diffs are avoided wherever possible, because chaining is what produces artefacts.
3. A file changed by **several** rounds gets a chained diff, newest round first: each stage's `latexdiff` markup is immediately renamed into that round's private macro namespace (`\ATBDIFadd`, `\BTBDIFadd`, ...), so earlier stages see it as opaque tokens and attribution cannot leak between rounds.
4. Files `\input` in the **preamble** are taken verbatim from the newest revision (markup cannot render before `\begin{document}`); the root file's preamble likewise, with only its body diffed.
5. **Deleted-text archaeology.** latexdiff colours a deletion only by its deleter; to also show the *author*, every deletion span (always plain old-side text) is searched for in the earlier revisions of the file — whitespace-insensitively, since latexdiff does not preserve exact spacing — and wrapped in a colour-only command for the earliest revision that contains it. The result: deleted text renders dimmed in its author's colour with the strike line in the deleter's colour. Spans shorter than 12 characters stay unattributed (grey) to avoid false matches.
6. Raw-content environments (`CCSXML`, `verbatim`, `lstlisting`, `tikzpicture`, `filecontents`) are treated as opaque blocks that swap wholesale rather than being marked up word-by-word.
7. Everything is flattened into one `.tex`, per-round colour definitions and a legend are injected, and `latexmk` compiles it inside a copy of the newest tree so the bibliography, figures, and document class resolve normally.

## Limitations

- The chain must be linear: each revision must contain the previous one's changes. For git histories the first-parent path is used; content merged in via PRs is attributed to the round that brought it onto the first-parent line.
- A passage rewritten by **several** rounds can render noisily (the same original text struck once per round). Attribution of the surviving text stays correct; deleted-text author attribution is heuristic (whitespace-insensitive substring matching) and falls back to grey when a span cannot be found in an earlier revision.
- The bibliography compiles from the newest `.bib`; `.bib` changes are not marked up.
- The root file must contain `\begin{document}` and `\end{document}` directly (not via `\input`).
- At most 26 rounds (one markup namespace letter per round); author squashing keeps real histories well under this.
- An `\input` line added or removed between revisions may end up wrapped in markup that prevents inlining; the compile then pulls that file from the newest tree unmarked.

## License

GPL-3.0-or-later. If you distribute a modified version, the GPL requires you to make its source available under the same terms.
