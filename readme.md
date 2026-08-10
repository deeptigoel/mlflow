if model_exists(registered_name, client):

    logger.info(
        "Model '%s' already exists in registry. "
        "Skipping model registration.",
        registered_name,
    )

    # Get existing registered versions
    versions = client.search_model_versions(
        f"name = '{registered_name}'"
    )

    if not versions:
        raise RuntimeError(
            f"Model '{registered_name}' exists, "
            "but no model versions were found."
        )

    # Return the latest existing version
    latest_version = max(
        versions,
        key=lambda v: int(v.version),
    )

    return {
        "registered_name": registered_name,
        "version": latest_version.version,
        "already_registered": True,
    }

####


logger.info(
    "Registering model '%s'.",
    registered_name,
)

registered_model = mlflow.register_model(
    model_uri=model_uri,
    name=registered_name,
)

version = registered_model.version


######


```text
Load data
   ↓
Preprocess
   ↓
Parent MLflow Run
   ├── Initial run tags
   ├── Environment
   ├── Parameters
   ├── Input dataset lineage
   ├── Input artifact
   │
   ├── Inference
   │
   ├── Log output artifact
   ├── Log model
   ├── Validate
   │
   └── Register validated models
          ├── parent_run_id
          ├── dataset_version
          └── validation_status
                    ↓
             Evaluation Run
                    ├── evaluation_run_id
                    ├── ground-truth artifacts
                    ├── evaluate_product
                    ├── quality labels
                    ├── composite score
                    ├── Champion
                    └── Challenger
                              ↓
                 Update registered versions
                    ├── evaluation_run_id
                    ├── promotion_role
                    ├── composite_score
                    └── UC alias
```


