Implemenetd :

✅ MLflow integration is complete.
✅ Parent and nested evaluation runs are implemented.
✅ Model validation is implemented.
✅ Model registration is implemented (Databricks + UC toggle).
✅ Model version tags (source run ID, dataset version, etc.) are implemented.
✅ promotion_utils.py has been created, but promotion logic is only partially implemented.
✅ lineage_utils.py has been created, but lineage querying is largely pending.
⏳ Champion/challenger selection is in progress (composite score).
⏳ Unity Catalog-specific lineage will be enabled later using the existing toggle.


1. Design lineage data model and API contract (do this first)

Purpose: Decide what a lineage query should return.

You don't write much code here. You define a response model, for example:

{
  model_name,
  registered_name,
  version,
  source_run_id,
  evaluation_run_id,
  dataset_version,
  validation_status,
  registration_status,
  model_card,
  metrics,
  artifacts,
  uc_lineage (optional)
}

This becomes the contract for both the notebook and the API.

File:

lineage_utils.py (response object/dataclass)
README or design notes
2. Implement MLflow lineage resolver

This is the core implementation.

Given:

registered_model_name
version

it should resolve:

source run ID
parent run
evaluation run
dataset version
validation status
metrics
artifacts
model card
registered model information

Initially this can use only MLflowClient.

Later, if UC is enabled:

augment with Unity Catalog lineage.

File:

lineage_utils.py

Functions such as:

resolve_lineage(...)
get_model_version(...)
get_source_run(...)
get_evaluation_run(...)
build_lineage(...)
3. Expose lineage on API

Once lineage_utils.py works:

API becomes very thin.

GET /lineage/{model}/{version}

Internally:

lineage_utils.resolve_lineage(...)

returns JSON.

If you're using FastAPI, this is just one endpoint.

4. Notebook for ad hoc queries

Very useful for debugging.

Example:

from lineage_utils import resolve_lineage

resolve_lineage(
    model="BioBART",
    version=4
)

Returns a dataframe/dictionary.

No API required.

5. Documentation

Document:

request
model
version
response
source_run
evaluation_run
dataset_version
metrics
validation
artifacts

Also explain:

Without UC
↓

MLflow lineage only

With UC

↓

MLflow
+
Unity Catalog lineage
Suggested implementation order
✅ Finish promotion_utils.py (champion/challenger).
✅ Finalize composite score.
✅ Design the lineage response model.
✅ Implement the MLflow lineage resolver in lineage_utils.py.
✅ Create the notebook for testing lineage.
✅ Add the API endpoint.



#########################
########################

lineage data model is not the implementation—it's the contract of what your lineage service promises to return.

Think of it this way:

MLflow stores information in different places (runs, model versions, metrics, tags, artifacts).
Your lineage resolver collects all of that.
The JSON is simply the final combined view returned to the notebook/API.

For your implementation, I'd proceed as follows.

Step 1 — Design the lineage data model (No coding initially)

This is the blueprint.

Before writing any code, decide:

"If somebody asks for model version 3, what information should I return?"

Don't think about MLflow APIs yet.

Think about the consumer.

For example:

Model
│
├── name
├── version
├── alias
├── description
├── registration_time
│
Run
│
├── source_run_id
├── parent_run_id
├── evaluation_run_id
├── experiment_name
│
Dataset
│
├── dataset_version
├── product
├── therapeutic_area
│
Validation
│
├── validation_status
├── validation_time
│
Metrics
│
├── ROUGE
├── BLEU
├── METEOR
├── BERTScore
├── Composite Score
│
Artifacts
│
├── model_card
├── evaluation_report
├── summary_output
│
Promotion
│
├── Champion
├── Challenger
├── Promotion decision
│
Unity Catalog
│
├── input tables
├── output tables
└── lineage graph



It is not stored anywhere.

You will build it.

Step 2 — Identify where every field comes from

This is the important exercise.

For every field ask:

Where does it live?

Example:

Field	Source
model_name	Registered Model
version	Registered Model
alias	UC
source_run_id	Model Version Tag
evaluation_run_id	Run Tag
validation_status	Run Tag
dataset_version	Run Tag
metrics	Evaluation Run
model_card	Artifact
summary	Artifact

Once you know the source of every field,

implementing the resolver becomes easy.

Step 3 — Build the MLflow lineage resolver

Now create functions.

Instead of one huge function:

resolve_lineage()

break it into smaller helpers.

Example:

resolve_lineage()

