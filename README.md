# GL Reconciliation Performer

### REFrameWork Template ###
**Robotic Enterprise Framework**

* Built on top of *Transactional Business Process* template
* Uses *State Machine* layout for the phases of automation project
* Offers high level logging, exception handling and recovery
* Keeps external settings in *Config.xlsx* file and Orchestrator assets
* Pulls credentials from Orchestrator assets and *Windows Credential Manager*
* Gets transaction data from Orchestrator queue and updates back status
* Takes screenshots in case of system exceptions


### Role in the Spectre Agent Ecosystem ###

This performer is part of a multi-agent AI system built on UiPath Maestro and Spectre AI agents. When this bot fails, it does not require manual developer intervention — instead it triggers one of two AI agents depending on the type of failure:

```
Orchestrator Queue
      |
      v
GL Reconciliation_Performer
      |
      |-- Application Exception --> Maestro --> SpectreCodingAgent
      |                                              - Finds GitHub repo (topic: 3202)
      |                                              - Scans XAML files
      |                                              - Opens draft PR with fix
      |
      |-- Business Rule Exception --> Maestro --> SpectreInvestigationAgent
                                                       - Fetches Orchestrator logs
                                                       - Posts summary to Slack
                                                       - Asks Finance team what to do
```

**Process Number:** ICSAUTO-3202
**Queue Name:** GLReconciliationQueue
**Team:** Finance


### How It Works ###

1. **INITIALIZE PROCESS**
 + ./Framework/*InitiAllSettings* - Load configuration data from Config.xlsx file and from assets
 + ./Framework/*GetAppCredential* - Retrieve credentials from Orchestrator assets or local Windows Credential Manager
 + ./Framework/*InitiAllApplications* - Open and login to applications used throughout the process

2. **GET TRANSACTION DATA**
 + ./Framework/*GetTransactionData* - Fetches transactions from an Orchestrator queue defined by Config("OrchestratorQueueName") or any other configured data source

3. **PROCESS TRANSACTION**
 + *Process* - Invokes business logic workflows based on queue item data and error simulation arguments
 + *ReconcileGLAccount.xaml* - Reconciles GL account by aggregating Amount and PreviousBalance fields
 + *ValidateGLVariance.xaml* - Validates GL account variance against approved thresholds using LINQ lookup
 + ./Framework/*SetTransactionStatus* - Updates the status of the processed transaction (Orchestrator transactions by default): Success, Business Rule Exception or System Exception

4. **END PROCESS**
 + ./Framework/*CloseAllApplications* - Logs out and closes applications used throughout the process


### Queue Item Fields ###

| Field | Type | Description |
|---|---|---|
| GLAccount | String | GL account number e.g. 400099 |
| CostCenter | String | Cost center code e.g. CC-4520 |
| Amount | Decimal | Transaction amount e.g. 98450.00 |


### Error Simulation (for Spectre Agent Testing) ###

Pass these input arguments to `Main.xaml` to trigger specific error scenarios:

| Argument | Type | Description |
|---|---|---|
| in_ErrorNeeded | Boolean | Set to True to trigger an error |
| in_ErrorType | String | "Application" or "Business" |
| in_ErrorMessage | String | Optional custom message (not used by logic) |

**Triggering an Application Exception (routes to SpectreCodingAgent):**

Set `in_ErrorType = "Application"`. Process.xaml invokes `ReconcileGLAccount.xaml` which crashes with `System.NullReferenceException` because:
- `SpecificContent("PreviousBalance")` returns `Nothing` — key is not in the queue item
- `.ToString` is called on `Nothing`, throwing at runtime
- Secondary bug: `Split(".")` uses a String literal instead of `Char()` array

The SpectreCodingAgent diagnoses both bugs and opens a draft PR fixing `ReconcileGLAccount.xaml`.

**Triggering a Business Rule Exception (routes to SpectreInvestigationAgent):**

Set `in_ErrorType = "Business"`. Process.xaml invokes `ValidateGLVariance.xaml` which:
- Builds a GL variance thresholds dictionary (only specific GL accounts registered)
- LINQ lookup: `glVarianceThresholds.Where(Function(kv) kv.Key.Equals(glAccount, StringComparison.OrdinalIgnoreCase)).Select(Function(kv) kv.Value).FirstOrDefault()`
- `FirstOrDefault()` returns `0D` when GL account is not in the dictionary
- Check `amount > 0` always passes, so every transaction throws `BusinessRuleException`

The SpectreInvestigationAgent fetches logs, summarises the issue, posts to the Finance Slack channel, and asks the team whether to process manually, update the GL account configuration, or escalate.

**Example Maestro inputs for testing:**

Application exception:
```
transaction_id: 400099
process_name: ICSAUTO-3202 GL Reconciliation
diagnosis: NullReferenceException in ReconcileGLAccount.xaml — SpecificContent key PreviousBalance missing from queue item. Secondary bug: Split() using String literal instead of Char() array.
recommended_action: Add null guard on PreviousBalance lookup. Change Split(".") to Split(New Char(){"."c}) in ReconcileGLAccount.xaml.
confidence: High
```

Business exception:
```
transaction_id: 400099
description: GL account 400099 for cost center CC-4520 could not be auto-reconciled. The transaction amount exceeds the configured variance threshold and requires a human decision. Please advise whether to approve for manual reconciliation, update the GL account configuration, or escalate.
team: Finance
process_name: ICSAUTO-3202 GL Reconciliation
```
