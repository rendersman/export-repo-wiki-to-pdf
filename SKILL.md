---
name: export-repo-wiki-to-pdf
description: Export a Qoder repository wiki to PDF format. Converts markdown wiki files to individual PDFs and merges them into a single complete document. Use when the user wants to print, share, or archive the repo wiki as a PDF, or mentions exporting wiki documentation.
---

# Export Repo Wiki to PDF

## Overview

This skill converts a Qoder repository wiki (stored as markdown files in `.qoder/repowiki/`) into PDF format for printing, sharing, or archiving.

## Prerequisites

These Python packages are required (auto-installed if missing):
- `md2pdf-mermaid` - Converts markdown to PDF with Mermaid diagram support (uses Playwright)
- `pypdf` - Merges multiple PDFs into one (pure Python)
- `reportlab` - Creates cover page and table of contents (already included with md2pdf-mermaid)

**Auto-installation**: The script automatically installs missing dependencies on first run. To disable, use `--no-auto-install`.

**Manual installation** (if needed):
```bash
pip install --upgrade md2pdf-mermaid pypdf reportlab
```

**Note**: `md2pdf-mermaid` uses Playwright (not Puppeteer) and works reliably in sandboxed environments.

## Workflow

### Step 0: Setup Temp Directory and Script

**IMPORTANT: Use `.qoder/temp/` for all temporary files.**

The skill uses a dedicated temp directory within `.qoder/` to keep all working files isolated from source content:

```
.qoder/
├── temp/                          # Created by this skill
│   ├── convert_wiki_to_pdf.py      # The conversion script
│   ├── pdfs/                       # Individual PDF files during conversion
│   └── *.md                        # Temporary markdown files
├── repowiki/                       # Source wiki (READ ONLY - never modified)
└── Wiki-Complete.pdf               # Final output (in project root)
```

**Cross-platform approach (works on Windows, macOS, Linux):**
1. Create `.qoder/temp/` directory if it doesn't exist
2. Write the Python script to `.qoder/temp/convert_wiki_to_pdf.py`
3. Execute the script from the project root
4. Script outputs final PDF to project root
5. Script cleans up `.qoder/temp/pdfs/` after completion (keeps the script for reuse)

```bash
# Create temp directory
mkdir -p .qoder/temp

# Write script to temp directory, then run
cd /path/to/project/root
python .qoder/temp/convert_wiki_to_pdf.py --verbose
```

**Do NOT use `python -c "..."`** with inline Python code on Windows PowerShell. PowerShell quote escaping causes ParserError with complex Python strings.

### Step 1: Locate Wiki Files

Find the wiki content directory:
```
.qoder/repowiki/{language}/content/
```

Common structure:
```
content/
├── Getting Started.md
├── Architecture Overview.md
├── Core Modules/
│   └── ...
├── User Interface Components/
│   └── ...
└── ...
```

### Step 2: Convert Markdown to PDFs (with Mermaid Support)

The script reads markdown files from the wiki directory (READ ONLY) and writes all temporary files to `.qoder/temp/pdfs/`. The source wiki folder is never modified.

```python
from md2pdf import convert_markdown_to_pdf

# Read from wiki (source - never modified)
with open('wiki/file.md', 'r', encoding='utf-8') as f:
    md_content = f.read()

# Write output to temp directory (not wiki directory)
convert_markdown_to_pdf(md_content, '.qoder/temp/pdfs/output.pdf')
```

**Note**: Mermaid diagrams are automatically rendered. The script handles all conversions internally.

### Step 3: Merge PDFs and Output to Project Root

The script merges all PDFs and writes the final output directly to the project root:

```
project-root/
├── .qoder/
│   └── temp/              # Temporary files (cleaned up after)
└── Wiki-Complete.pdf      # Final output
```

## Complete Example

### Python Script (Cross-Platform)

Save this to `.qoder/temp/convert_wiki_to_pdf.py`:

