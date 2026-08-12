temp_load_models() — add try/except because this is where the actual pretrained-model deserialization failure occurs. Log the specific model name and re-raise so the pipeline stops cleanly.
log_transformer_model() — keep your existing try/except to capture MLflow model-logging failures separately.

You don't need special handling for the downstream model variable error; treat it as a consequence of the failed model deserialization.

Suggested log/comment:

Investigated model deserialization failures. safetensors HeaderTooSmall occurs during pretrained-model loading; the subsequent local model error is downstream. Added exception handling during model loading and MLflow logging to capture the failing model and terminate safely.



def temp_load_models():

    models = {}

    for model_alias, hf_model_name in MODEL_NAMES.items():

        try:
            logger.info(
                "Loading model: %s",
                hf_model_name,
            )

            model_pipeline = pipeline(
                "summarization",
                model=hf_model_name,
            )

            models[model_alias] = model_pipeline

        except Exception:

            logger.exception(
                "Failed to load model '%s' (%s)",
                model_alias,
                hf_model_name,
            )

            raise

    return models
