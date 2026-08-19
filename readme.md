Endpoint
Method: GET
Path: /api/v1/models/{model_name}/versions/{version}/lineage
Request
model_name — required string
version — required string
Response
Define the response structure using your already-created lineage_response_schema.json.
Specify what each section means: model, dataset, runs, evaluation, promotion, model card, lineage.
Status codes
200 — lineage retrieved successfully
404 — model/version not found
500 — lineage retrieval failure



class LineageRequest(BaseModel):
    model_name: str
    version: str


class LineageResponse(BaseModel):
    model: ModelInfo
    dataset: DatasetInfo
    runs: RunInfo
    evaluation: EvaluationInfo
    promotion: PromotionInfo
    model_card: ModelCardInfo
    lineage: LineageInfo



class ModelInfo(BaseModel):
    name: str
    version: str
    product: str
    model_type: str



class DatasetInfo(BaseModel):
    version: str | None
    source: str | None
    source_type: str | None


src/pandora/
    api/
        ...
    core/
        lineage_resolver.py
    models/          ← if a models/schema area doesn't already exist
        lineage.py
API endpoint
     ↓
LineageRequest
     ↓
lineage_resolver
     ↓
LineageResponse





    
