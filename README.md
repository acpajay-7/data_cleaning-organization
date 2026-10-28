# Enterprise B2B & MCA Director Data Pipeline
## Cleaning, Normalization & Standardization

A Python ETL pipeline for ingesting, standardizing, imputing, and cleaning B2B enterprise datasets, IT industry leadership lists, and director databases registered with India's Ministry of Corporate Affairs (MCA).

The pipeline uses Pandas and Excel-processing libraries to improve data consistency and prepare `.xlsx` datasets for analysis, deduplication, search, and downstream CRM workflows.

## 1. Project Overview

Raw corporate and director datasets sourced from business directories and statutory registries can contain structural and semantic issues, including:

- **Completely empty columns:** Tracking metadata such as `Email Confidence`, `Tertiary Email`, `Other Phone`, and `Qualify Contact` may contain 100% missing values.
- **Inconsistent headers:** Column names may contain mixed casing, punctuation, or extra whitespace, such as `Company Name for Emails` and `Director Email Address `.
- **Unclean phone numbers and missing values:** Phone fields may contain leading quotes, inconsistent formatting, or missing corporate contact numbers.
- **Mixed casing and whitespace:** Names, company names, cities, and email IDs may have inconsistent capitalization or leading/trailing spaces.
- **Missing financial metrics:** MCA records may have missing `Authorized Capital` and `Paid Capital` values.
- **Identifier formatting:** CIN, DIN, and mobile fields may require careful type handling to preserve their values.

This pipeline automates column pruning, `snake_case` header normalization, configurable missing-value imputation, text normalization, and standardized Excel exports.

> **Important:** Placeholder values such as `unknown`, `na`, and `0000000000` represent missing information; they do not verify that a contact detail is real or current.

## 2. Processed Datasets and Schemas

| Source dataset | Domain / entities | Key transformations | Export target |
|---|---|---|---|
| `indian_businessman_2026.xlsx` | Business owners and founders | Removed completely empty columns (reported as 7); removed redundant `mobile_phone`; imputed selected missing values; normalized text | `cleaned_data4.xlsx` |
| `it_information_technology_39185.xlsx` | IT executives and C-suite contacts (approximately 39k+ rows, based on the supplied description) | Standardized headers to `snake_case`; converted `mobile` to string where applicable; stripped whitespace; lowercased text fields | `cleaned_data5.xlsx` |
| `mca_companies_directors_mobile_database.xlsx` | MCA-registered companies and directors (approximately 1.5k rows, based on the supplied description) | Filled missing `authorized_capital` and `paid_capital` values with `unknown`; normalized CIN, DIN, names, addresses, and email fields | `cleaned_data6.xlsx` |

*Verify record counts and transformation summaries against the actual source files and notebook output before publishing.*

## 3. Data Cleaning Pipeline Architecture

```text
Raw Excel Files (.xlsx)
        |
        v
[1. Null-Column Pruning]
        |
        |-- Drop columns where all values are missing
        v
[2. Header Standardization]
        |
        |-- Trim spaces, lowercase, replace spaces with underscores
        |-- Remove unwanted punctuation
        v
[3. Domain-Specific Imputation]
        |
        |-- Phone fields: configured placeholder values
        |-- Geography fields: "unknown"
        |-- Capital fields: "unknown" or an explicitly defined fallback
        v
[4. Text and Type Normalization]
        |
        |-- Strip leading/trailing whitespace
        |-- Normalize selected text fields to lowercase
        |-- Preserve identifier fields as strings where possible
        v
[5. Excel Export]
        |
        v
Cleaned Enterprise Datasets (.xlsx)
```

## 4. Key Implementation Details

### A. Dynamic Pruning of Completely Empty Columns

Removes empty columns without hardcoding their names.

```python
df = df.dropna(axis=1, how="all")
```

This retains columns that contain at least one non-missing value.

### B. Universal Column Normalization

Converts multi-word headers into consistent programmatic column names.

```python
df.columns = (
    df.columns
    .str.strip()
    .str.lower()
    .str.replace(r"\s+", "_", regex=True)
    .str.replace(r"[^\w]", "", regex=True)
)
```

Example: `Company Name for Emails` becomes `company_name_for_emails`.

Check for duplicate column names after normalization, because different original headers can sometimes normalize to the same name.

### C. Contextual Missing-Value Imputation

Applies configured fallback values instead of automatically dropping records with incomplete contact information.

```python
imputation_rules = {
    "corporate_phone": "unknown",
    "company_phone": "unknown",
    "state": "unknown",
    "company_city": "unknown",
    "company_state": "unknown",
    "company_country": "unknown",
    "secondary_email_status": "na",
    "secondary_email_verification_source": "na",
    "authorized_capital": "unknown",
    "paid_capital": "unknown",
}

for col, fill_value in imputation_rules.items():
    if col in df.columns:
        df[col] = df[col].fillna(fill_value)
```

