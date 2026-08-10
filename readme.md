```python
from datetime import datetime
import time

import mlflow
from mlflow import MlflowClient

from config import (
    MLFLOW_ENABLED,
    MAX_TOKENS_PER_CHUNK,
    BATCH_SIZE,
    MAX_LENGTH,
    MIN_TOKENS,
    PRODUCT_NAME,
    THERAPEUTIC_AREA,
    START_DATE,
    END_DATE,
    DATASET_VERSION,
    USE_UNITY_CATALOG,
    UC_CATALOG,
    UC_SCHEMA,
)

from mlflow_utils import (
    setup_experiment,
    set_run_tags,
    update_run_tags,
    log_environment,
    log_parameters,
    log_artifact,
    log_metrics,
    log_eval_quality_tags,
)

from data_loader import load_data, filter_data
from preprocess import preprocess_df, combine_documents
from summarizer import run_pipeline
from io_utils import save_summary, get_output_path

from validation import validate_model
from register_model import register_model
from evaluation import evaluate_product, add_quality_labels
from promotion_utils import (
    calculate_composite_score,
    prepare_promotion_metadata,
)

from model_card import log_model_card
from model_utils import log_transformer_model
from evaluation_utils import save_eval_metric


def main():

    start_time = time.time()

    # ============================================================
    # 1. LOAD DATA
    # ============================================================

    df = load_data(
        file_path="input.xlsx",
        sheet_name="Sheet1",
        converters={
            "PRODUCT": str,
            "THERAPEUTIC_AREA": str,
        },
    )

    df = filter_data(
        df=df,
        product_name=PRODUCT_NAME,
        therapeutic_area=THERAPEUTIC_AREA,
        start_date=START_DATE,
        end_date=END_DATE,
    )

    df = preprocess_df(
        df=df,
        text_column="TEXT",
    )

    combined_text = combine_documents(df)

    # ============================================================
    # 2. DATASET VERSION
    # ============================================================

    dataset_version = (
        f"{PRODUCT_NAME}_"
        f"{THERAPEUTIC_AREA}_"
        f"{START_DATE}_"
        f"{END_DATE}"
    )

    # ============================================================
    # 3. RUN WITHOUT MLFLOW
    # ============================================================

    if not MLFLOW_ENABLED:

        summary_results, models = run_pipeline(
            combined_text
        )

        output_file = get_output_path(
            PRODUCT_NAME,
            START_DATE,
            END_DATE,
        )

        save_summary(
            df,
            summary_results,
            output_file,
        )

        return summary_results

    # ============================================================
    # 4. MLFLOW SETUP
    # ============================================================

    mlflow.set_tracking_uri("Databricks")

    if USE_UNITY_CATALOG:

        experiment_name = (
            f"{UC_CATALOG}."
            f"{UC_SCHEMA}."
            f"Summarization"
        )

    else:

        experiment_name = (
            f"/Users/{user_name}/Summarization"
        )

    setup_experiment(experiment_name)

    run_name = (
        f"Summarization_"
        f"{datetime.now():%Y%m%d_%H%M%S}"
    )

    # ============================================================
    # 5. PARENT / INFERENCE RUN
    # ============================================================

    with mlflow.start_run(run_name=run_name):

        parent_run_id = (
            mlflow.active_run().info.run_id
        )

        client = MlflowClient()

        # --------------------------------------------------------
        # Initial run tags
        # --------------------------------------------------------

        set_run_tags(
            description="MLflow experiment",
            product=PRODUCT_NAME,
            run_type="inference",
            run_scope="summarization_pipeline",
            run_id=parent_run_id,
            capability="abstractive summarization",
            dataset_version=dataset_version,
            git_sha="testing_git",
        )

        # --------------------------------------------------------
        # Environment
        # --------------------------------------------------------

        log_environment()

        # --------------------------------------------------------
        # Parameters
        # --------------------------------------------------------

        log_parameters(
            {
                "chunk_size": MAX_TOKENS_PER_CHUNK,
                "batch_size": BATCH_SIZE,
                "max_length": MAX_LENGTH,
                "min_tokens": MIN_TOKENS,
                "num_documents": len(df),
            }
        )

        # ========================================================
        # 6. INFERENCE
        # ========================================================

        summary_results, models = run_pipeline(
            combined_text
        )

        # ========================================================
        # 7. SAVE SUMMARY
        # ========================================================

        output_file = get_output_path(
            PRODUCT_NAME,
            START_DATE,
            END_DATE,
        )

        save_summary(
            df,
            summary_results,
            output_file,
        )

        log_artifact(output_file)

        # ========================================================
        # 8. LOG MODEL ARTIFACTS
        # ========================================================

        registered_models = []

        for model_name, model_pipeline in models.items():

            # ----------------------------------------------------
            # Log model
            # ----------------------------------------------------

            log_transformer_model(
                model_name,
                model_pipeline,
            )

            model_uri = (
                f"runs:/{parent_run_id}/{model_name}"
            )

            # ----------------------------------------------------
            # Validation
            # ----------------------------------------------------

            validation_status = validate_model(
                model_pipeline,
            )

            update_run_tags(
                validation_status=(
                    "passed"
                    if validation_status
                    else "failed"
                )
            )

            # ----------------------------------------------------
            # Register only validated models
            # ----------------------------------------------------

            if validation_status:

                registered_info = register_model(
                    model_uri=model_uri,
                    run_id=parent_run_id,
                    model_name=model_name,
                    description=(
                        "Abstractive summarization model"
                    ),
                    dataset_version=dataset_version,
                    validation_status=validation_status,
                )

                registered_models.append(
                    {
                        "model_name": model_name,
                        "registered_name": (
                            registered_info[
                                "registered_name"
                            ]
                        ),
                        "version": (
                            registered_info[
                                "version"
                            ]
                        ),
                    }
                )

                update_run_tags(
                    registration_status="registered"
                )

            else:

                update_run_tags(
                    registration_status="skipped"
                )

        # --------------------------------------------------------
        # If no model passed validation, stop promotion/evaluation
        # --------------------------------------------------------

        if not registered_models:

            update_run_tags(
                pipeline_status="failed",
                failure_reason=(
                    "No model passed validation"
                ),
            )

            return False

        # ========================================================
        # 9. MODEL CARD
        # ========================================================

        log_model_card(
            MODEL_CARD_FILE
        )

        # ========================================================
        # 10. EVALUATION RUN
        # ========================================================

        with mlflow.start_run(
            run_name="Evaluation",
            nested=True,
        ):

            evaluation_run_id = (
                mlflow.active_run().info.run_id
            )

            # ----------------------------------------------------
            # Evaluation run tags
            # ----------------------------------------------------

            set_run_tags(
                description="Evaluation Metrics",
                product=PRODUCT_NAME,
                run_type="evaluation",
                run_scope="scoring",
                run_id=parent_run_id,
                parent_run_id=parent_run_id,
                capability=(
                    "abstractive summarization evaluation"
                ),
                dataset_version=dataset_version,
                git_sha="testing_git",
            )

            update_run_tags(
                evaluation_run_id=evaluation_run_id
            )

            # ====================================================
            # 11. EVALUATE MODELS
            # ====================================================

            eval_results = evaluate_product(
                PRODUCT_NAME,
                GROUNDTRUTH_DIR,
                output_file,
                summary_column=SUMMARY_COLUMN_IN_GROUNDTRUTH,
                summary_column_in_hf=SUMMARY_COLUMN_IN_HF_RESULT,
                source_doc_column=SOURCE_DOC_COLUMN,
            )

            # ====================================================
            # 12. ADD QUALITY LABELS
            # ====================================================

            eval_labels_df = add_quality_labels(
                eval_results
            )

            # ====================================================
            # 13. CALCULATE COMPOSITE SCORE
            # ====================================================

            eval_df_composite = (
                calculate_composite_score(
                    eval_labels_df
                )
            )

            # ====================================================
            # 14. PREPARE CHAMPION / CHALLENGER
            # ====================================================

            promotion_info = (
                prepare_promotion_metadata(
                    eval_df_composite
                )
            )

            # ====================================================
            # 15. PROMOTION SUMMARY
            # ====================================================

            champion = promotion_info.get(
                "champion"
            )

            challenger = promotion_info.get(
                "challenger"
            )

            champion_score = promotion_info.get(
                "champion_score"
            )

            challenger_score = promotion_info.get(
                "challenger_score"
            )

            print(
                f"Champion: {champion}"
            )

            print(
                f"Challenger: {challenger}"
            )

            # ----------------------------------------------------
            # Evaluation-run tags
            # ----------------------------------------------------

            if champion:

                update_run_tags(
                    champion_model=champion
                )

            if challenger:

                update_run_tags(
                    challenger_model=challenger
                )

            update_run_tags(
                promotion_status="completed"
            )

            # ----------------------------------------------------
            # Evaluation-run metrics
            # ----------------------------------------------------

            if champion_score is not None:

                mlflow.log_metric(
                    "champion_composite_score",
                    float(champion_score),
                )

            if challenger_score is not None:

                mlflow.log_metric(
                    "challenger_composite_score",
                    float(challenger_score),
                )

            # ====================================================
            # 16. SAVE EVALUATION OUTPUT
            # ====================================================

            if eval_df_composite is not None:

                eval_out_file = save_eval_metric(
                    eval_df_composite,
                    PRODUCT_NAME,
                )

                # ------------------------------------------------
                # Log evaluation metrics / parameters
                # ------------------------------------------------

                row = (
                    eval_df_composite
                    .iloc[0]
                    .to_dict()
                )

                metrics = {}
                params = {}

                for key, value in row.items():

                    try:

                        metrics[key] = float(value)

                    except (
                        ValueError,
                        TypeError,
                    ):

                        params[key] = str(value)

                if metrics:

                    log_metrics(metrics)

                if params:

                    log_eval_quality_tags(
                        params
                    )

                log_artifact(
                    eval_out_file
                )

                update_run_tags(
                    evaluation_status="complete"
                )

            else:

                update_run_tags(
                    evaluation_status="failed",
                    promotion_status="failed",
                )

                return False

            # ====================================================
            # 17. LINK EVALUATION TO REGISTERED MODEL VERSIONS
            # ====================================================

            for registered_model in registered_models:

                model_name = (
                    registered_model[
                        "model_name"
                    ]
                )

                registered_name = (
                    registered_model[
                        "registered_name"
                    ]
                )

                version = (
                    registered_model[
                        "version"
                    ]
                )

                # ------------------------------------------------
                # Evaluation run ID
                # ------------------------------------------------

                client.set_model_version_tag(
                    name=registered_name,
                    version=version,
                    key="evaluation_run_id",
                    value=evaluation_run_id,
                )

                # ------------------------------------------------
                # Parent run ID
                # ------------------------------------------------

                client.set_model_version_tag(
                    name=registered_name,
                    version=version,
                    key="parent_run_id",
                    value=parent_run_id,
                )

                # ------------------------------------------------
                # Dataset version
                # ------------------------------------------------

                client.set_model_version_tag(
                    name=registered_name,
                    version=version,
                    key="dataset_version",
                    value=dataset_version,
                )

                # ------------------------------------------------
                # Get promotion role
                # ------------------------------------------------

                promotion_role = (
                    promotion_info[
                        "aliases"
                    ].get(model_name)
                )

                # ------------------------------------------------
                # Store promotion role
                # ------------------------------------------------

                if promotion_role:

                    client.set_model_version_tag(
                        name=registered_name,
                        version=version,
                        key="promotion_role",
                        value=promotion_role,
                    )

                # ------------------------------------------------
                # Store composite score
                # ------------------------------------------------

                model_rows = eval_df_composite[
                    eval_df_composite[
                        "model_name"
                    ] == model_name
                ]

                if not model_rows.empty:

                    composite_score = float(
                        model_rows[
                            "composite_score"
                        ].iloc[0]
                    )

                    client.set_model_version_tag(
                        name=registered_name,
                        version=version,
                        key="composite_score",
                        value=str(
                            composite_score
                        ),
                    )

                    # ------------------------------------------------
                    # UC alias
                    # ------------------------------------------------

                    if (
                        USE_UNITY_CATALOG
                        and promotion_role
                    ):

                        client.set_registered_model_alias(
                            name=registered_name,
                            alias=promotion_role,
                            version=version,
                        )

            # ====================================================
            # 18. FINAL PIPELINE STATUS
            # ====================================================

            execution_time = (
                time.time() - start_time
            )

            mlflow.log_metric(
                "execution_time_seconds",
                execution_time,
            )

            mlflow.log_metric(
                "documents_processed",
                len(df),
            )

            mlflow.log_metric(
                "summaries_generated",
                len(summary_results),
            )

            update_run_tags(
                pipeline_status="completed"
            )

    return True


if __name__ == "__main__":
    main()
```
