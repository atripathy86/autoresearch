# CMF Instrumentation for Autoresearch

This branch adds minimal CMF (Common Metadata Framework) instrumentation to track autoresearch experiments. CMF: (https://github.com/HewlettPackard/cmf)

## What Changed

### 1. Dependencies
- Added `cmflib>=0.1.0` to `pyproject.toml`
- Run `uv sync` to install after checking out this branch

### 2. Prepare Script (`prepare.py`)
- Added `--with-cmf` flag to enable CMF tracking (optional, off by default)
- When enabled, tracks:
  - Data download configuration
  - Tokenizer training parameters
  - Output artifacts (data directory, tokenizer directory)
  - Creates `.cmf_enabled` marker file so train.py automatically uses CMF too
- Metadata stored in `mlmd` file

### 3. Train Script (`train.py`)
- CMF tracking is disabled by default
- Supports `--with-cmf` flag directly
- Automatically enables CMF tracking if any of:
  - `--with-cmf` flag is passed, OR
  - `WITH_CMF=1` environment variable is set, OR
  - `.cmf_enabled` marker file exists (created by `prepare.py --with-cmf`)
- Tracks when enabled:
  - Model architecture configuration (layers, heads, embedding dimensions)
  - Training hyperparameters (learning rates, batch size, etc.)
  - Training metrics (val BPB, training time, MFU, tokens, etc.)
  - Final model artifact saved as `model.pt`

## Usage

### Without CMF (default)
```bash
uv sync
uv run prepare.py
uv run train.py
```

### With CMF (options)

Initialize CMF: 
- Update git-remote and cmf-server URL's
```
cmf init local --path ~/cmf_client_data \
--git-remote-url https://github.com/user/experiment-repo.git \
--cmf-server-url http://cmf-server/api \
--neo4j-user neo4j --neo4j-password password \
--neo4j-uri neo4j://cmf-server:7687 
```

OR if using OSDF Remote: 
- Populate ~/.fdp/osdf_token 
- Update git-remote and cmf-server URL's
```
cmf init osdfremote --path https://fdp-origin.labs.hpe.com:8443/fdp-hpe/autoresearch \
--git-remote-url https://github.com/user/experiment-repo.git \
--cmf-server-url http://cmf-server/api \
--neo4j-user neo4j --neo4j-password password \
--neo4j-uri neo4j://cmf-server:7687 
```

**Option 1: Consistent automatic tracking (recommended)**
```bash
# This enables CMF for both prepare and train automatically via marker file
uv run prepare.py --with-cmf
uv run train.py  # automatically uses CMF because .cmf_enabled exists
```

**Option 2: Explicit per-script control**
```bash
# Enable CMF for both scripts explicitly
uv run prepare.py --with-cmf
uv run train.py --with-cmf

# Enable CMF only for prepare
uv run prepare.py --with-cmf
uv run train.py  # no CMF (unless .cmf_enabled exists)

# Enable CMF only for train (prepare must have run)
uv run prepare.py
uv run train.py --with-cmf
```

**Option 3: Environment variable override**
```bash
# Enable CMF via env var
WITH_CMF=1 uv run train.py

# Explicitly disable train CMF even if .cmf_enabled exists
WITH_CMF=0 uv run train.py  # (empty or 0 will disable)
```

### Push Artifacts and Metadata
```bash
cmf artifact push
cmf metadata push
```

## Tracking Details

CMF tracks:
- **Pipeline Stage**: Prepare (data prep), Train (training loop)
- **Execution Type**: Prepare, Train-execution
- **Artifacts**:
  - Data directory (`~/.cache/autoresearch/data`)
  - Tokenizer directory (`~/.cache/autoresearch/tokenizer`)
  - Trained model (`model.pt`)
- **Metrics**: val_bpb, training time, peak VRAM, MFU, etc.
- **Custom Properties**: Model config, hyperparameters

## Minimal Changes

Following the example from `cmf/examples/example-get-started`, the instrumentation adds:
1. Pipeline context creation
2. Execution logging with custom properties
3. Dataset logging for inputs/outputs
4. Metric collection and committing
5. Model artifact logging
6. Finalization

All changes are non-invasive and the default behavior (no CMF) remains unchanged.
