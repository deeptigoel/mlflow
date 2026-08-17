1. End-to-end testing

In your notebook, you can have a subsection:

End-to-End Testing

Execute the complete pipeline once.
Verify successful flow: evaluation → composite scoring → promotion → model registration → model-card generation.
Verify traceability in the MLflow UI.
2. Registered model & traceability verification

Then add a separate cell to inspect the existing registered model:

import mlflow
from mlflow import MlflowClient


client = MlflowClient()


registered_name = "YOUR_REGISTERED_MODEL_NAME"
version = "YOUR_MODEL_VERSION"


# Retrieve model version
model_version = client.get_model_version(
    name=registered_name,
    version=version,
)


print("Model:", registered_name)
print("Version:", version)
print("Source:", model_version.source)
print("Run ID:", model_version.run_id)


# Retrieve model-version tags
print("\nModel Version Tags:")
for k, v in model_version.tags.items():
    print(f"{k}: {v}")
3. Retrieve the evaluation run
evaluation_run_id = model_version.tags.get("evaluation_run_id")


evaluation_run = client.get_run(evaluation_run_id)


print("Evaluation Run ID:", evaluation_run.info.run_id)


print("\nMetrics:")
for k, v in evaluation_run.data.metrics.items():
    print(f"{k}: {v}")


print("\nParameters:")
for k, v in evaluation_run.data.params.items():
    print(f"{k}: {v}")


print("\nTags:")
for k, v in evaluation_run.data.tags.items():
    print(f"{k}: {v}")
4. Check logged artifacts
artifacts = client.list_artifacts(evaluation_run_id)


print("Evaluation artifacts:")
for artifact in artifacts:
    print(
        artifact.path,
        artifact.file_size,
    )

For nested artifact folders:

eval_artifacts = client.list_artifacts(
    evaluation_run_id,
    path="eval_output",
)


for artifact in eval_artifacts:
    print(artifact.path, artifact.file_size)
5. Verify the model can actually be loaded

This is particularly useful given the previous deserialization issue:

model_uri = f"models:/{registered_name}/{version}"


loaded_model = mlflow.transformers.load_model(
    model_uri
)


print("Model loaded successfully:", type(loaded_model))

If you're using UC aliases:

model_uri = f"models:/{registered_name}@Champion"
6. Verify the model card

If the model card is logged under the evaluation run:

card_artifacts = client.list_artifacts(
    evaluation_run_id,
    path="model_card",
)


for artifact in card_artifacts:
    print(artifact.path, artifact.file_size)
Suggested PR testing section

I would structure it as:

Testing Performed

1. End-to-End Testing

Performed end-to-end testing of the pipeline from evaluation through model registration, promotion, and model-card generation.
Verified end-to-end traceability across evaluation metrics, composite scores, promotion roles, registered model versions, and model cards.

2. MLflow Verification

Verified experiment, parent/nested runs, tags, parameters, metrics, and artifacts in the MLflow UI.
Verified registered model/version metadata and promotion aliases.
Verified the registered model can be loaded successfully without rerunning evaluation.
Verified evaluation outputs, model artifacts, and model-card artifacts are present and non-empty.

This is stronger than simply saying “verified in UI” because you're programmatically validating the persisted MLflow state as well as visually checking it.
