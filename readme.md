| Provenance section | Field                  | Source / Function                                                                                 |
| ------------------ | ---------------------- | ------------------------------------------------------------------------------------------------- |
| **model**          | `name`                 | MLflow Model Registry → `client.get_model_version()`                                              |
|                    | `version`              | MLflow Model Registry → `client.get_model_version()`                                              |
|                    | `product`              | MLflow run tag `product` → `set_run_tags(product=PRODUCT_NAME)`                                   |
|                    | `model_type`           | Pipeline/model configuration; **not currently logged explicitly**                                 |
| **dataset**        | `version`              | MLflow run tag `dataset_version`                                                                  |
|                    | `source`               | MLflow run tag `source_path` via `set_source_lineage_tags()`                                      |
|                    | `source_type`          | MLflow run tag `source_file_type` via `set_source_lineage_tags()`                                 |
| **runs**           | `parent_run_id`        | `mlflow.active_run().info.run_id`                                                                 |
|                    | `evaluation_run_id`    | Evaluation `child_run_id` → `update_run_tags(evaluation_run_id=...)`                              |
| **evaluation**     | `ground_truth.source`  | `GROUNDTRUTH_DIR` → `log_artifacts()`                                                             |
|                    | `ground_truth.version` | **Not currently logged/defined in the shared pipeline**                                           |
|                    | `metrics`              | `evaluate_product()` → `add_quality_labels()` → `calculate_composite_score()` → `log_metrics()`   |
|                    | `quality_labels`       | `add_quality_labels()` → `log_eval_quality_tags()`                                                |
|                    | `composite_score`      | `calculate_composite_score()` → model-version tag `composite_score`                               |
| **promotion**      | `role`                 | `prepare_promotion_metadata()` → `promotion_info["aliases"]` → model-version tag `promotion_role` |
|                    | `champion_model`       | `prepare_promotion_metadata()` → `promotion_info["champion"]` → `update_run_tags()`               |
|                    | `challenger_model`     | `prepare_promotion_metadata()` → `promotion_info["challenger"]` → `update_run_tags()`             |
|                    | `champion_score`       | `prepare_promotion_metadata()` → `promotion_info["champion_score"]`                               |
|                    | `challenger_score`     | `prepare_promotion_metadata()` → `promotion_info["challenger_score"]`                             |
|                    | `validation_status`    | `validate_model()` → `update_run_tags(validation_status=...)`                                     |
| **model_card**     | `path`                 | `build_model_card()` → `save_model_card()` → `log_model_card()`                                   |
| **lineage**        | `platform`             | Databricks/MLflow configuration                                                                   |
|                    | `catalog`              | Unity Catalog metadata, if applicable                                                             |
|                    | `schema`               | Unity Catalog metadata, if applicable                                                             |
|                    | `upstream/downstream`  | Unity Catalog lineage, if available                                                               |




prepare_promotion_metadata() determines the champion from the ranked evaluation dataframe; then update_run_tags(champion_model=...) logs that dictionary to the evaluation run.

So for your provenance mapping:

JSON field	Source	Function
promotion.role	promotion_info["aliases"]	prepare_promotion_metadata()
promotion.champion_model	promotion_info["champion"]	prepare_promotion_metadata() → update_run_tags()
promotion.composite_score	promotion_info["champion_score"] / model-version composite_score	calculate_composite_score() → prepare_promotion_metadata()

And specifically, the champion dictionary itself comes from _select_champion() inside prepare_promotion_metadata().