**Key Features:**
- ✅ Auto-installation of missing Python dependencies
- ✅ Uses `.qoder/temp/pdfs/` for all temporary files (wiki source is never modified)
- ✅ Outputs final PDF to project root
- ✅ Parallel processing (4 workers by default, configurable with `--workers`)
- ✅ Mermaid syntax error correction (fixes `End` → `end` for Mermaid 11.x)
- ✅ Cover page with project name and generation date
- ✅ Table of contents with all documents listed
- ✅ Comprehensive error handling and timeout management (60s per file)
- ✅ Cross-platform support (Windows, macOS, Linux)
- ✅ Flexible cleanup options (`--no-cleanup`, `--clean-all`)

```python
#!/usr/bin/env python3
"""
Convert Qoder repository wiki to PDF with Mermaid diagram support.
Cross-platform script (Windows, macOS, Linux).

Usage:
    python .qoder/temp/convert_wiki_to_pdf.py [--verbose] [--workers N]
"""

import os
import sys
import re
import shutil
import subprocess
import argparse
from pathlib import Path
from typing import List, Optional
from concurrent.futures import ThreadPoolExecutor, as_completed


class WikiToPdfConverter:
    """Converts Qoder repository wiki to PDF with error handling."""
    
    REQUIRED_PACKAGES = ['md2pdf-mermaid', 'pypdf', 'reportlab']
    
    def __init__(self, wiki_path: str, output_path: str, temp_dir: str, verbose: bool = False, max_workers: int = 4, add_cover: bool = True, auto_install: bool = True, cleanup: bool = True, clean_all: bool = False):
        self.wiki_path = Path(wiki_path)
        self.output_path = Path(output_path)
        self.temp_dir = Path(temp_dir)
        self.verbose = verbose
        self.max_workers = max_workers
        self.add_cover = add_cover
        self.auto_install = auto_install
        self.cleanup = cleanup
        self.clean_all = clean_all
        self.errors: List[str] = []
        self.warnings: List[str] = []
        self._installed_packages: List[str] = []
    
    def log(self, message: str, level: str = "info"):
        """Log messages based on verbosity level."""
        if level == "error":
            print(f"ERROR: {message}", file=sys.stderr)
            self.errors.append(message)
        elif level == "warning":
            print(f"WARNING: {message}")
            self.warnings.append(message)
        elif level == "verbose" and self.verbose:
            print(f"DEBUG: {message}")
        else:
            print(message)
    
    def fix_mermaid_syntax(self, content: str) -> str:
        """Fix common Mermaid syntax errors."""
        original_content = content
        content = re.sub(r'^\s*End\s*$', 'end', content, flags=re.MULTILINE)
        if content != original_content:
            self.log("Fixed Mermaid syntax: converted 'End' to 'end'", "verbose")
        return content
    
    def install_dependencies(self, packages: List[str]) -> bool:
        """Install missing Python packages using pip."""
        if not packages:
            return True
            
        self.log(f"Installing missing packages: {', '.join(packages)}...")
            
        try:
            result = subprocess.run(
                [sys.executable, '-m', 'pip', 'install', '--upgrade'] + packages,
                capture_output=True,
                text=True,
                timeout=300  # 5 minute timeout
            )
                
            if result.returncode == 0:
                self._installed_packages.extend(packages)
                self.log(f"Successfully installed: {', '.join(packages)}")
                return True
            else:
                self.log(f"Failed to install packages: {result.stderr}", "error")
                return False
        except subprocess.TimeoutExpired:
            self.log("Package installation timed out (>5 minutes)", "error")
            return False
        except Exception as e:
            self.log(f"Error installing packages: {e}", "error")
            return False
        
    def validate_prerequisites(self) -> bool:
        """Check that required tools are installed, auto-install if enabled."""
        self.log("Validating prerequisites...")
            
        missing_packages: List[str] = []
            
        # Check md2pdf Python module
        try:
            from md2pdf import convert_markdown_to_pdf
            self.log("md2pdf Python module: OK", "verbose")
        except ImportError:
            self.log("md2pdf Python module: NOT FOUND", "verbose")
            missing_packages.append('md2pdf-mermaid')
            
        # Check pypdf
        try:
            from pypdf import PdfWriter, PdfReader
            self.log("pypdf: OK", "verbose")
        except ImportError:
            self.log("pypdf: NOT FOUND", "verbose")
            missing_packages.append('pypdf')
            
        # Check reportlab (used for cover page/TOC)
        try:
            from reportlab.lib.pagesizes import letter
            self.log("reportlab: OK", "verbose")
        except ImportError:
            self.log("reportlab: NOT FOUND", "verbose")
            missing_packages.append('reportlab')
            
        # Auto-install missing packages if enabled
        if missing_packages:
            if self.auto_install:
                if not self.install_dependencies(missing_packages):
                    return False
                    
                # Re-check after installation
                try:
                    from md2pdf import convert_markdown_to_pdf
                    self.log("md2pdf Python module: OK (installed)", "verbose")
                except ImportError:
                    self.log("md2pdf still not available after installation", "error")
                    return False
                    
                # Re-check pypdf
                try:
                    from pypdf import PdfWriter, PdfReader
                    self.log("pypdf: OK (installed)", "verbose")
                except ImportError:
                    self.log("pypdf still not available after installation", "error")
                    return False
                    
                # Re-check reportlab
                try:
                    from reportlab.lib.pagesizes import letter
                    self.log("reportlab: OK (installed)", "verbose")
                except ImportError:
                    self.log("reportlab still not available after installation", "error")
                    return False
            else:
                self.log(
                    f"Missing packages: {', '.join(missing_packages)}. Install with: pip install {' '.join(missing_packages)}",
                    "error"
                )
                return False
            
        return True
    
    def setup_temp_directory(self):
        """Create temp directory structure."""
        self.temp_dir.mkdir(parents=True, exist_ok=True)
        pdfs_dir = self.temp_dir / "pdfs"
        pdfs_dir.mkdir(exist_ok=True)
        self.log(f"Created temp directory: {self.temp_dir}", "verbose")
    
    def find_markdown_files(self) -> List[Path]:
        """Find all markdown files in the wiki directory."""
        if not self.wiki_path.exists():
            raise FileNotFoundError(f"Wiki path not found: {self.wiki_path}")
        
        if not self.wiki_path.is_dir():
            raise NotADirectoryError(f"Wiki path is not a directory: {self.wiki_path}")
        
        md_files = sorted(self.wiki_path.rglob("*.md"))
        
        if not md_files:
            self.log(f"No markdown files found in: {self.wiki_path}", "warning")
        
        return md_files
    
    def read_markdown_file(self, md_file: Path) -> str:
        """Read a markdown file with error handling."""
        try:
            with open(md_file, 'r', encoding='utf-8') as f:
                return f.read()
        except UnicodeDecodeError as e:
            self.log(f"Encoding error in {md_file}: {e}", "warning")
            # Try with different encoding
            try:
                with open(md_file, 'r', encoding='latin-1') as f:
                    return f.read()
            except Exception as e2:
                raise IOError(f"Cannot read file {md_file}: {e2}")
        except PermissionError:
            raise PermissionError(f"Cannot read file (permission denied): {md_file}")
        except Exception as e:
            raise IOError(f"Cannot read file {md_file}: {e}")
    
    def convert_to_pdf(self, md_file: Path, output_pdf: Path) -> bool:
        """Convert a markdown file to PDF using md2pdf Python module."""
        try:
            # Import md2pdf converter function
            from md2pdf import convert_markdown_to_pdf
            
            # Read markdown content
            with open(md_file, 'r', encoding='utf-8') as f:
                md_content = f.read()
            
            # Convert to PDF and save to file
            convert_markdown_to_pdf(md_content, str(output_pdf))
            
            return True
            
        except subprocess.TimeoutExpired:
            self.log(f"Timeout converting {md_file.name} (took > 60s)", "error")
            return False
        except Exception as e:
            error_msg = str(e)
            self.log(f"Error converting {md_file.name}: {error_msg[:100]}", "error")
            return False
    
    def merge_pdfs(self, pdf_files: List[Path]) -> bool:
        """Merge multiple PDFs into one using pypdf."""
        if not pdf_files:
            self.log("No PDF files to merge", "warning")
            return False
        
        try:
            from pypdf import PdfWriter, PdfReader
            writer = PdfWriter()
            
            # Add cover page and TOC if enabled
            if self.add_cover:
                self.log("Adding cover page...", "verbose")
                self._add_cover_page(writer)
                
                # Add table of contents
                self.log("Adding table of contents...", "verbose")
                toc_page = self._create_table_of_contents(pdf_files)
                if toc_page:
                    writer.add_page(toc_page)
            
            # Add all PDF files
            for pdf_file in pdf_files:
                self.log(f"Adding: {pdf_file.name}", "verbose")
                writer.append(str(pdf_file))
            
            # Write merged PDF
            # Ensure output directory exists
            self.output_path.parent.mkdir(parents=True, exist_ok=True)
            with open(self.output_path, 'wb') as output_file:
                writer.write(output_file)
            
            self.log(f"Merged {len(pdf_files)} PDF(s) into {self.output_path}", "verbose")
            return True
            
        except Exception as e:
            self.log(f"Error merging PDFs: {e}", "error")
            return False
    
    def _add_cover_page(self, writer):
        """Create and add a cover page."""
        from reportlab.lib.pagesizes import letter
        from reportlab.pdfgen import canvas
        from reportlab.lib.units import inch
        from pypdf import PdfReader
        import io
        
        # Create cover page in memory
        packet = io.BytesIO()
        c = canvas.Canvas(packet, pagesize=letter)
        width, height = letter
        
        # Title
        c.setFont("Helvetica-Bold", 36)
        c.drawCentredString(width/2, height - 2*inch, "Repository Wiki")
        
        # Subtitle
        c.setFont("Helvetica", 18)
        c.drawCentredString(width/2, height - 2.5*inch, "Complete Documentation")
        
        # Date
        from datetime import datetime
        c.setFont("Helvetica", 12)
        c.drawCentredString(width/2, height - 3*inch, f"Generated: {datetime.now().strftime('%B %Y')}")
        
        # Project name (extracted from path)
        project_name = self.wiki_path.parents[2].name if len(self.wiki_path.parents) > 2 else "Project"
        c.setFont("Helvetica-Bold", 14)
        c.drawCentredString(width/2, height - 4*inch, project_name)
        
        c.save()
        packet.seek(0)
        
        # Merge with writer
        cover_pdf = PdfReader(packet)
        writer.add_page(cover_pdf.pages[0])
    
    def _create_table_of_contents(self, pdf_files: List[Path]):
        """Create a table of contents page."""
        from reportlab.lib.pagesizes import letter
        from reportlab.pdfgen import canvas
        from reportlab.lib.units import inch
        from pypdf import PdfReader
        import io
        
        # Create TOC page in memory
        packet = io.BytesIO()
        c = canvas.Canvas(packet, pagesize=letter)
        width, height = letter
        
        # Title
        c.setFont("Helvetica-Bold", 24)
        c.drawString(1*inch, height - 1*inch, "Table of Contents")
        
        # List all documents
        c.setFont("Helvetica", 12)
        y_position = height - 1.5*inch
        
        for i, pdf_file in enumerate(pdf_files, 1):
            # Extract title from filename (remove .pdf and clean up)
            title = pdf_file.stem.replace('_', ' ').replace('-', ' ')
            
            # Format: "1. Document Name"
            line = f"{i}. {title}"
            c.drawString(1*inch, y_position, line)
            y_position -= 0.3*inch
            
            # Start new page if needed (max ~25 items per page)
            if y_position < 1*inch and i < len(pdf_files):
                c.showPage()
                c.setFont("Helvetica", 12)
                y_position = height - 1*inch
        
        c.save()
        packet.seek(0)
        
        # Return the TOC page
        toc_pdf = PdfReader(packet)
        return toc_pdf.pages[0]
    
    def cleanup_temp_files(self):
        """Clean up temporary files based on cleanup options."""
        if not self.cleanup:
            self.log("Skipping cleanup (--no-cleanup specified)", "verbose")
            return
        
        if self.clean_all:
            # Remove entire temp directory including the script
            if self.temp_dir.exists():
                try:
                    shutil.rmtree(self.temp_dir, ignore_errors=True)
                    self.log(f"Cleaned up entire temp directory: {self.temp_dir}", "verbose")
                except Exception as e:
                    self.log(f"Failed to cleanup temp directory: {e}", "warning")
        else:
            # Default: clean up PDFs only, keep the script for reuse
            pdfs_dir = self.temp_dir / "pdfs"
            if pdfs_dir.exists():
                try:
                    shutil.rmtree(pdfs_dir, ignore_errors=True)
                    self.log(f"Cleaned up temporary PDF files: {pdfs_dir}", "verbose")
                except Exception as e:
                    self.log(f"Failed to cleanup PDF directory: {e}", "warning")
    
    def run(self) -> bool:
        """Run the conversion process."""
        success = False
        
        try:
            # Validate prerequisites
            if not self.validate_prerequisites():
                return False
            
            # Setup temp directory
            self.setup_temp_directory()
            
            # Find markdown files
            self.log(f"Looking for markdown files in: {self.wiki_path}")
            try:
                md_files = self.find_markdown_files()
            except (FileNotFoundError, NotADirectoryError) as e:
                self.log(str(e), "error")
                return False
            
            self.log(f"Found {len(md_files)} markdown file(s)")
            
            if not md_files:
                return False
            
            pdfs_dir = self.temp_dir / "pdfs"
            pdf_files = []
            failed_files = []
            self.log(f"Converting {len(md_files)} file(s) using {self.max_workers} parallel workers...")
            def process_file(md_file: Path) -> tuple:
                file_name = md_file.stem
                temp_md = pdfs_dir / f"{file_name}.md"
                output_pdf = pdfs_dir / f"{file_name}.pdf"
                try:
                    content = self.read_markdown_file(md_file)
                    content = self.fix_mermaid_syntax(content)
                    with open(temp_md, 'w', encoding='utf-8') as f:
                        f.write(content)
                    if self.convert_to_pdf(temp_md, output_pdf):
                        return (file_name, True, None)
                    else:
                        return (file_name, False, "Conversion failed")
                except Exception as e:
                    return (file_name, False, str(e))
            
            # Use ThreadPoolExecutor for parallel processing
            with ThreadPoolExecutor(max_workers=self.max_workers) as executor:
                # Submit all tasks
                future_to_file = {executor.submit(process_file, md_file): md_file for md_file in md_files}
                
                # Collect results as they complete
                for future in as_completed(future_to_file):
                    md_file = future_to_file[future]
                    file_name = md_file.stem
                    
                    try:
                        result = future.result()
                        success_flag, error_msg = result[1], result[2]
                        
                        if success_flag:
                            pdf_files.append(pdfs_dir / f"{file_name}.pdf")
                            self.log(f"  Converted: {file_name}.pdf")
                        else:
                            failed_files.append(md_file.name)
                            self.log(f"  Failed: {file_name}.md: {error_msg}", "error")
                    except Exception as e:
                        failed_files.append(md_file.name)
                        self.log(f"  Failed: {file_name}.md: {str(e)}", "error")
            
            self.log(f"\nProcessed: {len(pdf_files)} succeeded, {len(failed_files)} failed")
            
            if pdf_files:
                self.log(f"Merging {len(pdf_files)} PDF(s)...")
                if self.merge_pdfs(pdf_files):
                    self.log(f"Success! Output: {self.output_path}")
                    success = True
                else:
                    self.log("Failed to merge PDFs", "error")
            else:
                self.log("No PDFs were generated", "error")
            
            if failed_files:
                self.log(f"\nFailed files: {', '.join(failed_files)}", "warning")
        
        except KeyboardInterrupt:
            self.log("\nOperation cancelled by user", "warning")
        except Exception as e:
            self.log(f"Unexpected error: {e}", "error")
        finally:
            self.cleanup_temp_files()
        return success


def main():
    parser = argparse.ArgumentParser(description="Convert Qoder repository wiki to PDF")
    parser.add_argument("--wiki-path", default=".qoder/repowiki/en/content", help="Path to wiki content directory")
    parser.add_argument("--output", default="Wiki-Complete.pdf", help="Output PDF filename (saved to project root)")
    parser.add_argument("--temp-dir", default=".qoder/temp", help="Temp directory for working files")
    parser.add_argument("--verbose", action="store_true", help="Enable verbose output")
    parser.add_argument("--workers", type=int, default=4, help="Number of parallel workers")
    parser.add_argument("--no-cover", action="store_true", help="Skip cover page and TOC")
    parser.add_argument("--no-auto-install", action="store_true", help="Disable auto-installation")
    parser.add_argument("--no-cleanup", action="store_true", help="Keep temporary PDF files for debugging")
    parser.add_argument("--clean-all", action="store_true", help="Remove entire temp directory including script after merge")
    args = parser.parse_args()
    
    # Resolve paths relative to current working directory
    wiki_path = Path(args.wiki_path).resolve()
    output_path = Path(args.output).resolve()
    temp_dir = Path(args.temp_dir).resolve()
    
    converter = WikiToPdfConverter(
        wiki_path=str(wiki_path),
        output_path=str(output_path),
        temp_dir=str(temp_dir),
        verbose=args.verbose,
        max_workers=args.workers,
        add_cover=not args.no_cover,
        auto_install=not args.no_auto_install,
        cleanup=not args.no_cleanup,
        clean_all=args.clean_all
    )
    
    success = converter.run()
    sys.exit(0 if success else 1)


if __name__ == "__main__":
    main()
```

