# AWS SageMaker Lessons for IMU

This repo documents key workflows for working in **AWS SageMakerAI**,
including setting up new spaces, managing environments, linking GitHub,
and querying Snowflake.

------------------------------------------------------------------------

## 📕 How to Set Up a New Space and Find MFIs

### 🔑 Key Concepts

-   **SageMaker as a Cloud Jupyter Environment**
    -   Managed JupyterLab environments (called *spaces*).\
    -   Each space = a separate EC2 instance with configurable specs.
-   **Public vs. Private Spaces**
    -   Private → visible only to you (recommended).\
    -   Public → visible to team.\
    -   Visibility cannot be changed after creation.
-   **Instance Configuration**
    -   Memory: 4 GB → 32 GB\
    -   GPU: optional\
    -   Disk space: **5 GB default**, max **100 GB**.\
    -   Limitation: large datasets may exceed disk limits.
-   **Filesystem Model**
    -   Each space has its own filesystem.\
    -   Shared persistence via `user/default/efs`.

------------------------------------------------------------------------

### 🧭 Creating a New SageMaker Space

1.  **Log in & Launch Notebook**
    -   Navigate to **Studio Applications and IDEs**.\
    -   Choose domain (e.g., `data-science`).\
    -   Launch **JupyterLab** or **RStudio**.
2.  **Create New Space**
    -   Click *Create Jupyter Lab Space*.\
    -   Configure:
        -   **Name**: e.g. `paul32`\
        -   **Visibility**: private\
        -   **Instance type**: e.g. `ml.m5.4xlarge` (32 GB)\
        -   **Disk space**: 100 GB (recommended)\
        -   **Runtime image**: latest
3.  **Start/Stop Instances**
    -   Stop unused spaces to avoid billing.
4.  **Shared Filesystem**
    -   Store persistent files in `user/default/efs`.

------------------------------------------------------------------------

### 🧩 Best Practices & Warnings

-   ✅ Always stop unused SageMaker spaces.\
-   ✅ Use `efs` folder for cross-space persistence.\
-   ✅ Set disk to **100 GB**, not default 5 GB.\
-   ⚠️ Package access passwords expire quickly.\
-   ⚠️ Notebook kernels must be registered to use custom envs.

------------------------------------------------------------------------

### 🖼️ SageMaker Studio UI

-   Launcher → terminal, notebooks, etc.\
-   Spaces selectable via dashboard.\
-   Upload/download buttons for file transfer.\
-   Kernels listed by display name.\
-   Shared filesystem appears as `user/default/efs`.

------------------------------------------------------------------------

## 💾 Linking SageMaker Studio to GitHub

``` bash
# Create a PAT token (per session, not per space)
cd /home/sagemaker-user

# Store credentials globally
git config --global credential.helper store

# Clone repo into shared EFS folder
cd ~/user-default-efs/
git clone https://github.com/your-username/your-repo-name.git
```

------------------------------------------------------------------------

## 📗 Virtual Environments & Custom Kernels with UV

🚨 `uv` = ultrafast Python package manager (Rust-based), replacement for
pip/pip-tools.

-   Manages **virtual environments** and dependencies.\
-   Works with **PyPI + AWS CodeArtifact**.\
-   Uses **`pyproject.toml`** + **`uv.lock`**.

------------------------------------------------------------------------

### 1. Setting Up New Venv with UV

``` bash
cd /home/sagemaker-user/user-default-efs/project-X

# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Create venv
uv init
uv venv
```

------------------------------------------------------------------------

### 2. Installing Dependencies
Append this to end of the pyproject.toml to get the authentication script working 
``` bash
[tool.uv.sources]
imu-snowflake-main = { index = "aws" }

[[tool.uv.index]]
url = "https://imu-257033705919.d.codeartifact.eu-west-1.amazonaws.com/pypi/imu/simple/"
name = "aws"
```

``` bash
# Authenticate with AWS CodeArtifact
source codeartifact-auth.sh

# Add packages
uv add ipykernel
uv add pandas
uv add imu-snowflake-main

# Install from pyproject.toml
uv sync
```
------------------------------------------------------------------------

### 3. Register Venv as Jupyter Kernel

``` bash
uv run python -m ipykernel install --user   --name project_x   --display-name "Project X"
```

------------------------------------------------------------------------

### 4. Running Scripts

