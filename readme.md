 model_card = build_model_card(
        model_name=model_name,
        registered_name=registered_name,
        model_version=version,
        parent_run_id=parent_run_id,
        evaluation_run_id=child_run_id,
        validation_status="passed",
        dataset_version=dataset_version,
        evaluation_metrics=evaluation_metrics,
        promotion_role=promotion_role,
        composite_score=composite_score,
    )

    card_paths = save_model_card(
        model_card,
        output_dir=f"/tmp/model_cards/{model_name}_{version}",
    )

    log_model_card(card_paths)

    tag_model_version_with_card(
        client=client,
        registered_name=registered_name,
        version=version,
        model_card_path="model_card/model_card.json",
    )
