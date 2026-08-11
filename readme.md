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




from __future__ import annotations

import json
from datetime import datetime, timezone
from pathlib import Path
from typing import Any, Dict, Optional

import mlflow
from mlflow import MlflowClient


def build_model_card(
    *,
    model_name: str,
    registered_name: str,
    model_version: str,
    parent_run_id: str,
    evaluation_run_id: Optional[str],
    validation_status: str,
    dataset_version: str,
    evaluation_metrics: Dict[str, Any],
    promotion_role: Optional[str] = None,
    composite_score: Optional[float] = None,
) -> Dict[str, Any]:
    """
    Build a standardized model card from pipeline metadata.

    No model-card information is hard-coded; values come from
    registration, validation, evaluation and promotion results.
    """

    return {
        "model": {
            "name": model_name,
            "registered_name": registered_name,
            "version": str(model_version),
        },
        "lineage": {
            "parent_run_id": parent_run_id,
            "evaluation_run_id": evaluation_run_id,
            "dataset_version": dataset_version,
        },
        "validation": {
            "status": validation_status,
        },
        "evaluation": {
            "metrics": evaluation_metrics,
        },
        "promotion": {
            "role": promotion_role,
            "composite_score": composite_score,
        },
        "generated_at": datetime.now(
            timezone.utc
        ).isoformat(),
    }


def save_model_card(
    model_card: Dict[str, Any],
    output_dir: str,
) -> Dict[str, str]:
    """
    Save model card in JSON and Markdown formats.
    """

    output_path = Path(output_dir)
    output_path.mkdir(parents=True, exist_ok=True)

    json_path = output_path / "model_card.json"
    markdown_path = output_path / "model_card.md"

    with json_path.open("w", encoding="utf-8") as file:
        json.dump(
            model_card,
            file,
            indent=2,
            ensure_ascii=False,
        )

    markdown_path.write_text(
        render_model_card_markdown(model_card),
        encoding="utf-8",
    )

    return {
        "json": str(json_path),
        "markdown": str(markdown_path),
    }


def render_model_card_markdown(
    model_card: Dict[str, Any],
) -> str:
    """
    Convert the structured model-card dictionary into Markdown.
    """

    model = model_card["model"]
    lineage = model_card["lineage"]
    validation = model_card["validation"]
    evaluation = model_card["evaluation"]
    promotion = model_card["promotion"]

    metrics = evaluation.get("metrics", {})

    metric_lines = "\n".join(
        f"- **{key}:** {value}"
        for key, value in metrics.items()
    )

    return f"""# Model Card

## Model

- **Model name:** {model["name"]}
- **Registered model:** {model["registered_name"]}
- **Version:** {model["version"]}

## Lineage

- **Parent run ID:** {lineage["parent_run_id"]}
- **Evaluation run ID:** {lineage["evaluation_run_id"]}
- **Dataset version:** {lineage["dataset_version"]}

## Validation

- **Status:** {validation["status"]}

## Evaluation Metrics

{metric_lines}

## Promotion

- **Role:** {promotion["role"]}
- **Composite score:** {promotion["composite_score"]}

## Generated

{model_card["generated_at"]}
"""


def log_model_card(
    model_card_paths: Dict[str, str],
    artifact_path: str = "model_card",
) -> None:
    """
    Log generated model-card files to the active MLflow run.
    """

    for file_path in model_card_paths.values():
        mlflow.log_artifact(
            file_path,
            artifact_path=artifact_path,
        )


def tag_model_version_with_card(
    *,
    client: MlflowClient,
    registered_name: str,
    version: str,
    model_card_path: str,
) -> None:
    """
    Add model-card metadata to the registered model version.
    """

    client.set_model_version_tag(
        name=registered_name,
        version=version,
        key="model_card_path",
        value=model_card_path,
    )

    client.set_model_version_tag(
        name=registered_name,
        version=version,
        key="model_card_status",
        value="generated",
    )


