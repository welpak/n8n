# n8n
n8n issues

## Fixes Made (December 2025)

### 1. "JSON parameter needs to be valid JSON" Error in USPS Workflow
-   **Issue**: The `Get Rates1` node in the "Chilly Water Rates USPS" workflow (`workflows/3_usps.json`) was using an invalid syntax for the JSON Body parameter (`={ ... }` with unquoted keys). This caused n8n to fail parsing the JSON.
-   **Fix**: Updated the `jsonBody` to use a proper JavaScript Object Expression `={{ { ... } }}` and verified that it returns a valid structure.

### 2. Multi-Item / Batch Processing Support ("Huge Payload")
-   **Issue**: Several nodes across the workflows were using inconsistent referencing that could break batch processing (e.g., using `$item("0")` which locks to the first item).
-   **Fix**:
    -   **Main Workflow (`workflows/2_main.json`)**: Replaced instances of `$item("0").$node[...]` with `$('NodeName').item.json...`. This ensures that when multiple items are processed, each execution correctly references its corresponding item from upstream nodes, rather than always using the first item.
    -   **USPS Workflow (`workflows/3_usps.json`)**: Updated the `Get Best Rate1` (Code Node) to loop through all input items (`$input.all()`) instead of processing only the first one (`$input.first()`), ensuring all rates in a batch are processed.
    -   **General**: Preserved correct `$('NodeName').item.json` references elsewhere to ensure n8n's Paired Item functionality works as expected.

### 3. Workflow Files
-   The fixed workflows are saved in the `workflows/` directory:
    -   `workflows/1_trigger.json`: The trigger/loader workflow.
    -   `workflows/2_main.json`: The main orchestration workflow ("Chilly Water Rates").
    -   `workflows/3_usps.json`: The fixed USPS sub-workflow.
    -   `workflows/4_ups.json`: The fixed UPS sub-workflow.

These changes ensure the system can handle single requests as well as large batches without error and with correct data mapping for every item.
