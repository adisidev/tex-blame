# latexdiff3

Three-way `latexdiff` for multi-author rounds. Given three revisions of a LaTeX project — `BASE → A → B`, where B builds on A — it compiles a single PDF in which:

- text present in **BASE** stays **black**,
- text added in round **A** is set in one colour (default `Teal`),
- text added in round **B** is set in a second colour (default `RoyalBlue`),
- deletions are struck through in the colour of the round that removed them.

The typical use: a co-author pushed a round of edits (A), you wrote your own round on top (B), and you want one document that shows who contributed what relative to the last common state — instead of two separate two-way diffs.

## Requirements

- `latexdiff` (tested with 1.4.0)
- `latexmk` + a TeX distribution
- Python 3.8+
- `git` (for `--git` mode)
- colour names are `xcolor` names; the document (or its class) must load `xcolor`

## Install

Put the `latexdiff3` script somewhere on your `PATH`:

```sh
ln -s "$(pwd)/latexdiff3" ~/.local/bin/latexdiff3
```

## Usage

Diff three git revisions of a paper (run anywhere, point `--repo` at the checkout):

```sh
latexdiff3 --git BASE_SHA A_SHA B_SHA \
  --repo ~/github/my-paper --main docs/src/manuscript.tex \
  --label1 "co-author" --label2 "me" \
  -o threeway.pdf --open
```

Or three plain directories, each containing the full project tree:

```sh
latexdiff3 --dirs old/ theirs/ mine/ --main manuscript.tex
```

Options:

| flag | meaning |
|---|---|
| `--git BASE A B` / `--dirs BASE A B` | the three revisions (git SHAs/refs, or directories) |
| `--repo PATH` | git repository root for `--git` (default `.`) |
| `--main PATH` | main `.tex` file, relative to the project root (required) |
| `--label1/--label2` | legend labels for the two rounds |
| `--color1/--color2` | `xcolor` colour names (default `Teal` / `RoyalBlue`) |
| `--scaffold CMD` | scaffold macro handled specially (repeatable; default `todo`, `decide`) |
| `--no-scaffold` | disable scaffold handling |
| `--no-legend` | skip the legend box injected after `\maketitle` |
| `--no-pdf` | write the merged `.tex` next to you and stop |
| `--open` | open the PDF when done (macOS) |
| `--keep` | keep the temporary work directory (debugging) |
| `--latexdiff-arg ARG` | pass an extra argument to every underlying `latexdiff` call |
| `-o PATH` | output PDF path (default `<main>-diff3.pdf` in the cwd) |

## Scaffold macros: seeing who answered a TODO

Papers often carry scaffold markers — `\todo{...}`, `\decide{...}` — that a later round *answers* by deleting the marker and writing real prose. Plain `latexdiff` hides a deleted macro call inside an invisible `%DIFDELCMD` comment, so the answered todo silently disappears from the diff.

`latexdiff3` declares scaffold macros as latexdiff *text commands*: their argument is diffed like prose, so an answered todo stays visible, **struck through in the colour of the round that answered it**, right next to the prose that replaced it. Edits *within* a todo diff word-by-word the same way.

Add your own markers with `--scaffold mynote --scaffold draft` (this replaces the default list; include `todo`/`decide` if you still want them). Scaffold macros must take a single-paragraph argument.

## How it works

1. The three trees are extracted and the `\input`/`\include` graph is walked from the main file.
2. **Per-file attribution.** A file changed in only one round gets a plain two-way diff (BASE vs B) attributed wholly to that round — chained diffs are avoided wherever possible because they are what produces artefacts.
3. A file changed in **both** rounds gets a two-stage diff: A vs B is diffed first and its markup renamed into a private namespace (`\ADIFadd`/`\ADIFdel`), then BASE is diffed against that marked file, producing the outer `\DIFadd`/`\DIFdel` markup for round A. The rename keeps the two rounds' markup from colliding, and the B-round spans are treated as opaque tokens in the second pass so attribution doesn't leak.
4. Files `\input` in the **preamble** are taken verbatim from B (markup cannot render before `\begin{document}`). The root file's preamble likewise comes from B; only its body is diffed.
5. Everything is flattened into one `.tex`, colour definitions and a legend are injected, and `latexmk` compiles it inside a copy of B's tree so the bibliography, figures, and document class resolve normally.

## Limitations

- Assumes a linear history: B must contain A's changes (e.g. A merged/cherry-picked before B was written). If A and B are divergent siblings, merge first and use the merged state as B.
- In files both rounds touched, a passage that **both** rounds rewrote can render noisily (the same original text struck once per round). Attribution of the surviving text stays correct.
- `verbatim`/`listings` content is not supported (latexdiff's verbatim preamble extensions are stripped to avoid duplicate-definition clashes).
- The bibliography compiles from B's `.bib`; `.bib` changes are not marked up.
- The root file must contain `\begin{document}` and `\end{document}` directly (not via `\input`).
- An `\input` line added or removed between revisions may end up wrapped in markup that prevents flattening from inlining it; the compile will then pull the file from B's tree unmarked.
