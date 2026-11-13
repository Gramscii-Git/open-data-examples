# Gramscii European Open Data Catalogue

A catalogue of 78,150 public datasets from 34 European, international and Italian
providers: what each dataset is, how it is structured, which codes and labels it
uses in each published language, where it can be read, and under which terms.
It is built by Gramscii from the providers' own catalogues and structure
services, and published on AWS as Apache Parquet tables.

The catalogue holds metadata only. It contains no observation values, feature
geometries, rasters or granules, and no search vectors: values stay with the
original provider, which each entry names and links.

- **Bucket:** `s3://gramscii-european-open-data-catalogue` (region `eu-central-1`),
  public and readable without an AWS account.
- **Same release on Hugging Face:** [Gramscii-IT/european-open-data-catalogue](https://huggingface.co/datasets/Gramscii-IT/european-open-data-catalogue),
  as one verified archive with a browsable viewer.
- **Publisher and release checks:** [Gramscii-Git/open-data-catalogue](https://github.com/Gramscii-Git/open-data-catalogue).
- **Tutorial:** [Get To Know A Dataset](get-to-know-a-dataset.ipynb).

## Providers

| Group | Providers (partition ids) |
| --- | --- |
| Official statistics | ISTAT (`istat`), Eurostat (`eurostat`, `eurostat-comext`), the European Commission's DG EMPL (`ec-empl`) and DG GROW (`ec-grow`), OECD (`oecd`), ILO (`ilo`), ECB (`ecb`), BIS (`bis`), Deutsche Bundesbank (`bundesbank`), INSEE (`insee`), FAO (`fao`) |
| Italian public finance | Dove vanno i nostri soldi (`dvns`) and AgID's Cruscotto Italia municipal indicators (`cruscotto`) |
| National open data portals | data.gouv.fr (`datagouv`) |
| Italian cities | Milano (`milano`), Roma (`roma`), Torino (`torino`), Bologna (`bologna`), Napoli (`napoli`) |
| Urban and territorial planning | Comune di Milano PGT (`milano-pgt`), Regione Lombardia PGT (`lombardia-pgt`), Città metropolitana di Torino PRGC (`torino-prgc`), Provincia autonoma di Bolzano (`bolzano-puc`), Comune di Genova PUC (`genova-puc`), Regione del Veneto PTRC (`veneto-ptrc`), Regione Autonoma della Sardegna's municipal plans (`sardegna-puc`), Regione Siciliana PRG (`sicilia-prg`) |
| Sardinia | Sardinia's datasets on dati.gov.it (`datigov-sardegna`), Sardegna Salute (`sardegna-salute`), Sardegna Autonomie (`sardegna-autonomie`), the Region's GeoServer layers (`sardegna-geoserver`), GBIF's occurrence datasets (`gbif-sardegna`) and the European Environment Agency's map services over the island (`eea-sardegna`) |

The manifest of each release counts the datasets of every provider.

## Bucket layout

```
s3://gramscii-european-open-data-catalogue/
└── catalogue/
    ├── manifest.json
    ├── opendata_catalog/
    │   ├── provider=istat/data_0.parquet
    │   ├── provider=eurostat/data_0.parquet
    │   └── provider=…/data_0.parquet
    ├── opendata_structures/provider=…/data_0.parquet
    ├── opendata_structure_dims/provider=…/data_0.parquet
    ├── opendata_terms/provider=…/data_0.parquet
    ├── opendata_labels/provider=…/data_0.parquet
    ├── opendata_meta_reports/provider=…/data_0.parquet
    ├── opendata_document_codes/provider=…/data_0.parquet
    ├── opendata_documents/provider=…/data_0.parquet
    ├── opendata_native_objects/data_0.parquet
    ├── opendata_native_bindings/provider=…/data_0.parquet
    ├── opendata_native_exclusions/provider=…/data_0.parquet
    └── opendata_held_codes/provider=…/data_0.parquet
```

Every table keyed by provider is partitioned the Hive way, `provider=<id>`: a
reader that needs one provider reads only that provider's files, and tools that
understand Hive partitioning (DuckDB, Amazon Athena, Apache Spark, pandas through
PyArrow) expose `provider` as a column. Files are compressed with Zstandard.

`catalogue/manifest.json` describes the release:

| Field | Meaning |
| --- | --- |
| `taken_at` | When the catalogue snapshot was taken (UTC). |
| `archive` | SHA-256 and size of the verified archive on Hugging Face that these tables were written from. |
| `document_contract_sha256` | The contract the searchable descriptions were built under. |
| `tables` | For every table: its rows, its partition column, and each column with its type. |
| `files` | Every Parquet file with its size and SHA-256. |

Each release replaces the previous one. Read `manifest.json` first and check the
files you read against it; for a release that never changes, pin a commit of the
Hugging Face repository.

## Tables

One row of each table is one row of the catalogue's database, with the same keys.

| Table | One row for | Key |
| --- | --- | --- |
| `opendata_catalog` | each dataset: titles and names per language, descriptions, categories, keywords, agency, version, period covered, last update, territorial level, links (`sources`), licence and attribution, and whether its structure is available | `provider`, `dataset_id` |
| `opendata_structures` | each dataset's structure: its dimensions with their code lists or classification fields, time coverage, key order, observation or feature count, spatial reference and extent, the source it was read from, and the provider's answer when it could not be read (`error`) | `provider`, `dataset_id` |
| `opendata_structure_dims` | each dimension of a data structure: the concept it measures and the code list it draws from | `provider`, `structure_id`, `dimension_id` |
| `opendata_terms` | each code of a code list or concept scheme, in each published language | `provider`, `language`, `scope`, `code` |
| `opendata_labels` | each dataset's labels in one language | `provider`, `dataset_id`, `language` |
| `opendata_meta_reports` | each provider metadata report (methodology and quality texts) | `provider`, `report_key` |
| `opendata_document_codes` | each dimension a dataset's searchable description names, with the codes it names | `provider`, `dataset_id`, `dimension_id` |
| `opendata_documents` | each dataset's searchable description in one language, with its provenance (`projection`) | `provider`, `dataset_id`, `language` |
| `opendata_native_objects` | each provider response kept verbatim (the SDMX structure messages of ISTAT and Eurostat), addressed by the SHA-256 of its bytes | `sha256` |
| `opendata_native_bindings` | each dataset read from native objects: which structure and flow it was read from, and a receipt (URL, status, time, size, SHA-256) for every response | `provider`, `dataset_id` |
| `opendata_native_exclusions` | each native reference a provider lists that the catalogue leaves out, with the reason | `provider`, `reference` |
| `opendata_held_codes` | each dimension of a dataset: the codes that hold data, when that was checked and where it was learnt | `provider`, `dataset_id`, `dimension_id` |

Column types follow the database: text, booleans, integers, dates, timestamps
(stored as UTC instants), lists of text (`codes`), binary (`body`) and JSON. JSON
columns hold nested metadata exactly as the provider described it, so its shape can
differ between providers; read them with a JSON function rather than expecting a
fixed schema. `manifest.json` lists every column with its type.

Provider identifiers and dataset keys are the providers' own. `opendata_catalog`
links to the other tables through `provider` and `dataset_id`, and to a dataset's
code lists through `structure_id` and `opendata_structure_dims`.

## Reading the catalogue

With [DuckDB](https://duckdb.org), no download and no AWS account:

```sql
INSTALL httpfs;
LOAD httpfs;
SET s3_region = 'eu-central-1';

SELECT provider, count(*) AS datasets
FROM read_parquet('s3://gramscii-european-open-data-catalogue/catalogue/opendata_catalog/*/*.parquet',
                  hive_partitioning = true)
GROUP BY provider
ORDER BY datasets DESC;
```

One provider, by its partition:

```sql
SELECT dataset_id, title, licence, period_start, period_end
FROM read_parquet('s3://gramscii-european-open-data-catalogue/catalogue/opendata_catalog/provider=istat/*.parquet');
```

With Python and PyArrow:

```python
import pyarrow.dataset as ds
from pyarrow import fs

s3 = fs.S3FileSystem(anonymous=True, region="eu-central-1")
catalogue = ds.dataset("gramscii-european-open-data-catalogue/catalogue/opendata_catalog",
                       filesystem=s3, format="parquet", partitioning="hive")
istat = catalogue.to_table(filter=ds.field("provider") == "istat",
                           columns=["dataset_id", "title", "licence"]).to_pandas()
```

With the AWS CLI:

```sh
aws s3 ls --no-sign-request s3://gramscii-european-open-data-catalogue/catalogue/
aws s3 cp --no-sign-request s3://gramscii-european-open-data-catalogue/catalogue/manifest.json .
```

## Quality

A release is published only after it passes the publisher's quality gate: every
provider within its declared scope, no structure, licence or searchable description
missing, and every document built under the declared contract. Structure errors a
provider's own contract declares permanent are published with the provider's
answer. The Parquet tables are written from the archive the Hugging Face read-back
verified, and every table is read back and compared with that archive before
upload. Passing the gate does not guarantee complete provider coverage or the
accuracy of the source data.

## Rights and attribution

Each dataset keeps its provider's terms, recorded in `opendata_catalog`'s
`licence`, `attribution` and `sources` fields; consult the original provider's
terms before reusing a dataset. Missing metadata is reported by the quality checks,
never replaced by an assumed licence. Gramscii's compilation and this documentation
are released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/); those
terms cover Gramscii's contribution, not the underlying sources.

The catalogue draws on the `istat_lookup.duckdb` territorial lookup of
[onData's ISTAT MCP Server](https://github.com/ondata/istat_mcp_server) and on the
[opensdmx](https://github.com/ondata/opensdmx) constraint archive, under their MIT
licence; thanks to Vincenzo Patruno and onData.

## Updates and contact

Each verified release replaces the previous one, about once a month. Questions and corrections: open an issue in this repository or write
to amministrazione@gramscii.com.