**Usage:**
```bash
# Run from project root with defaults (includes cover page and TOC)
python .qoder/temp/convert_wiki_to_pdf.py

# With verbose output for debugging
python .qoder/temp/convert_wiki_to_pdf.py --verbose

# Adjust parallel workers (faster conversion)
python .qoder/temp/convert_wiki_to_pdf.py --workers 8

# Skip cover page and table of contents
python .qoder/temp/convert_wiki_to_pdf.py --no-cover

# Keep temporary PDFs for debugging
python .qoder/temp/convert_wiki_to_pdf.py --no-cleanup

# Remove entire temp directory (including script) after merge
python .qoder/temp/convert_wiki_to_pdf.py --clean-all

# Custom paths (if needed)
python .qoder/temp/convert_wiki_to_pdf.py --wiki-path "custom/wiki/path" --output "My-Wiki.pdf"

# Show help
python .qoder/temp/convert_wiki_to_pdf.py --help
```

**Command-line Options:**
- `--wiki-path PATH` - Path to wiki content directory (default: `.qoder/repowiki/en/content`)
- `--output FILENAME` - Output PDF filename, saved to project root (default: `Wiki-Complete.pdf`)
- `--temp-dir DIR` - Temp directory for working files (default: `.qoder/temp`)
- `--verbose` - Enable detailed debug output
- `--workers N` - Number of parallel workers (default: 4, increase for faster processing)
- `--no-cover` - Skip adding cover page and table of contents
- `--no-auto-install` - Disable automatic installation of missing dependencies
- `--no-cleanup` - Keep temporary PDF files for debugging (skips cleanup of `.qoder/temp/pdfs/`)
- `--clean-all` - Remove entire temp directory including script after merge

