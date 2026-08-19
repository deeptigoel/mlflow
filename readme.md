| Provenance section | Field                  | Source / Function                                                                              |
| ------------------ | ---------------------- | ---------------------------------------------------------------------------------------------- |
| **model**          | `name`                 | MLflow Model Registry / `client.get_model_version()`                                           |
|                    | `version`              | MLflow Model Registry / `client.get_model_version()`                                           |
|                    | `framework`            | Model metadata/configuration — **not currently logged as a tag**                               |
|                    | `model_type`           | Pipeline/model configuration — **not currently logged as a tag**                               |
| **dataset**        | `version`              | MLflow tag: `dataset_version`                                                                  |
|                    | `source`               | MLflow tag: `source_path`                                                                      |
|                    | `source_type`          | MLflow tag: `source_file_type`                                                                 |
| **runs**           | `parent_run_id`        | `mlflow.active_run().info.run_id`                                                              |
|                    | `evaluation_run_id`    | Evaluation run's `child_run_id`, stored through `update_run_tags()`                            |
| **evaluation**     | `ground_truth.source`  | Ground-truth artifact/input; corresponding tag needs to be logged                              |
|                    | `ground_truth.version` | Ground-truth metadata; corresponding tag needs to be logged                                    |
|                    | `metrics`              | `evaluate_product()` → `eval_results` → `calculate_composite_score()` / evaluation run metrics |
|                    | `quality_labels`       | `add_quality_labels()` → logged through evaluation metadata/tags                               |
|                    | `composite_score`      | `calculate_composite_score()` → `promotion_info` / MLflow metric or model-version tag          |
| **promotion**      | `role`                 | `prepare_promotion_metadata()` → `promotion_info["aliases"]` / `promotion_role`                |
|                    | `validation_status`    | `validate_model()` → `update_run_tags(validation_status=...)`                                  |
| **model_card**     | `path`                 | `build_model_card()` → `save_model_card()` → `log_model_card()`                                |
| **lineage**        | `platform`             | Databricks/MLflow configuration                                                                |
|                    | `catalog`              | Unity Catalog metadata, if applicable                                                          |
|                    | `schema`               | Unity Catalog metadata, if applicable                                                          |
|                    | `upstream/downstream`  | Unity Catalog lineage, if available                                                            |
