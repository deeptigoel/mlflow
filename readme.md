from typing import Optional

import mlflow


def set_source_lineage_tags(
    source_delta_table_version: Optional[str] = None,
    source_file_type: Optional[str] = None,
    source_path: Optional[str] = None,
    silver_table_name: Optional[str] = None,
    uc_catalog: Optional[str] = None,
    uc_schema: Optional[str] = None,
    workspace_schema: Optional[str] = None,
) -> None:
    """
    Set MLflow source-lineage tags for Delta-table or file-based inputs.

    Delta sources:
        - source_type
        - source_delta_table_name
        - source_delta_table_version
        - source_table_catalog
        - source_table_schema
        - source_table_full_name

    File sources:
        - source_type
        - source_path

    Args:
        source_delta_table_version:
            Version of the source Delta table. If provided, the source
            is treated as a Delta source.

        source_file_type:
            File type such as 'csv' or 'excel'. Used when the source
            is not a Delta table.

        source_path:
            Path to the source file for CSV/Excel inputs.

        silver_table_name:
            Name of the Silver Delta table.

        uc_catalog:
            Unity Catalog catalog name, when applicable.

        uc_schema:
            Unity Catalog schema name, when applicable.

        workspace_schema:
            Workspace/schema name used when Unity Catalog is not applicable.
    """

    tags = {}

    # ---------------------------------------------------------
    # Delta source
    # ---------------------------------------------------------
    if source_delta_table_version is not None:

        if not silver_table_name:
            raise ValueError(
                "silver_table_name is required for a Delta source."
            )

        tags.update(
            {
                "source_type": "delta",
                "source_delta_table_name": silver_table_name,
                "source_delta_table_version": str(
                    source_delta_table_version
                ),
            }
        )

        # Unity Catalog
        if uc_catalog and uc_schema:

            tags.update(
                {
                    "source_table_catalog": uc_catalog,
                    "source_table_schema": uc_schema,
                    "source_table_full_name": (
                        f"{uc_catalog}."
                        f"{uc_schema}."
                        f"{silver_table_name}"
                    ),
                }
            )

        # Non-UC workspace/schema
        elif workspace_schema:

            tags.update(
                {
                    "source_table_schema": workspace_schema,
                    "source_table_full_name": (
                        f"{workspace_schema}."
                        f"{silver_table_name}"
                    ),
                }
            )

        # No catalog/schema information
        else:

            tags["source_table_full_name"] = silver_table_name

    # ---------------------------------------------------------
    # File source - CSV / Excel
    # ---------------------------------------------------------
    elif source_file_type:

        if not source_path:
            raise ValueError(
                "source_path is required for a file-based source."
            )

        file_type = source_file_type.lower().strip()

        if file_type in {"xlsx", "xls", "excel"}:
            normalized_type = "excel"

        elif file_type == "csv":
            normalized_type = "csv"

        else:
            normalized_type = file_type

        tags.update(
            {
                "source_type": normalized_type,
                "source_path": str(source_path),
            }
        )

    # ---------------------------------------------------------
    # Invalid / missing source information
    # ---------------------------------------------------------
    else:
        raise ValueError(
            "Either source_delta_table_version or "
            "source_file_type must be provided."
        )

    # ---------------------------------------------------------
    # Set all generated tags on the active MLflow run
    # ---------------------------------------------------------
    mlflow.set_tags(tags)

set_source_lineage_tags(
    source_delta_table_version=SOURCE_DELTA_TABLE_VERSION,
    source_file_type=SOURCE_FILE_TYPE,
    source_path=INPUT_FILE_PATH,
    silver_table_name=SILVER_TABLE_NAME,
)