↓

get_registered_model()

↓

get_source_run()

↓

get_evaluation_run()

↓

get_metrics()

↓

get_artifacts()

↓

combine_everything()

Every helper has only one job.

Step 4 — Combine everything

Now create the final JSON.

Suppose

get_source_run()

returns

run_id
dataset_version
validation_status

and

get_metrics()

returns

ROUGE
BLEU
Composite

Simply merge them.

Final response becomes

{
   "model": {...},
   "run": {...},
   "dataset": {...},
   "metrics": {...},
   "artifacts": {...}
}
Step 5 — Notebook

Notebook should simply do

resolve_lineage(
    model="BioBART",
    version=4
)

and print the JSON.

No FastAPI yet.

Just ensure the resolver works.

Step 6 — API

Now the API becomes tiny.

Instead of writing MLflow code again,

the endpoint simply calls

resolve_lineage()

and returns the JSON.

Step 7 — Unity Catalog

This is why you already added the UC toggle.

Current implementation

resolve_lineage()

↓

MLflow

Later

resolve_lineage()

↓

MLflow

↓

if UC enabled

↓

UC System Tables

↓

append UC lineage

No redesign required.

Step 8 — Documentation

Finally document

Input

model_name
version

Output

Model

↓

Run

↓

Evaluation

↓

Artifacts

↓

Metrics

↓

Promotion

↓

UC lineage

This becomes your API documentation.

The order I would actually implement
Finish composite score.
Finish promotion_utils (Champion/Challenger).
Design the lineage JSON contract.
Create helper functions in lineage_utils.py.
Build resolve_lineage().
Test it in a notebook.
Add the API endpoint.
Extend it with Unity Catalog when UC is available.

This sequence minimizes rework because each step builds on the previous one, and the API layer stays very thin by reusing the resolver.



###############
##############


Suggested Lineage JSON Contract (Inference Pipeline)


{
  "model": {
    "registered_name": "BioBART",
    "version": 3,
    "alias": "Champion",
    "description": "Abstractive summarization model",
    "registration_time": "...",
    "validation_status": "Passed"
  },

  "pipeline": {
    "source_run_id": "...",
    "evaluation_run_id": "...",
    "experiment_name": "...",
    "run_type": "summarization",
    "pipeline_version": "v1.0"
  },

  "dataset": {
    "dataset_version": "...",
    "product": "ABC",
    "therapeutic_area": "Respiratory",
    "start_date": "...",
    "end_date": "...",
    "documents_processed": 120
  },

  "inference": {
    "model_name": "BioBART",
    "chunk_size": 4000,
    "max_tokens": 3000
  },

  "evaluation": {
    "rouge1": 0.48,
    "rouge2": 0.31,
    "rougeL": 0.45,
    "bleu": 0.28,
    "meteor": 0.39,
    "bert_score": 0.91,
    "factual_consistency": 0.87,
    "composite_score": 0.83
  },

  "promotion": {
    "candidate": true,
    "promotion_status": "Champion",
    "promotion_reason": "Highest composite score"
  },

  "artifacts": {
    "summary_file": "...",
    "evaluation_report": "...",
    "model_card": "..."
  },

  "unity_catalog": {
    "enabled": false,
    "lineage": null
  }
}


Why this structure?

It separates concerns:

model → Registry information.
pipeline → MLflow runs.
dataset → What was summarized.
inference → Configuration used.
evaluation → Metrics.
promotion → Champion/Challenger decision.
artifacts → Files.
unity_catalog → Future extension.


2. What should go into promotion_utils.py?

Think of promotion_utils.py as answering one question:

"Given the evaluation results, which model should become Champion?"

It should not know anything about inference or evaluation implementation. It should only work with the evaluation results.

A good structure would be:

promotion_utils.py

├── calculate_composite_score()
│
├── rank_models()
│
├── select_champion()
│
├── select_challenger()
│
├── assign_aliases()
│
├── compare_with_existing_champion()   (later, when UC aliases are used)
│
└── promote_model()                    (later)
Functions

calculate_composite_score(df)

Computes the weighted composite score for each model.
Adds a composite_score column.

rank_models(df)

Sorts models by composite score (highest first).

select_champion(df)

Returns the top-ranked model.

select_challenger(df)

Returns the second-ranked model (if available).

assign_aliases(champion, challenger)

For now, simply returns:
{
    champion_model: "Champion",
    challenger_model: "Challenger"
}

