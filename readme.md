def log_transformer_model(
    model_name,
    model_pipeline,
):
    """
    Log a Hugging Face/Transformer model to MLflow.
    """

    try:
        mlflow.transformers.log_model(
            transformers_model=model_pipeline,
            artifact_path=model_name,
        )

        logger.info(
            "Successfully logged transformer model: %s",
            model_name,
        )

        return True

    except Exception as e:

        logger.exception(
            "Failed to log transformer model '%s': %s",
            model_name,
            str(e),
        )

        # Optional MLflow tag
        mlflow.set_tag(
            "model_logging_status",
            "failed",
        )

        mlflow.set_tag(
            "model_logging_error",
            str(e),
        )

        return False
