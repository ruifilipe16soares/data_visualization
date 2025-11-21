# Data Transformation Scripts

With the database schema now replicated as indices in **Elasticsearch**, and with real-time events being properly mirrored,  
we could already build some basic visualizations to expose our data.  
However, this replication process introduces a few important challenges:

---

## Replication Challenges

- **Elasticsearch does not support relationships between indices**, unlike MySQL which allows table relationships through `JOIN`s.
- Without this capability, it becomes significantly harder to analyze and correlate information coming from the database.

For example:  
Imagine a schema containing two tables — **motorista** (driver) and **veiculo** (vehicle).

---

### Example Tables

#### motoristas

| id | nome           | email                 | telefone       | cnh       | veiculo_id | disponivel | avaliacao_media | criado_em           |
|----|----------------|----------------------|----------------|-----------|-------------|-------------|------------------|---------------------|
| 1  | João Silva     | joao.silva@email.com | (11) 99999-1111 | CNH123456 | 1           | 1           | 4.80             | 2025-11-15 22:47:21 |
| 2  | Maria Santos   | maria.santos@email.com | (11) 99999-2222 | CNH654321 | 2           | 1           | 4.90             | 2025-11-15 22:47:21 |
| 3  | Pedro Oliveira | pedro.oliveira@email.com | (11) 99999-3333 | CNH987654 | 3           | 0           | 4.70             | 2025-11-15 22:47:21 |

#### veiculos

| id | marca   | modelo  | ano  | placa   | cor    | capacidade_passageiros |
|----|---------|----------|------|---------|--------|-------------------------|
| 1  | Toyota  | Corolla  | 2022 | ABC1D23 | Prata  | 4                       |
| 2  | Honda   | Civic    | 2021 | XYZ4E56 | Preto  | 4                       |
| 3  | Hyundai | HB20     | 2023 | DEF7G89 | Branco | 4                       |

---

The **motorista** table has a column `veiculo_id` that references the **veiculo** table.  
If we wanted, for example, to create a chart showing the **top 5 most common car brands among drivers**,  
we would need to combine information from both tables.

In SQL, we could easily achieve this with a `JOIN`:

```sql
SELECT 
    m.nome AS motorista,
    v.marca AS marca_veiculo,
    v.modelo AS modelo_veiculo
FROM motoristas m
INNER JOIN veiculos v ON m.veiculo_id = v.id;
```
---

## Limitation in Elasticsearch

Elasticsearch does not support joins between indices, so this kind of relational query is **not possible directly**.

---

## Why We Need Data Transformation Scripts

To overcome this limitation — and also to address other issues such as:

- Date and timezone formatting  
- Adding computed or derived fields  
- Enriching indices with related information  

—we created **Python transformation scripts** to process and combine data after ingestion.

These scripts make building dashboards much easier and provide cleaner, more meaningful datasets for visualization.

---

### Example

The file **`combine_index.py`** demonstrates a simple example of how this data merging and enrichment can be performed.

---
---

# Hash Mechanism for Incremental Data Transformation

After creating the first version of the transformation scripts, we noticed that on every execution,  
**all documents were being transformed and reindexed**, even those that hadn’t changed since the last run.  
This made the process unnecessarily slower and more resource-intensive.

To solve this, we implemented a **hash-based mechanism** that ensures only **new or modified documents** are processed.

---

## The Problem

- Every execution reprocessed the entire dataset  
- No distinction between old and new documents  
- Unnecessary load on Elasticsearch and CPU

---

## The Solution: Hash-Based Processing

Each document gets a **unique summary** (a hash) generated from its content.  
The script then stores these hashes in a file (`current_hashes.pkl`) to compare them in the next run.

This allows the script to detect which documents have changed or been added since the last execution.

---

## Hash File Structure

At the beginning of each transformation script:

```python
HASH_DIR = os.path.join("hashes", new_index)
os.makedirs(HASH_DIR, exist_ok=True)
CURRENT_HASH_FILE = os.path.join(HASH_DIR, "current_hashes.pkl")
PREVIOUS_HASH_FILE = os.path.join(HASH_DIR, "previous_hashes.pkl")
```

This creates an organized folder structure for each index:

hashes/
└── motoristas/
    ├── current_hashes.pkl
    └── previous_hashes.pkl

---

## Hash Generation Function

The function responsible for generating a SHA-256 hash for each document:

```python
def generate_hash(doc_source):
    doc_json = json.dumps(doc_source, sort_keys=True, default=str)
    return hashlib.sha256(doc_json.encode('utf-8')).hexdigest()
```
Each document is serialized to a sorted JSON string (`sort_keys=True`) and hashed.  
Even the slightest change in data will produce a completely different hash.

---

## Loading Previous Hashes

Before the transformation begins, the script loads the hashes from the previous execution:

- Reads `current_hashes.pkl` (if it exists)
- Renames it as `previous_hashes.pkl` for the next comparison
- Prepares a fresh `current_hashes.pkl` for the current run

---

## Comparing Hashes During Transformation

Within the main transformation loop:

```python
doc_hash = generate_hash(source)
previous_hash = previous_hashes.get(doc["_id"])
current_hashes[doc["_id"]] = doc_hash
if doc_hash == previous_hash:
    continue  # Nothing changed, skip processing
```
---

## Result

- If the hash **matches** the previous one → the document hasn’t changed → **skipped**
- If the hash **differs** → the document is **transformed and reindexed**

This optimization ensures that **only new or updated documents are processed**,  
saving significant **time and computational resources** on each run.


## Automating the Script Execution with Cron

To keep the transformation process always up to date, the script can be scheduled to run **every hour** using the Linux `crontab`.

---

### Open the Crontab Editor

Run the following command in your terminal:

```bash
crontab -e
```
---

### Add the Scheduled Task

Add this line at the end of the file to execute the script every hour:

```bash
0 * * * * /usr/bin/python3 /your/path/scripts/combine_index.py >> /your/path/scripts/combine_index.log 2>&1

This means:
- `0 * * * *` → runs at minute 0 of every hour  
- `/usr/bin/python3` → path to Python interpreter  
- `/your/path/scripts/combine_index.py` → path to your transformation script  
- `>> ...log 2>&1` → saves both output and errors into a log file
```
---

###  Verify Scheduled Tasks

List your active cron jobs:

```bash
crontab -l
```
---

-> From now on, the script will automatically run **every hour**, checking for new or modified data, transforming it, and indexing updates into Elasticsearch.
-> With the new indices created, we can now build data views in Kibana based on these indices.
These data views act as the foundation for creating rich visualizations and interactive dashboards,
allowing us to explore and analyze the transformed data through charts, tables, and filters tailored to our needs.

---

## Example Visualization

Below is an example of one of the visualizations created in Kibana,  
built using the transformed data and the defined data views:

![Example Kibana Visualization](/mnt/data/example_grafic.PNG)

