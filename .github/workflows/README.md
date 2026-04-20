# Junco Legacy Hotfix CI/CD Pipeline

This directory contains the GitHub Actions workflows used to validate hotfixes for legacy versions of the `junco` package. 

Because legacy environments are locked and cannot use modern CRAN packages without breaking, this pipeline uses a "Time Machine" approach combined with dynamic memory injection (`hotpatchR`) to safely test fixes exactly as they will behave in production.

## Architecture Overview

Unlike standard R package CI pipelines, this workflow performs a **dual-checkout**:
1. It pulls your current hotfix script (`hotfix_repo`).
2. It pulls the pristine, historical source code of the legacy package (`junco_source`).

It then installs the legacy package, dynamically injects your hotfix into its locked memory, and runs the legacy test suite.

## Key Mechanisms (Do Not Alter Without Review)

### 1. The CRAN Time Machine (RSPM)
**Variable:** `RSPM: "https://packagemanager.posit.co/cran/2024-04-20"`
Legacy packages rely on legacy dependencies. If we build `junco v0.1.3` using today's CRAN, tests will fail due to deprecated upstream functions. We lock the Posit Public Package Manager (RSPM) to the exact date the legacy container was finalized (e.g., `2024-04-20`). This guarantees zero dependency drift.

### 2. Specific R Version Binding
**Variable:** `matrix.config.r: '4.5.2'`
This pipeline is hardcoded to run on R `4.5.2` to exactly mirror the `2025q4_r450_1_0_0` production container. Do not upgrade this to `release` or `latest` unless the target container's R version is also being upgraded.

### 3. Memory Injection (`hotpatchR`)
This pipeline uses `hotpatchR` to rewrite the package namespace in memory. This provides a 1:1 simulation of how the hotfix script will execute in the end-user's R session.

---

## How to Create a Pipeline for a New Legacy Version

When you need to support a new legacy version (e.g., `v0.1.4`), duplicate the existing YAML file and update the following **four targets**:

### Step 1: Update Workflow Name and Triggers
Rename the file (e.g., `hotfix-v0.1.4.yml`) and update the header name so it is easily identifiable in the GitHub Actions UI.
```yaml
name: Hotfix Tests v0.1.4 🔥
```

### Step 2: Update the Legacy Branch/Tag Reference
Tell the pipeline which historical version of `junco` to check out as the baseline.
```yaml
      - name: Checkout Junco v0.1.4
        uses: actions/checkout@v4
        with:
          ref: refs/heads/v0.1.4  # <-- Update this to the target branch or tag
          path: junco_source
```

### Step 3: Update the Hotfix Target File
Update the R script injection path so it looks for the correct file in your `feature/hotfix-*` branch.
```yaml
      - name: Execute tests via hotpatchR
        run: |
          # ...
          apply_hotfix_file("hotfix_repo/dev/junco_hotfix_v0-1-4.R") # <-- Update filename
```

### Step 4: Align the Time Machine Date
Determine when the new legacy version was deployed, and update the RSPM snapshot date to match that era.
```yaml
    env:
      # e.g., Set to the month v0.1.4 was released
      RSPM: "[https://packagemanager.posit.co/cran/2024-10-15](https://packagemanager.posit.co/cran/2024-10-15)" 
```

Once these four variables are updated, the pipeline will automatically trigger whenever a developer pushes a branch starting with `feature/hotfix-*` and will test their script against the newly defined legacy environment.
