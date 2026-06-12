---
title: "PDF Search and Download — Research Note"
date: 2026-06-11
tags: [python, internet-archive, pdfs, automation]
---

# PDF Search and Download — Research Note

## What it does

This pipeline uses Python to search the Internet Archive for a list of book
titles, check whether matching records exist, and download any available PDF
files.

## What's in place

Two workflows are recorded below:

1. **One title at a time** — search a title, get its identifier, and download
   the PDF. Takes the best (first) match. Good for exploring a single title by
   hand.
2. **Batch pipeline** — run a whole list of titles end to end and download the
   PDFs. Also takes the best (first) match per title, the same way, just across
   a list.

## Workflow 1 — One at a time

### 1. Search a title → get the best match

Search by title, list the matches so you can see them, then keep the first
(best) one — same as the batch pipeline. `identifier` is what the next steps
use — no copying and pasting.

```python
from internetarchive import get_session

session = get_session()

search = session.search_items(
    'title:"Historiæ ecclesiasticæ gentis Anglorum libri quinque"'
)

items = list(search.iter_as_items())

for n, item in enumerate(items, start=1):
    print(f"[{n}] identifier: {item.identifier}")
    print(f"    title:      {item.metadata.get('title')}")

# keep the first (best) match for the next steps
identifier = items[0].identifier
print("\nUsing identifier:", identifier)
```

### 2. Find the PDF file for that identifier

Reuse `identifier` to look up the item and grab the name of its first PDF.

```python
item = session.get_item(identifier)

pdf_name = next(
    (f["name"] for f in item.files if f["name"].endswith(".pdf")),
    None,
)

print("PDF file:", pdf_name)
```

### 3. Download the PDF (renamed to `Title (Year).pdf`)

Internet Archive saves PDFs under their original (often cryptic) filenames, so
download the file, then rename it using the item's title and year.

```python
import os
import re

def pdf_filename(title, year):
    """Build a clean 'Title (Year).pdf' name, safe for the filesystem."""
    title = re.sub(r'[\\/:*?"<>|]', "", str(title)).strip()
    title = re.sub(r"\s+", " ", title)
    year = str(year).strip()
    return f"{title} ({year}).pdf" if year else f"{title}.pdf"

new_name = pdf_filename(item.metadata.get("title", identifier),
                        item.metadata.get("year", ""))

os.makedirs("downloaded_pdfs", exist_ok=True)
item.download(files=[pdf_name], destdir="downloaded_pdfs",
              no_directory=True, verbose=True)

os.rename(
    os.path.join("downloaded_pdfs", pdf_name),
    os.path.join("downloaded_pdfs", new_name),
)
print("Saved as:", new_name)
```

## Workflow 2 — Batch pipeline

For a batch of texts, this is the cleaner approach: search the Internet Archive
over HTTP, collect the best match for each title into a pandas DataFrame, then
download PDFs for every title that was found.

### 1. Imports and title list

```python
import os
import time
import requests
import pandas as pd
from internetarchive import get_session

titles = [
    "Memorials of Saint Dunstan, Archbishop of Canterbury, edited from various manuscripts",
    "Cædmon manuscript: Cædmon’s metrical paraphrase of parts of the Holy Scriptures",
    "Catalogue général des manuscrits des bibliothèques publiques de France",
    "Ueber den Hymnus Cædmons",
    "Grundriss zur Geschichte der angelsächsischen Litteratur",
    "Über den Hymnus Cädmons",
    "Alt- und mittelenglisches Übungsbuch zum Gebrauche bei Universitäts-Vorlesungen",
    "Early English literature (to Wiclif)",
]
```

### 2. Search function

Query the advanced-search endpoint and return the best (first) match, or `None`.

```python
def search_archive(title):
    url = "https://archive.org/advancedsearch.php"

    params = {
        "q": f'title:"{title}"',
        "fl[]": ["identifier", "title"],
        "rows": 5,
        "output": "json",
    }

    r = requests.get(url, params=params)
    docs = r.json()["response"]["docs"]

    if not docs:
        return None

    return docs[0]  # best match
```

### 3. Check all titles → DataFrame

Run every title through the search and collect the results into a DataFrame.

```python
results = []

for t in titles:
    match = search_archive(t)

    if match:
        results.append({
            "title": t,
            "found": True,
            "identifier": match["identifier"],
            "archive_title": match.get("title", ""),
        })
    else:
        results.append({
            "title": t,
            "found": False,
            "identifier": None,
            "archive_title": None,
        })

    time.sleep(0.2)  # be polite to the API

df = pd.DataFrame(results)
df  # display in a notebook
```

### 4. Download PDFs for found titles

Iterate over the DataFrame and download every PDF for each matched item.

```python
session = get_session()
os.makedirs("downloads", exist_ok=True)

for _, row in df.iterrows():
    if not row["found"]:
        continue

    item_id = row["identifier"]
    print("Downloading:", item_id)

    item = session.get_item(item_id)

    for f in item.files:
        name = f["name"]

        if name.endswith(".pdf"):
            item.download(
                files=[name],
                destdir=f"downloads/{item_id}",
            )
```