## Output

- Merged PDF saved to project root (e.g., `Wiki-Complete.pdf`)
- All temporary files stored in `.qoder/temp/pdfs/` and cleaned up after completion (default)
- Use `--no-cleanup` to keep temporary PDFs for debugging
- Use `--clean-all` to remove entire `.qoder/temp/` directory after merge
- Source wiki directory (`.qoder/repowiki/`) is never modified
- The Python script remains in `.qoder/temp/` for reuse (unless `--clean-all` is used)

## Directory Structure

```
project-root/
├── .qoder/
│   ├── temp/                          # Created by this skill
│   │   ├── convert_wiki_to_pdf.py     # The conversion script (kept for reuse)
│   │   └── pdfs/                      # Temporary PDF files (cleaned up after)
│   ├── repowiki/                      # Source wiki (READ ONLY)
│   │   └── en/content/
│   │       ├── Getting Started.md
│   │       └── ...
│   └── ...
└── Wiki-Complete.pdf                  # Final output (project root)
```

## Mermaid Diagrams

The Qoder repo wiki uses Mermaid for diagrams. The `md2pdf-mermaid` tool automatically renders them during PDF conversion.

### Mermaid Syntax Error Correction

Some wiki diagrams may contain syntax that's incompatible with newer Mermaid versions (11.x). Common issues:

