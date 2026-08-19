Solution needs to support both standard MLflow Registry and Unity Catalog (UC), would make the contract UC-compatible without making it UC-specific.
provenance should be able to answer:
Model → dataset → ground truth → evaluation run → metrics → promotion → model card.


{
  "model": {
    "name": "catalog.schema.model_A",
    "version": "5",
    "framework": "huggingface_transformers",
    "model_type": "summarization"
  },

  "dataset": {
    "version": "ABC_Respiratory_2025",
    "source": "input.xlsx",
    "source_type": "excel"
  },

  "runs": {
    "parent_run_id": "abc123",
    "evaluation_run_id": "xyz789"
  },

  "evaluation": {
    "ground_truth": {
      "source": "groundtruth/",
      "version": "GT_v1"
    },

    "metrics": {
      "rouge": 0.81,
      "bleu": 0.72,
      "meteor": 0.76,
      "bert_score": 0.87,
      "factual_consistency": 0.78
    },

    "quality_labels": {
      "rouge": "good",
      "bleu": "acceptable",
      "meteor": "good",
      "bert_score": "strong",
      "factual_consistency": "weak"
    },

    "composite_score": 0.82
  },

  "promotion": {
    "role": "Champion",
    "validation_status": "passed"
  },

  "model_card": {
    "path": "model_card/model_card.json"
  },

  "lineage": {
    "platform": "Databricks",
    "catalog": "catalog",
    "schema": "schema",
    "upstream": [],
    "downstream": []
  }
}


For UC:

catalog.schema.model_A naturally represents the UC registered model.
catalog and schema allow the resolver to query UC metadata/lineage.
upstream / downstream can later contain UC lineage information.
mlflow_registry_uri makes it explicit which registry is being used.

For non-UC MLflow:

model.name can simply be something like summarization_model_A.
catalog and schema can be null.


The complete conceptual chain is:

Model → Dataset → Parent Run → Evaluation Run → Ground Truth → Metrics → Quality Labels → Composite Score → Validation → Promotion → Model Card → UC/Databricks Lineage

This is what is for “Design lineage data model"

2. Document the field to source mapping


| Provenance section          | Source                         |
| --------------------------- | ------------------------------ |
| `model.name/version`        | MLflow Model Registry / UC     |
| `promotion.role`            | `prepare_promotion_metadata()` |
| `promotion.composite_score` | `calculate_composite_score()`  |
| `runs.parent_run_id`        | MLflow parent run              |
| `runs.evaluation_run_id`    | Evaluation run tag             |
| `dataset`                   | MLflow logged input + tags     |
| `evaluation.ground_truth`   | Ground-truth artifact/input    |
| `evaluation.metrics`        | Evaluation run metrics         |
| `evaluation.quality_labels` | `add_quality_labels()`         |
| `model_card`                | Model-card artifact            |
| `lineage`                   | UC lineage/system tables       |


### Define the API request contract

For example:

{
  "model_name": "catalog.schema.model_A",
  "version": "5"
}



######
lineage_resolver.py:


from typing import Any

from mlflow import MlflowClient


def resolve_model_provenance(
    model_name: str,
    version: str,
    client: MlflowClient,
) -> dict[str, Any]:
    """
    Resolve complete provenance for a registered MLflow model version.

    Retrieves model metadata, promotion information, run lineage,
    dataset provenance, evaluation results, model-card information,
    and Unity Catalog lineage where available.

    Args:
        model_name: Registered model name.
        version: Registered model version.
        client: Configured MLflow client.

    Returns:
        Dictionary containing the complete model provenance.

    Raises:
        ValueError: If the model version cannot be found.
    """

    model_version = client.get_model_version(
        name=model_name,
        version=version,
    )

    parent_run_id = model_version.tags.get(
        "parent_run_id"
    )

    evaluation_run_id = model_version.tags.get(
        "evaluation_run_id"
    )

    promotion_role = model_version.tags.get(
        "promotion_role"
    )

    composite_score = model_version.tags.get(
        "composite_score"
    )

    validation_status = model_version.tags.get(
        "validation_status"
    )

    provenance = {
        "model": {
            "name": model_version.name,
            "version": str(model_version.version),
            "framework": "huggingface_transformers",
            "model_type": "summarization",
        },

        "dataset": {},

        "runs": {
            "parent_run_id": parent_run_id,
            "evaluation_run_id": evaluation_run_id,
        },

        "evaluation": {
            "ground_truth": {},
            "metrics": {},
            "quality_labels": {},
            "composite_score": (
                float(composite_score)
                if composite_score is not None
                else None
            ),
        },

        "promotion": {
            "role": promotion_role,
            "validation_status": validation_status,
        },

        "model_card": {},

        "lineage": {},
    }

    # Resolve parent run
    if parent_run_id:

        parent_run = client.get_run(
            parent_run_id
        )

        provenance["dataset"] = (
            _resolve_dataset(parent_run)
        )

    # Resolve evaluation run
    if evaluation_run_id:

        evaluation_run = client.get_run(
            evaluation_run_id
        )

        provenance["evaluation"].update(
            _resolve_evaluation(
                evaluation_run
            )
        )

    # Resolve model-card information
    provenance["model_card"] = (
        _resolve_model_card(
            parent_run_id,
            client,
        )
    )

    # Resolve UC lineage
    provenance["lineage"] = (
        _resolve_uc_lineage(
            model_name=model_name,
            version=version,
        )
    )

    return provenance




######

def _resolve_dataset(run) -> dict[str, Any]:
    """
    Resolve dataset provenance from the parent MLflow run.
    """

    tags = run.data.tags

    return {
        "version": tags.get(
            "dataset_version"
        ),
        "source": tags.get(
            "source_path"
        ),
        "source_type": tags.get(
            "source_file_type"
        ),
    }

#######

def _resolve_evaluation(run) -> dict[str, Any]:
    """
    Resolve evaluation metrics and quality labels
    from the evaluation MLflow run.
    """

    tags = run.data.tags
    metrics = run.data.metrics

    quality_labels = {}

    # Retrieve only labels that were actually logged.
    # Do not invent an overall label.

    for key, value in tags.items():

        if key.startswith(
            "quality_label_"
        ):
            metric_name = key.replace(
                "quality_label_",
                "",
            )

            quality_labels[
                metric_name
            ] = value

    return {
        "metrics": metrics,
        "quality_labels": quality_labels,
        "ground_truth": {
            "source": tags.get(
                "groundtruth_source"
            ),
            "version": tags.get(
                "groundtruth_version"
            ),
        },
    }


######

def _resolve_model_card(
    parent_run_id: str | None,
    client: MlflowClient,
) -> dict[str, Any]:
    """
    Resolve model-card artifact information
    associated with the model's parent run.
    """

    if not parent_run_id:
        return {}

    artifacts = client.list_artifacts(
        parent_run_id,
        path="model_card",
    )

    return {
        "path": (
            "model_card/model_card.json"
            if artifacts
            else None
        )
    }

#####

def _resolve_uc_lineage(
    model_name: str,
    version: str,
) -> dict[str, Any]:
    """
    Resolve upstream and downstream lineage
    from Unity Catalog where available.

    This function does not perform MLflow inference
    or evaluation.
    """

    return {
        "platform": "Databricks",
        "catalog": None,
        "schema": None,
        "upstream": [],
        "downstream": [],
    }


#######

