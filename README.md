# n8n
n8n issues

## Fixes Made (December 2025)

### 1. "JSON parameter needs to be valid JSON" Error in USPS Workflow
-   **Issue**: The `Get Rates1` node in the "Chilly Water Rates USPS" workflow (`workflows/3_usps.json`) was using an invalid syntax for the JSON Body parameter (`={ ... }` with unquoted keys). This caused n8n to fail parsing the JSON.
-   **Fix**: Updated the `jsonBody` to use a proper JavaScript Object Expression `={{ { ... } }}` and verified that it returns a valid structure.

### 2. Multi-Item / Batch Processing Support ("Huge Payload")
-   **Issue**: Several nodes across the workflows were using inconsistent referencing (e.g., using `$item("0")`) that could break batch processing. Additionally, the Main workflow was configured to pass empty inputs (`{}`) to sub-workflows via `defineBelow`, causing sub-workflows to receive `undefined` values for critical fields like address and weight.
-   **Fix**:
    -   **Main Workflow (`workflows/2_main.json`)**:
        -   Replaced `$item("0").$node[...]` with `$('NodeName').item.json...` to ensure correct paired item referencing.
        -   Changed `Execute Workflow` nodes to use `mappingMode: "passThroughAll"` to ensure the full item data is passed to sub-workflows.
        -   Removed incorrect `.body` property access in `Dynamic Params1`, as the input data is flat (from Excel).
    -   **USPS Workflow (`workflows/3_usps.json`)**:
        -   Updated `Dynamic Params` to reference properties directly (e.g., `$json.Name` instead of `$json.body.Name`) to match the flat input structure.
        -   Updated `Get Best Rate1` (Code Node) to loop through all input items (`$input.all()`).
    -   **UPS Workflow (`workflows/4_ups.json`)**:
        -   Updated `Dynamic Params1` to reference properties directly (removed `.body`).

### 3. Workflow Files
-   The fixed workflows are saved in the `workflows/` directory:
    -   `workflows/1_trigger.json`: The trigger/loader workflow.
    -   `workflows/2_main.json`: The main orchestration workflow ("Chilly Water Rates").
    -   `workflows/3_usps.json`: The fixed USPS sub-workflow.
    -   `workflows/4_ups.json`: The fixed UPS sub-workflow.

These changes ensure the system can handle single requests as well as large batches without error and with correct data mapping for every item.
