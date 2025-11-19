# Task Creation Guide for DataLakeBench

This guide explains how to author high‑quality tasks for DataLakeBench. Keep questions explicit in constraints, short in answers, and fully verifiable from the listed structured and unstructured data files.

## Overview

DataLakeBench evaluates multi‑hop reasoning over a data lake of ~46,000 Wikipedia structured data files (e.g., tables) and unstructured data files (e.g., passages). Good tasks require agents to:
- Search and retrieve relevant data files from the data lake
- Cross‑reference multiple files (**minimum 5+ hops required**)
- Follow sequential dependencies (cannot parallelize all steps)
- Produce a short, normalized final answer

## Data Source Access

- Bucket: `s3://dataseekplatform/`
- Access: Public; no AWS credentials needed

CLI
```bash
# List (public, unsigned)
aws s3 ls s3://dataseekplatform/ --no-sign-request
# Download
aws s3 cp s3://dataseekplatform/csv_table_13227_New_Zealand_at_the_2012_Summer_Olympics_0.csv . --no-sign-request
# Quick keyword search
aws s3 ls s3://dataseekplatform/ --no-sign-request | grep -i "olympic"
```

Python
```python
import boto3
from botocore import UNSIGNED
from botocore.config import Config

s3 = boto3.client('s3', config=Config(signature_version=UNSIGNED))
resp = s3.list_objects_v2(Bucket='dataseekplatform', MaxKeys=10)
for obj in resp.get('Contents', []):
    print(obj['Key'])

s3.download_file('dataseekplatform',
                 'csv_table_13227_New_Zealand_at_the_2012_Summer_Olympics_0.csv',
                 'local.csv')
```

Current file naming convention examples:
- Structured data files (e.g., CSV tables): `csv_table_{TABLE_ID}_{WIKIPEDIA_PAGE_NAME}_{TABLE_INDEX}.csv`
- Unstructured data files (e.g., TXT passages): `txt_text_{TEXT_ID}_{WIKIPEDIA_PAGE_NAME}_{TEXT_INDEX}.txt`

## Core Principles

- Short, structured answers:
  - End questions with a schema: “Answer as: [field1]; [field2]; $[amount].”
  - Keep the answer one line; avoid prose.
- Verifiable and minimal:
  - Every token in the answer must be checkable from `required_files`.
  - Include the minimal sufficient set of structured (e.g., CSV) or unstructured (e.g., TXT) data files.
- Deterministic reasoning:
  - Steps should be factual and reproducible, not subjective.

## Task JSON Format

Each task follows this structure with a complete example demonstrating all requirements:

```json
{
  "name": "Complex Nested Reasoning - Chained Olympic and Film Query",
  "questions": [
    {
      "question": "In the 2012 Summer Olympics, find the sport where both New Zealand and Netherlands won gold medals. Then, identify the month when New Zealand won its gold medal in that sport. Among Australian films released in that month, find those containing 'horror' in their genre. Of these horror films, which one was released earliest? This film has multiple genres - identify all of them. Does this film include 'Comedy' as one of its genres? If yes, find all films in that same month that contain 'Comedy' in their genre, sort them by release date, and identify the film released last. Who directed this latest comedy film? Count the number of words in this director's full name. Answer as: [number]",
      "answer": "2",
      "required_files": [
        "csv_table_13227_New_Zealand_at_the_2012_Summer_Olympics_0.csv",
        "txt_text_13227_New_Zealand_at_the_2012_Summer_Olympics_0.txt",
        "csv_table_13188_Netherlands_at_the_2012_Summer_Olympics_0.csv",
        "txt_text_13188_Netherlands_at_the_2012_Summer_Olympics_0.txt",
        "csv_table_5689_List_of_Australian_films_of_2012_0.csv",
        "txt_text_5689_List_of_Australian_films_of_2012_0.txt"
      ],
      "reasoning": [
        "Step 1: Load New Zealand 2012 Olympics data and filter for gold medals",
        "Step 2: Load Netherlands 2012 Olympics data and filter for gold medals",
        "Step 3: Find intersection of sports where both won gold → 'Sailing'",
        "Step 4: Extract the date when NZ won gold in Sailing → '10 August'",
        "Step 5: Parse the month from the date → 'August'",
        "Step 6: Load Australian films from 2012",
        "Step 7: Filter films released in August 2012",
        "Step 8: Filter for films containing 'horror' in Genre column → 2 films found",
        "Step 9: Sort horror films by release date and select earliest",
        "Step 10: Earliest horror film → '100 Bloody Acres' (4 August 2012)",
        "Step 11: Extract Genre field → 'Comedy, horror'",
        "Step 12: Parse genres and check if 'Comedy' is present → Yes",
        "Step 13: Filter all August films for those containing 'Comedy' in genre",
        "Step 14: Sort comedy films by release date → 3 films found",
        "Step 15: Select the film with latest release date → 'Save Your Legs!' (19 August)",
        "Step 16: Extract Director field → 'Boyd Hicklin'",
        "Step 17: Count words in director's name (split by spaces) → 2"
      ]
    }
  ]
}
```

**Key requirements:**

- **`question`**: Must be written in natural language and specify a final answer schema (e.g., "Answer as: [field]; $[amount]"). The question should chain multiple reasoning steps, requiring results from multiple tables to derive subsequent queries.
- **`answer`**: Must be short, normalized, ASCII, and unambiguous, matching the specified schema
- **`required_files`**:
  - Structured (CSV) or unstructured (TXT) data files from `s3://dataseekplatform/`
  - **Minimum 5+ files required** to maintain multi-hop complexity
  - List only files necessary to derive the answer; keep minimal but sufficient
  - Verify all listed files exist in the data lake
- **`reasoning`**:
  - **Minimum 5+ sequential steps** that depend on previous results
  - Each step should be based on factual data with a closed-form solution (not subjective interpretation)
  - Steps cannot be fully parallelized—they must build on each other, often requiring combining results from multiple tables


## Evidence (`required_files`) Guidelines

- Structured (e.g., CSV) or unstructured (e.g., TXT) data files only from the data lake
- List each file that is necessary to derive the answer; keep minimal
- Verify all listed files exist in `s3://dataseekplatform/`
- **Ensure at least 5+ files are required** to maintain multi-hop complexity

## Reasoning Steps

Write a sequential plan with **minimum 5+ steps** that a strong agent can follow. Steps must be:
- **Deterministic**: Based on factual data, not subjective interpretation
- **Sequential**: Each step depends on previous results (cannot be fully parallelized)

## Common Pitfalls

- **Too explicit**: Avoid giving column names and step‑by‑step hints inside the question
- **Parallelizable chains**: Ensure at least one step depends on the previous result
- **Ambiguous answers**: Constrain years, roles, and positions to yield a single answer; use deterministic tie-breakers (e.g., "earliest", "alphabetically first", "most")
- **Insufficient complexity**: Ensure questions require **minimum 5+ hops** across multiple data files
- **Wrong file types**: Only include structured (e.g., CSV) or unstructured (e.g., TXT) data files in `required_files`
- **Open-world ambiguity**: Avoid questions where different valid data sources could yield different answers; use specific constraints or aggregations (counts, intersections)

Happy task creation!