Later, when Unity Catalog aliases are enabled, this function can actually call:

client.set_registered_model_alias(...)

compare_with_existing_champion()

Placeholder for the future.
Once UC is available, compare the new model against the current Champion before changing aliases.
Suggested flow
Evaluation DataFrame
        │
        ▼
calculate_composite_score()
        │
        ▼
rank_models()
        │
        ▼
select_champion()
        │
        ▼
select_challenger()
        │
        ▼
assign_aliases()
        │
        ▼
register_model()

This keeps the promotion logic isolated, reusable, and ready for the future UC alias implementation.


#############

#############


Flow
Evaluation DataFrame
        │
        ▼
calculate_composite_score()
        │
        ▼
rank_models()
        │
        ▼
select_champion()
        │
        ▼
select_challenger()
        │
        ▼
assign_aliases()
        │
        ▼
return promotion_result
        │
        ▼
register_model(...)
1. Rank Models
def rank_models(eval_df):
    """
    Sort models based on composite score.
    """

    ranked_df = (
        eval_df
        .sort_values(
            by="composite_score",
            ascending=False,
        )
        .reset_index(drop=True)
    )

    return ranked_df
2. Select Champion
def select_champion(ranked_df):
    """
    Select the best performing model.
    """

    if ranked_df.empty:
        return None

    return ranked_df.iloc[0]
3. Select Challenger
def select_challenger(ranked_df):
    """
    Select second best model.
    """

    if len(ranked_df) < 2:
        return None

    return ranked_df.iloc[1]
4. Assign Aliases

For now don't touch UC.

Just decide aliases.

def assign_aliases(
    champion,
    challenger,
):
    """
    Prepare alias mapping.
    """

    aliases = {}

    if champion is not None:

        aliases[
            champion["model_name"]
        ] = "Champion"

    if challenger is not None:

        aliases[
            challenger["model_name"]
        ] = "Challenger"

    return aliases
5. Prepare Promotion Result

This is the function your pipeline should consume.

def prepare_promotion_result(
    ranked_df,
):
    """
    Prepare promotion information.
    """

    champion = select_champion(ranked_df)

    challenger = select_challenger(ranked_df)

    aliases = assign_aliases(
        champion,
        challenger,
    )

    return {

        "ranked_models": ranked_df,

        "champion": champion,

        "challenger": challenger,

        "aliases": aliases,
    }
Pipeline

Then your pipeline becomes

eval_df = calculate_composite_score(
    eval_df,
)

ranked_df = rank_models(
    eval_df,
)

promotion_result = prepare_promotion_result(
    ranked_df,
)

Now, while registering:

alias = promotion_result["aliases"].get(
    model_name,
    "",
)

register_model(
    ...
    alias=alias,
)
Why I prefer this design

Notice that:

Promotion utilities never call MLflow.
Promotion utilities only make business decisions.
Registration utilities handle MLflow registration.
Later, when Unity Catalog aliases are enabled, you only need to extend register_model() to apply the alias—promotion_utils.py stays unchanged.

This separation keeps each module focused on a single responsibility and makes the code easier to test



#############

#########

Recomended archictouere wrt to UC


evaluation.py
        │
        ▼
promotion_utils.py
        │
        ▼
Promotion Decision
        │
        ▼
register_model.py
        │
        ├── if use_unity_catalog:
        │       set_registered_model_alias(...)
        │       (future: UC-specific logic)
        │
        └── else:
                skip alias / use workspace logic

Where should the UC toggle be?

Only in code that actually interacts with MLflow/UC, for example:

if use_unity_catalog:
    client.set_registered_model_alias(...)
else:
    # Workspace Registry or no alias

or later in lineage_utils.py:

if use_unity_catalog:
    # Query UC system tables
else:
    # Build lineage using MLflow only
So your current flow should be
Evaluation DataFrame
        │
        ▼
promotion_utils
        │
        ▼
Champion = BioBART
Challenger = PEGASUS
Aliases = {...}
        │
        ▼
register_model(...)
        │
        ▼
if use_unity_catalog:
      apply aliases
else:
      skip


This is production-ready because:

promotion_utils.py contains pure business logic (easy to test).
register_model.py contains registry-specific logic (UC toggle).
lineage_utils.py contains lineage-specific logic (MLflow now, UC later).




      




#######
✅ Enhance with Unity Catalog lineage when UC is enabled.

This order lets you complete everything that depends only on MLflow now, while keeping the UC integration as a straightforward extension later.
