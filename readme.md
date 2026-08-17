PR Title

Enhance MLflow Lineage, Evaluation Logging and Model Card Automation

Summary

This PR enhances the summarization pipeline with centralized MLflow configuration, source/data lineage tracking, evaluation metadata logging, model-version tagging, and automated model-card generation.

The changes are intended to improve traceability, reproducibility, model governance, and reviewability across inference, evaluation, and model-promotion workflows.

Changes Included
1. Centralized MLflow Configuration

Introduced configure_mlflow() in mlflow_utils.py to centralize MLflow setup.

Function:

configure_mlflow()

The function is responsible for configuring:

MLFLOW_TRACKING_URI
MLFLOW_REGISTRY_URI
MLflow experiment
Unity Catalog/non-Unity Catalog configuration

These values are now intended to come from config.py rather than being hard-coded in the pipeline.

Reviewer note:
Please verify that environment-specific tracking/registry URIs and experiment names are correctly maintained in configuration rather than pipeline code.

2. Source Lineage Tagging

Added centralized source lineage handling through:

set_source_lineage_tags()

The function captures source-specific metadata such as:

Source file type
Source path
Delta table/version information where applicable
Catalog/schema/table information for UC-based sources

This provides a consistent mechanism for tracking where inference data originated.

Reviewer note:
Please verify that the lineage tags are sufficient for both current file-based ingestion and future Delta/Unity Catalog sources.

3. Input Dataset Logging

Added MLflow dataset tracking using:

mlflow.data.from_pandas()
mlflow.log_input()

The inference dataset is now associated with the MLflow run using an explicit context:

context="inference"

The original input file is also logged as an artifact under:

input/

This allows reviewers to distinguish between the MLflow dataset reference and the actual input artifact.

4. Ground Truth / Evaluation Data Logging

Evaluation-related data is logged separately under:

groundtruth/

The evaluation run is created as a nested MLflow run:

mlflow.start_run(
    run_name="Evaluation",
    nested=True
)

The evaluation run captures:

Evaluation run ID
Evaluation outputs
Evaluation metrics
Evaluation parameters
Promotion metadata

Reviewer note:
Human/reference summaries are evaluation inputs and should not automatically become model-card metrics. They are excluded from model-card metadata where appropriate.

5. Evaluation Metrics and Parameters

Evaluation results are now converted into model-specific metrics and parameters.

The important change is that metrics/parameters are extracted inside the registered-model loop, after filtering:

model_rows = eval_df_composite[
    eval_df_composite["model_name"] == model_name
]

This ensures that each model receives its own:

eval_metrics
eval_parameters

rather than accidentally reusing metrics from another model.

Excluded fields include:

EXCLUDE_FROM_MODEL_CARD = {
    "human_summary",
    "ground_truth_summary",
    "reference_summary",
}

Reviewer note:
Please specifically review this section because multiple models are evaluated in the same evaluation run. Model-specific filtering prevents cross-model metric contamination.

6. Model Logging with Error Handling

Transformer models are logged using:

log_transformer_model()

The function wraps:

mlflow.transformers.log_model()

in exception handling to capture model serialization/deserialization failures.

The pipeline checks the returned status:

model_logged = log_transformer_model(
    model_name,
    model_pipeline,
)

if not model_logged:
    update_run_tags(
        pipeline_status="failed",
    )
    return False

This prevents the pipeline from continuing to model registration when model logging has failed.

Reviewer note:
Please verify that model-registration logic never proceeds with an invalid/incompletely logged model artifact.

7. Model Validation and Registration

Each successfully logged model is validated before registration:

validate_model()

Models passing validation are registered using:

register_model()

The registered-model metadata includes:

Model name
Registered model name
Version
Dataset version
Registration status
8. Model-Version Evaluation and Promotion Tags

Each registered model version is linked back to the evaluation run using:

evaluation_run_id

Additional model-version tags include:

promotion_role
composite_score

For Unity Catalog, the appropriate promotion alias is also assigned.

This creates a traceability chain:

Dataset → Parent Run → Evaluation Run → Model Version → Promotion Role → Model Card

9. Automated Model Card Generation

Model cards are now generated programmatically using:

build_model_card()

The model card receives:

Model name
Registered model name
Model version
Parent run ID
Evaluation run ID
Validation status
Dataset version
Model-specific evaluation metrics
Evaluation parameters
Promotion role
Composite score

The generated card is then persisted using:

save_model_card()

and logged using:

log_model_card()

Finally, the model version is tagged with the model-card location using:

tag_model_version_with_card()
Overall Traceability

The intended flow after these changes is:

Input Dataset
     ↓
Source Lineage
     ↓
Parent MLflow Run
     ↓
Inference
     ↓
Logged Transformer Models
     ↓
Validation
     ↓
Registered Model Version
     ↓
Evaluation Run
     ↓
Model-specific Metrics/Parameters
     ↓
Champion / Challenger
     ↓
Model Version Tags / UC Alias
     ↓
Automated Model Card
Reviewer Focus Areas

I would specifically ask reviewers to look at these areas:

configure_mlflow()
Confirm tracking URI, registry URI and experiment configuration are fully centralized in config.py.
set_source_lineage_tags()
Confirm the lineage tags correctly support both file-based and future Delta/UC sources.
Model-specific evaluation extraction
Verify metrics and parameters are refreshed for every model and are not accidentally reused between models.
log_transformer_model() error handling
Verify serialization/deserialization failures correctly stop downstream registration.
Model-version tags and promotion aliases
Confirm champion/challenger roles and composite scores correspond to the correct model version.
Model-card generation
Verify that the model card contains only the relevant model's evaluation metrics/parameters and does not include raw human/reference summaries unnecessarily.
MLflow artifact organization
Confirm input, ground truth, evaluation output, model artifacts, and model cards have clear and non-overlapping artifact paths.
PR Note

Testing: Pipeline execution was validated across multiple models, with particular attention to model-specific evaluation metrics, MLflow lineage, model logging, registration, promotion metadata, and model-card generation.

Yes. Based on your current code, pipeline("summarization", model=hf_model_name) uses the Hugging Face Transformers loading mechanism, which normally downloads the model into the Hugging Face cache if it isn't already cached.

Your pipeline itself is not explicitly specifying a cache directory such as cache_dir=....

So effectively:

pipeline(
    "summarization",
    model=hf_model_name
)

→ checks Hugging Face cache
→ downloads missing model files/checkpoint shards
→ loads the model from the local cache.

Today 10:25 AM
I want to add one statement in not more than 20 to 30 words, you know, as discussed actually, XYZ, actually the manager name, she will be adding, you know, she has a solution to the failure which I was getting, which was regarding the run. So, which would get added. So I just want to mention that, though currently I have added those try except things, but yeah, I mean, that is something that needs to be updated. How to say this in not more than 30 words?

As discussed, XYZ will add the proposed solution for the run-related failure; the current try-except handling will be updated accordingly once incorporated.

Actually, PR accepted, other things have been updated, but she has, you know, a script that would, that handles actually because these were two different things. One was a load transformer model that was happening and other being the log thing, so which was mentioned. Yeah, so those things I want to mention in the PR.

You can add this PR note:

Run-related failure handling: The load_transformer_model and MLflow logging failures were separate issues. The current try-except handling is in place, while XYZ’s script-based solution for the run-related failure will be incorporated separately.
