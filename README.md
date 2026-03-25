# export-repo-wiki-to-pdf

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python](https://img.shields.io/badge/Python-3.8%2B-blue)](https://www.python.org/)

Export Qoder repository wiki to PDF format with full Mermaid diagram support. Converts markdown wiki files to individual PDFs and merges them into a single complete document.

## Features

- **Mermaid Diagram Support** - Automatically renders Mermaid diagrams in PDF output
- **Auto-Installation** - Missing Python dependencies are installed automatically
- **Parallel Processing** - Configurable worker threads for faster conversion
- **Cover Page & TOC** - Professional cover page and table of contents included
- **Cross-Platform** - Works on Windows, macOS, and Linux
- **Syntax Correction** - Automatically fixes common Mermaid syntax errors
- **Clean & Safe** - Source wiki is never modified; all temp files isolated

## Installation

### Prerequisites

- Python 3.8 or higher
- pip (Python package manager)

### Required Packages

The script will auto-install these on first run, or you can install manually:

```bash
pip install md2pdf-mermaid pypdf reportlab
```

**Note**: `md2pdf-mermaid` uses Playwright. If you encounter browser-related issues, run:
```bash
playwright install
```

## Quick Start

1. **Navigate to your project root** (where `.qoder/repowiki/` exists)

2. **Create the temp directory and script:**
   ```bash
   mkdir -p .qoder/temp
   ```

3. **Copy the conversion script** from [SKILL.md](SKILL.md) to `.qoder/temp/convert_wiki_to_pdf.py`

4. **Run the converter:**
   ```bash
   python .qoder/temp/convert_wiki_to_pdf.py
   ```

5. **Find your output:** `Wiki-Complete.pdf` in your project root

## Usage Examples

### Basic Usage
```bash
python .qoder/temp/convert_wiki_to_pdf.py
```

### With Verbose Output
```bash
python .qoder/temp/convert_wiki_to_pdf.py --verbose
```

### Faster Conversion (More Workers)
```bash
python .qoder/temp/convert_wiki_to_pdf.py --workers 8
```

### Custom Output Path
```bash
python .qoder/temp/convert_wiki_to_pdf.py --output "My-Documentation.pdf"
```

### Skip Cover Page
```bash
python .qoder/temp/convert_wiki_to_pdf.py --no-cover
```

### Keep Temporary Files (for debugging)
```bash
python .qoder/temp/convert_wiki_to_pdf.py --no-cleanup
```

## Command-Line Options

| Option | Description | Default |
|--------|-------------|---------|
| `--wiki-path PATH` | Path to wiki content directory | `.qoder/repowiki/en/content` |
| `--output FILENAME` | Output PDF filename | `Wiki-Complete.pdf` |
| `--temp-dir DIR` | Temp directory for working files | `.qoder/temp` |
| `--workers N` | Number of parallel workers | `4` |
| `--verbose` | Enable detailed debug output | - |
| `--no-cover` | Skip cover page and TOC | - |
| `--no-auto-install` | Disable auto-installation | - |
| `--no-cleanup` | Keep temporary files | - |
| `--clean-all` | Remove entire temp directory after | - |

## Directory Structure

```
project-root/
├── .qoder/
│   ├── temp/                          # Created by this tool
│   │   ├── convert_wiki_to_pdf.py     # The conversion script
│   │   └── pdfs/                      # Temporary PDF files (auto-cleaned)
│   └── repowiki/                      # Source wiki (READ ONLY)
│       └── en/content/
│           ├── Getting Started.md
│           └── ...
└── Wiki-Complete.pdf                  # Final output
```

## How It Works

1. **Locate** - Finds all markdown files in `.qoder/repowiki/{language}/content/`
2. **Convert** - Transforms each markdown file to PDF using `md2pdf-mermaid`
3. **Fix** - Automatically corrects Mermaid syntax errors (e.g., `End` → `end`)
4. **Merge** - Combines all PDFs into one with cover page and table of contents
5. **Clean** - Removes temporary files (optional)

## Troubleshooting

| Issue | Solution |
|-------|----------|
| PowerShell `ParserError` | Write script to file instead of using `python -c` |
| `ModuleNotFoundError` | Dependencies auto-install; or run `pip install md2pdf-mermaid pypdf reportlab` |
| Mermaid "Syntax error" | Script auto-fixes common issues; check `--verbose` output |
| Playwright/Chrome errors | Run `playwright install` |
| Slow conversion | Increase workers: `--workers 8` |
| Timeout errors | Reduce worker count or check file sizes |

See [SKILL.md](SKILL.md) for complete documentation and the full Python script.

## Limitations

- **Images**: Wiki images stored in Qoder IDE aren't exported to local filesystem
- **Interactive Elements**: Links work, but interactive features don't
- **File Size**: PDFs with many Mermaid diagrams can be large (10-30 MB typical)

## License

[MIT License](LICENSE)

## Contributing

Contributions are welcome! Please feel free to submit issues or pull requests.