Only columns present in the DataFrame are modified. Adjust the rules to match each dataset's schema and downstream requirements.

**Data quality note:** Text placeholders are generally safer than a fake numeric-looking phone number, which may be mistaken for a real contact detail.

### D. Universal Text Normalization

Strips surrounding whitespace and lowercases string values.

```python
string_cols = df.select_dtypes(include=["object", "string"]).columns

for col in string_cols:
    df[col] = df[col].apply(
        lambda value: value.strip().lower()
        if isinstance(value, str)
        else value
    )
```

Apply lowercase normalization selectively when original capitalization must be retained for display or legal records.

### E. Preserve Identifiers and Contact Numbers

Mobile numbers, CINs, and DINs are generally best stored as strings to preserve their representation.

```python
identifier_cols = ["mobile", "mobile_phone", "cin", "din"]

for col in identifier_cols:
    if col in df.columns:
        df[col] = df[col].astype("string").str.strip()
```

If Excel values were already imported as floating-point numbers, converting them to strings may not restore lost leading zeros or original formatting. Import identifier columns as strings from the source whenever possible.

### F. Export Cleaned Excel Files

```python
from pathlib import Path

output_dir = Path("data/cleaned")
output_dir.mkdir(parents=True, exist_ok=True)

df.to_excel(output_dir / "cleaned_output.xlsx", index=False)
```

Use the appropriate output filename for each dataset, such as `cleaned_data4.xlsx`, `cleaned_data5.xlsx`, or `cleaned_data6.xlsx`.

## 5. Repository Structure

```text
.
├── data/
│   ├── raw/
│   │   ├── indian_businessman_2026.xlsx
│   │   ├── it_information_technology_39185.xlsx
│   │   └── mca_companies_directors_mobile_database.xlsx
│   └── cleaned/
│       ├── cleaned_data4.xlsx
│       ├── cleaned_data5.xlsx
│       └── cleaned_data6.xlsx
├── notebooks/
│   └── data_cleaning_and_imputation.ipynb
├── requirements.txt
└── README.md
```

This is a suggested structure. Update it to match the actual files and folders in the repository.

## 6. Setup and Installation

### Prerequisites

- Python 3.10 or newer
- Google Colab, Jupyter Notebook, or VS Code with the Jupyter extension
- Pandas
- OpenPyXL

### Install Dependencies

```bash
pip install pandas openpyxl jupyter
```

For legacy `.xls` files, install `xlrd` if required:

```bash
pip install xlrd
```

A minimal `requirements.txt` could contain:

```text
pandas
openpyxl
jupyter
```

Add `xlrd` only if the source files require it.

## 7. Run the Pipeline

1. Clone or download the repository.
2. Place the original Excel files in `data/raw/`.
3. Install the required dependencies.
4. Open `notebooks/data_cleaning_and_imputation.ipynb` in Google Colab, Jupyter Notebook, or VS Code.
5. Check and update the input and output paths if necessary.
6. Execute all notebook cells sequentially.
7. Review the generated files in `data/cleaned/`.
8. Validate the output before using it for analytics or outreach.

To start Jupyter locally:

```bash
jupyter notebook
```

## 8. Validation Checklist

Before using the cleaned datasets, verify:

- [ ] Completely empty columns have been removed.
- [ ] Column names are standardized and unique.
- [ ] Missing-value placeholders are consistent.
- [ ] Phone numbers and statutory identifiers have been checked for unintended changes.
- [ ] Duplicate records have been assessed using suitable business keys.
- [ ] Row counts before and after cleaning have been compared.
- [ ] Excel exports open correctly and contain the expected columns.
- [ ] Sample records have been checked against the source data.

## 9. Data Quality and Responsible Use

- Imputation does not verify whether a contact detail is correct or current.
- Do not treat placeholder phone numbers or email values as real contact information.
- Keep raw source files unchanged and write cleaned data to separate files.
- Retain source references and document transformations where possible.
- Use business and director contact information responsibly and comply with applicable privacy, data-protection, and outreach requirements.

## 10. Future Improvements

Potential improvements include:

- Automated duplicate detection with configurable deduplication keys.
- Phone-number and email-format validation.
- Data-quality reports before and after cleaning.
- Logging and exception handling for invalid input files.
- Dataset-specific configuration files for imputation rules.
- Batch processing for all Excel files in `data/raw/`.
- Unit tests for header normalization, imputation, and export logic.

## 11. License and Data Attribution

Add a license file if you intend to distribute this project. Confirm that you have permission to process and share each source dataset, and document its origin and any applicable usage restrictions.