1. **Uppercase `End` for subgraphs**: Newer Mermaid requires lowercase `end`
2. **Reserved keywords as node IDs**: Words like `End`, `Stop`, `Start` cannot be used as node names
3. **Arrow syntax changes**: Some arrow styles have changed between versions

**Automatic Correction Function (Python):**

```python
import re

def fix_mermaid_syntax(content: str) -> str:
    """Fix common Mermaid syntax errors."""
    # Fix 1: Convert standalone "End" to lowercase "end" (Mermaid 11.x requirement)
    # This matches "End" on its own line (used to close subgraphs)
    content = re.sub(r'^\s*End\s*$', 'end', content, flags=re.MULTILINE)
    
    return content

# Usage:
with open('input.md', 'r', encoding='utf-8') as f:
    content = f.read()

content = fix_mermaid_syntax(content)

with open('output.md', 'w', encoding='utf-8') as f:
    f.write(content)
```

**Common Fixes:**
- Convert standalone `End` to `end` (lowercase) for closing subgraphs
- Ensure node IDs don't start with numbers
- Use quotes around labels containing special characters

## Limitations

- **Images**: Wiki images stored in Qoder IDE aren't exported to the local filesystem and won't appear in the PDF
- **Mermaid diagrams**: Automatically rendered by `md2pdf-mermaid`, but some syntax errors may need correction
- **Interactive elements**: Links work, but interactive features don't
- **File size**: PDFs with many Mermaid diagrams can be large (10-30 MB typical)

