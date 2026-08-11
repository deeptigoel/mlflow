| Tag                   | Example value                           | Set on              | Purpose                                 |
| --------------------- | --------------------------------------- | ------------------- | --------------------------------------- |
| `run_type`            | `inference` / `evaluation`              | Run                 | Identifies execution type               |
| `run_scope`           | `summarization_pipeline` / `scoring`    | Run                 | Identifies pipeline scope               |
| `capability`          | `abstractive summarization`             | Run                 | ML capability                           |
| `product`             | `ABC`                                   | Run                 | Product being processed                 |
| `dataset_version`     | `ABC_Respiratory_2025-01-01_2025-12-31` | Run                 | Input dataset/version identifier        |
| `git_sha`             | `<commit_sha>`                          | Run                 | Code version/reproducibility            |
| `parent_run_id`       | `<inference_run_id>`                    | Model version       | Links registered model to inference run |
| `validation_status`   | `passed` / `failed`                     | Model version       | Model validation result                 |
| `registration_status` | `registered` / `skipped`                | Run                 | Registration outcome                    |
| `model_name`          | `model_a` / `model_b`                   | Run                 | Models produced by inference            |
| `number_of_model`     | `2`                                     | Run                 | Number of generated models              |
| `evaluation_run_id`   | `<evaluation_run_id>`                   | Run + Model version | Links evaluation to model               |
| `promotion_role`      | `champion` / `challenger`               | Model version       | Promotion assignment                    |
| `composite_score`     | `0.87`                                  | Model version       | Final promotion score                   |
| `champion_model`      | `model_a`                               | Evaluation run      | Selected champion                       |
| `challenger_model`    | `model_b`                               | Evaluation run      | Selected challenger                     |
| `promotion_status`    | `completed` / `failed`                  | Evaluation run      | Promotion outcome                       |
| `evaluation_status`   | `complete` / `failed`                   | Evaluation run      | Evaluation outcome                      |
| `pipeline_status`     | `completed` / `failed`                  | Run                 | Overall pipeline status                 |
| `failure_reason`      | `No model passed validation`            | Run                 | Failure explanation                     |



| Tag                          | Example value                               | Purpose                    |
| ---------------------------- | ------------------------------------------- | -------------------------- |
| `source_type`                | `delta` / `csv` / `excel`                   | Identifies source type     |
| `source_delta_table_name`    | `silver_summarization`                      | Source Delta table         |
| `source_delta_table_version` | `125`                                       | Exact Delta version        |
| `source_table_catalog`       | `my_catalog`                                | UC catalog                 |
| `source_table_schema`        | `my_schema`                                 | UC/workspace schema        |
| `source_table_full_name`     | `my_catalog.my_schema.silver_summarization` | Fully qualified source     |
| `source_path`                | `input.xlsx`                                | File-based source location |


| Tag                 | Example value                |
| ------------------- | ---------------------------- |
| `model_card_status` | `generated`                  |
| `model_card_path`   | `model_card/model_card.json` |


