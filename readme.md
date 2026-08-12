from typing import Any, Dict, Optional

import mlflow


def configure_mlflow() -> Dict[str, Any]:
    """
    Configure MLflow tracking, registry, and experiment settings.

    Centralizes MLflow configuration so the pipeline does not directly
    manage tracking URIs, registry URIs, or experiment setup.


    Returns
    -------
    Dict[str, Any]
        Configuration details containing the experiment name,
        tracking URI, registry URI, and Unity Catalog flag.

    Raises
    ------
    ValueError
        If experiment_name is empty or invalid.

    RuntimeError
        If MLflow configuration fails.
    """

    try:
        # ---------------------------------
        # Tracking configuration
        # ---------------------------------

       
        mlflow.set_tracking_uri(MLFLOW_TRACKING_URI)

        # ---------------------------------
        # Registry configuration
        # ---------------------------------

        if USE_UNITY_CATALOG:
            mlflow.set_registry_uri(MLFLOW_REGISTRY_UC_URI)

        elif use_unity_catalog:
            mlflow.set_registry_uri("databricks-uc")

        # ---------------------------------
        # Experiment configuration
        # ---------------------------------

        setup_experiment(experiment_name)

        return {
            "experiment_name": MLFLOW_EXPERIMENT_NAME,
            "tracking_uri": mlflow.get_tracking_uri(),
            "registry_uri": mlflow.get_registry_uri(),
            "use_unity_catalog": USE_UNITY_CATALOG,
        }

    except Exception as exc:
        raise RuntimeError(
            "Failed to configure MLflow."
        ) from exc