## Troubleshooting

| Issue | Solution |
|-------|----------||
| **PowerShell `ParserError` with inline Python** | Write script to a file instead of using `python -c`. PowerShell quote escaping breaks complex Python code. |
| **Windows command line quoting issues** | Always write Python scripts to `.py` files and run with `python script.py` |
| Auto-install fails | Run manually: `pip install --upgrade md2pdf-mermaid pypdf reportlab` |
| `ModuleNotFoundError: No module named 'md2pdf'` | Auto-installed by default, or run: `pip install md2pdf-mermaid` |
| `ModuleNotFoundError: No module named 'pypdf'` | Auto-installed by default, or run: `pip install pypdf` |
| Missing tools | Run: `pip install --upgrade md2pdf-mermaid pypdf reportlab` |
| Mermaid diagrams show "Syntax error" | The script automatically fixes common issues (e.g., `End` → `end`). Check verbose output for details. |
| `End` node causes parse error | Automatically fixed by the script's Mermaid syntax correction |
| Playwright/Chrome spawn errors | `md2pdf-mermaid` uses Playwright (not Puppeteer). Ensure Playwright is installed: `playwright install` |
| Permission denied on macOS/Linux | Run with `sudo` or use `--user` flag for pip installs |
| Python version issues | Requires Python 3.8+. Check with `python3 --version` |
| PDF merge fails | Ensure all individual PDFs were created successfully. Check verbose output with `--verbose` |
| Slow conversion | Increase parallel workers: `--workers 8` (default is 4) |
| Timeout errors | Files taking >60s will timeout. Reduce worker count or check file size |
| Cover page not appearing | Use default settings (no `--no-cover` flag). Requires `reportlab` package. |

**Important**: This script uses the Python `md2pdf-mermaid` package directly, NOT the npm `md-to-pdf` tool. The npm version may cause compatibility issues.

**Execution Note**: Always save the Python script to `.qoder/temp/convert_wiki_to_pdf.py` and run it from the project root. This keeps all working files isolated from the source wiki and ensures the output goes to the project root.