``` bash
# Good (uses correct env)
uv run python script.py

# Bad (may use global env)
python script.py
```

------------------------------------------------------------------------

## 📘 Setting Up Projects in Amazon EFS

🚨 `user-default-efs` = shared, persistent filesystem across spaces.

**Recommended project structure:**

``` bash
/user/default/efs/
├── project-A/
│   ├── .venv/
│   ├── pyproject.toml
│   └── main.py
├── project-B/
│   ├── .venv/
│   ├── pyproject.toml
│   ├── notebook.ipynb
│   └── main.py
```

-   Each project has its **own venv**.\
-   `uv run` picks nearest `.venv` + `pyproject.toml`.

------------------------------------------------------------------------

## 📙 Reading MFI Data from Snowflake

### 1. IMU-Specific Tooling

-   Private package: `imu_snowflake_main`.\
-   Auth: `codeartifact_auth.sh`.\
-   Config: `snowflake.conf` in `/home/sagemaker-user/.local`.\
-   Queries run via **Ibis** → SQL-like queries in Python.

------------------------------------------------------------------------

### 2. Example Query

``` python
from imu.snowflake import SnowflakeClient

client = SnowflakeClient()  # if config in ‘/home/sagemaker-user/.local’


samples_table = client.table('V_FCS_TRACKED_SAMPLES', database='IMU.IMU')
samples_table  # inspect schema

bham_samples = samples_table.filter(samples_table['Project'] == 'BHAM')

bham_parquets = bham_samples.select(
    bham_samples['Sample ID'],
    bham_samples['OUTPUT_RAW_PARQUET_KEY'], 
    bham_samples['OUTPUT_XFORM_PARQUET_KEY'],
    bham_samples['Panel']
).execute()
```

------------------------------------------------------------------------

### 3. Parquet File Access (S3)

``` python
import pyarrow.parquet as pq

pq.ParquetDataset(s3_uri, columns=['CD4', 'CD56']).read().to_pandas()
```

------------------------------------------------------------------------

## 📗 Reading Membership Data from Snowflake

### 1. Membership Structures

-   **Natural Manual** → manual analysis\
-   **Natural Predicted** → ML analysis\
-   **Synthetic Manual** → synthetic manual\
-   **Synthetic Predicted** → synthetic ML

------------------------------------------------------------------------

### 2. Example Snowflake Queries

``` python
client = SnowflakeClient()

natural_manual = client.table('V_WSP_MEMBERSHIPS_LATEST', database='IMU.IMU_DAGSTER_LINEAGE')
natural_predicted = client.table('V_PREDICTED_MEMBERSHIPS_LATEST', database='IMU.IMU_DAGSTER_LINEAGE')
synthetic_manual = client.table('V_WSP_SYNTHETIC_MEMBERSHIPS_MANUAL_LATEST', database='IMU.IMU_DAGSTER_LINEAGE')
synthetic_predicted = client.table('V_WSP_SYNTHETIC_MEMBERSHIPS_PREDICTED_LATEST', database='IMU.IMU_DAGSTER_LINEAGE')
```

------------------------------------------------------------------------

### 3. Joining with Sample Sheet

``` python
samples_table = client.table('V_FCS_TRACKED_SAMPLES', database='IMU.IMU')

natural_manual_metadata = samples_table.join(
    natural_manual,
    (literal('s3://') + samples_table['S3 file path']) == natural_manual['FCS_S3_URI']
)
```

------------------------------------------------------------------------

### 4. Reading files from S3
**CVs**

``` bash
source codeartifact-auth.sh
uv add fsspec s3fs
```
**parquet**
``` python
import pandas as pd
import pyarrow.parquet as pq
import fsspec

events_path = natural_manual_metadata.loc[0, "EVENTS_S3_URI"]

with fsspec.open(events_path, "rb") as f:
    parquet_file = pq.ParquetFile(f)
    all_columns = parquet_file.schema.names

root_columns = [c for c in all_columns if "root" in c.lower()]

df = pd.read_parquet(events_path, columns=root_columns)
```

------------------------------------------------------------------------

## 📕 Reading GDrive Backups from S3

``` bash
aws s3 ls s3://imu-gdrive-backup/1/

# Copy into space
aws s3 cp s3://imu-gdrive-backup/1/myfile /home/sagemaker-user/
```
