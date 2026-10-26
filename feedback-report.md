# UiPath Feedback Report

**Type:** bug  
**Priority:** normal  
**Title:** `[RPA] Local OCR package-loader failure persists after dependency repair`

## What happened

A Windows UiPath Process project configured with local UiPath Document OCR continues to fail validation and build after the local OCR dependency was repaired through the active Studio project. The package is present in `project.json`, the NuGet cache, and resolved dependency metadata, but Studio/CLI still reports that the package is missing.

## Sample prompts

1. "Use a local OCR server instead of UiPath Document Understanding cloud OCR"
2. "Run Studio validation and project build now"
3. "Open Manage Packages in Studio and repair the local OCR dependency"
4. "Rerun validation and build after the local OCR dependency repair"

## Error

Validation and build both report:

```text
Please make sure you have the UiPath.DocumentUnderstanding.OCR.LocalServer package installed in order to use local server mode.

In order to use this activity in this Studio version, please install the UiPath.CoreIPC package, version 2.0.1 or higher.
```

## Environment

- Area: RPA
- UiPath CLI: 1.204.0
- RPA tool: 1.204.2
- Studio: 26.0.204.0-preview, cloud build 26.0.204-cloud.25540
- Operating system: Windows 10 10.0.19045.0
- Project type: UiPath Windows Process
- Target framework: Windows
- Expression language: Visual Basic
- PDF package: UiPath.PDF.Activities 4.6.0-preview
- OCR package: UiPath.OCR.Activities 4.6.0-preview
- Local OCR package: UiPath.DocumentUnderstanding.OCR.LocalServer 2.0.1
- Tenant: DefaultTenant
- Organization: n4zr

## Troubleshooting

- Workflow: `Main.xaml`
- OCR configuration: `UiPathDocumentOCR` with `UseLocalServer=True`
- The local OCR package was repaired through the project package manager.
- `UiPath.CoreIpc 2.1.0-20210811-12-fix` is present transitively in the resolved dependency graph.
- The local OCR runtime successfully executed during an earlier one-PDF smoke test.
- Before the latest regex edits, the workflow ran successfully against a PDF and generated `InvoiceResults.csv`.
- After invoice-regex edits, validation and build again failed with the same package-loader diagnostics.

Last failed commands:

```powershell
uip rpa validate --file-path "Main.xaml" --project-dir "<sanitized-project-path>" --output json
uip rpa build "<sanitized-project-path>" --output json
```

## Session retrospective

- **Intent**: Build an invoice PDF extraction workflow using local OCR, extract invoice number, date, vendor, and amount, validate the fields, and export the results.
- **Outcome**: Partial — the workflow runs locally with OCR, but the post-edit validation and build gates remain blocked by the package-loader error.
- **Tool & Skill Gaps**: Package repair succeeds and dependency metadata is correct, but validation does not recognize the local OCR package. The generic CoreIPC message does not identify the actual loader problem.
- **Friction**: The project required repeated package repair, dependency alignment, and validation attempts. Runtime execution succeeded before the latest parser edits, but the post-edit gate remains blocked.
- **Top 3 Improvements**:
  1. Make validation recognize `UiPath.DocumentUnderstanding.OCR.LocalServer` when it is present in the resolved dependency graph.
  2. Replace the generic CoreIPC message with the actual missing or incompatible assembly.
  3. Add a diagnostic command showing which package assemblies were loaded for a specific activity.

## Submission status

Automatic submission through `uip feedback send` failed with:

```text
Failed to create feedback issue: {}
Check your network connection and try again.
```

This report was prepared without credentials, API keys, login identity, invoice contents, or customer data.
