# SPIMquant-intro
Introduction and tutorial for SPIMquant

## Contents

- [`presentation.md`](presentation.md) — Marp-based slide deck covering the SPIMquant intro tutorial and demo.
- [`draft_tutorial.md`](draft_tutorial.md) — Outline / notes used to build the presentation.
- [`spimquant_analysis_tutorial.ipynb`](spimquant_analysis_tutorial.ipynb) — Jupyter notebook tutorial for exploring SPIMquant workshop outputs with the public workshop archive.

## Viewing the Presentation

The presentation is written for [Marp](https://marp.app/), a Markdown Presentation Ecosystem.

### Option 1 — Marp CLI

```bash
# Install Marp CLI
npm install -g @marp-team/marp-cli

# Export to HTML (recommended for sharing)
marp presentation.md --output presentation.html

# Export to PDF
marp presentation.md --pdf --output presentation.pdf

# Live preview in browser
marp presentation.md --preview
```

### Option 2 — VS Code Extension

Install the [Marp for VS Code](https://marketplace.visualstudio.com/items?itemName=marp-team.marp-vscode) extension to preview and export slides directly from the editor.

## Running the Tutorial Notebook

The notebook tutorial uses the public SPIMquant workshop archive and, by default, downloads it under `/tmp/spimquant_workshop`.

```bash
python -m pip install notebook pandas matplotlib
jupyter notebook spimquant_analysis_tutorial.ipynb
```

Inside the notebook, run the cells in order to:

- download or reuse the workshop archive
- inspect the extracted directory structure
- review participant metadata and subject-level tables
- summarize cohort- and group-level SPIMquant outputs
- identify QC artifacts to open in a browser, ITK-SNAP, or napari

## Links

- 🔗 SPIMquant GitHub: <https://github.com/khanlab/SPIMquant>
- 📖 SPIMquant Docs: <https://spimquant.readthedocs.io>
