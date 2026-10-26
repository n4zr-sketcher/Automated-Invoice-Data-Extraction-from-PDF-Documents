# Automated Invoice Data Extraction from PDF Documents

This guide explains how to set up, validate, build, and run the UiPath invoice extraction process.

> **Current readiness:** The workflow and project manifest are configured for local OCR, but the current CLI validation/build environment has reported that `UiPath.DocumentUnderstanding.OCR.LocalServer` is not recognized even though the package is present in the manifest and resolved dependency cache. Do not treat this project as production-ready until Studio validation and `uip rpa build` both complete without errors.

## 1. Project overview

The process scans PDF invoices directly inside the `Input` folder, sends them through the configured PDF OCR activity, identifies invoice number, invoice date, vendor, and amount, validates the extracted fields, and exports the results to a CSV file.

Under normal processing, the workflow attempts to add one row per PDF. There is no per-file exception handler, so an OCR, parsing, or file-processing exception can stop the run before later PDFs are recorded.

The workflow records one row per PDF successfully reached by the parsing step with these columns:

- `SourceFile`
- `InvoiceNumber`
- `InvoiceDate`
- `Vendor`
- `Amount`
- `Status`
- `ValidationReason`

The `Status` value is `Valid` when all required fields are found and valid. Otherwise, it is `Invalid`, and `ValidationReason` describes the missing or invalid fields.

## 2. Prerequisites

Install or verify the following:

- Windows 10 or later
- UiPath Studio 26.0.204 or compatible later version
- A UiPath Windows Process project
- The `UiPath.DocumentUnderstanding.OCR.LocalServer` package resolved in the project
- The DU.LocalServer runtime installed and available on the execution machine
- Permission to read PDF files from the project directory
- Permission to create the `Output` directory and write the result CSV

Package resolution and local runtime availability are separate prerequisites. Verify both: (1) the project dependency resolves in Studio and at build time, and (2) the local OCR runtime is available on the execution machine. Passing one check does not prove the other.

The project currently targets the Windows framework and uses Visual Basic expressions. `UiPath.PDF.Activities` and `UiPath.OCR.Activities` are currently preview package versions. For reproducible controlled testing, use the exact dependency versions listed in `project.json`; do not silently upgrade preview packages during setup.

## 3. Project dependencies

The project uses these dependencies:

```text
UiPath.System.Activities 26.10.0
UiPath.PDF.Activities 4.6.0-preview
UiPath.OCR.Activities 4.6.0-preview
UiPath.DocumentUnderstanding.OCR.LocalServer 2.0.1
```

To install or repair the dependencies in Studio:

1. Open the project in UiPath Studio.
2. Select **Manage Packages**.
3. Use the **Official** package source.
4. Install or verify the packages listed above.
5. Click **Save**.
6. Close and reopen the project so Studio refreshes the dependency cache.

Do not add a cloud OCR API key when using local OCR mode. Use the term `API key` consistently in logs and troubleshooting; local mode should not require one.

## 4. Local OCR configuration

The workflow uses the UiPath Document OCR activity with local-server mode enabled.

In `Main.xaml`, the OCR engine is configured with:

```xml
<ocr:UiPathDocumentOCR
    DisplayName="Local UiPath Document OCR"
    Image="[Image]"
    UseLocalServer="True"
    Timeout="100000" />
```

Confirm the following in Studio:

- `UseLocalServer` is `True`.
- No cloud `ApiKey` is required.
- No cloud `Endpoint` is required.
- DU.LocalServer is installed and available on the robot machine.

The local OCR worker may start only while the workflow is processing a document. It is normal for no OCR process or Windows service to be visible before execution.

## 5. Input and output folders

The workflow creates and uses folders relative to the project execution directory:

```text
Input\
Output\
```

Setup steps:

1. Open the project folder.
2. Create an `Input` folder if it does not already exist.
3. Copy at least one invoice PDF directly into `Input`.
4. Leave the PDFs in their original location; the workflow reads them without changing them.
5. The workflow creates `Output` automatically.

Important behavior:

- Only PDFs directly inside `Input` are scanned. PDFs in nested subfolders are ignored.
- The workflow uses `Environment.CurrentDirectory` to resolve these folders. Run it with the project directory as the working directory, or verify the resolved paths in the Output panel.
- The workflow is sequential and does not intentionally handle an empty `Input` folder. With no PDFs, `invoiceTable` remains uninitialized because it is created inside the loop; `OutputDataTable` may therefore fail instead of generating an empty CSV. Treat an empty `Input` folder as an unsupported negative test until the workflow is changed to initialize the table before the loop.

The result file is written to:

```text
Output\InvoiceResults.csv
```

Each successful run rewrites this file; it does not append to the previous run's results. Preserve any prior result externally before running another batch if you need to retain it.

## 6. Validate the workflow

From UiPath Studio:

1. Open `Main.xaml`.
2. Select **Validate File** or validate the project from the Studio ribbon.
3. Review any package, OCR, or activity errors.
4. Confirm that the local OCR dependency is loaded after reopening Studio.

From PowerShell, verify that the UiPath CLI is installed and run the validation from the project directory:

```powershell
uip --version

$project = "C:\Users\Dell\OneDrive\Documents\UiPath\Automated Invoice Data Extraction from PDF Documents"
Set-Location $project

uip rpa validate `
  --file-path "Main.xaml" `
  --project-dir $project `
  --output json
```

If the project is stored elsewhere, replace the value of `$project`. Record the CLI version and retain the complete JSON output from validation and build. The current observed failure is: `Please make sure you have the UiPath.DocumentUnderstanding.OCR.LocalServer package installed in order to use local server mode.` Record whether this message appears in `uip rpa validate`, `uip rpa build`, or both. The workflow uses the current working directory at runtime, so setting the location explicitly prevents `Input` and `Output` from being resolved in an unexpected folder.

## 7. Build the project

After validation reports no errors, build the complete project:

```powershell
uip rpa build `
  $project `
  --output json
```

A successful build confirms that the project compiles and its dependencies can be resolved by the CLI. It does not prove that local OCR can execute successfully at runtime. Because this project currently has a known local-OCR package-loader validation issue, do not proceed to runtime testing until this build completes without errors.

## 8. Run the process from Studio

For the first test run:

1. Place one known invoice PDF in `Input`.
2. Confirm DU.LocalServer is installed and available.
3. Open `Main.xaml` in Studio.
4. Click **Run**.
5. Monitor the Output panel.
6. Confirm that the workflow completes without OCR or file-access errors.
7. Open `Output\InvoiceResults.csv` and review the extracted fields.

For additional invoices, copy the PDFs into `Input` and run the process again. Each run rewrites `Output\InvoiceResults.csv`; it does not append to the previous results.

## 9. Run the process with the UiPath CLI

Run only after the project has passed validation and build. The following example assumes `$project` was defined in Section 6:

```powershell
Set-Location $project

uip rpa run `
  --file-path "Main.xaml" `
  --project-dir $project `
  --skip-build `
  --output json
```

The `--skip-build` option is appropriate when the project build has just passed. If the project has not been built successfully, omit `--skip-build` so the CLI performs its normal build step.

This command runs the workflow locally through the UiPath CLI; it does not submit an Orchestrator job. Inspect the JSON response and confirm a successful exit code, a completed run status, and no reported workflow or OCR errors. Do not rely only on the outer command result or on a single log line.

## 10. Expected output

For a valid invoice, an output row should resemble:

```text
SourceFile,InvoiceNumber,InvoiceDate,Vendor,Amount,Status,ValidationReason
invoice-001.pdf,INV-001,2026-10-20,Example Vendor,1250.00,Valid,
```

For an invoice with missing fields, the row should contain `Invalid` and a reason, for example:

```text
invoice-002.pdf,,2026-10-20,Example Vendor,,Invalid,Invoice number missing; Amount missing or invalid
```

These are illustrative rows, not guaranteed output from every invoice. OCR quality depends on the document scan, image resolution, layout, language, and print quality. Review invalid records before downstream processing. Vendor names or other OCR values containing commas, quotes, or line breaks should also be checked in the generated CSV for correct escaping.

## 11. Troubleshooting

### Local OCR package is reported as missing

1. Open **Manage Packages** in Studio.
2. Confirm `UiPath.DocumentUnderstanding.OCR.LocalServer 2.0.1` is installed.
3. Click **Save**.
4. Close and reopen Studio.
5. Reopen the project and run **Validate File** again.
6. Confirm that `project.json` contains the local OCR dependency.

The project has previously shown this validation message even though the package was present in the NuGet cache and resolved dependency metadata:

```text
Please make sure you have the UiPath.DocumentUnderstanding.OCR.LocalServer package installed in order to use local server mode.
```

If the message persists after repairing the package and reopening Studio, collect the validation output and package versions for UiPath support. Do not add `UiPath.CoreIPC` manually unless UiPath support specifically directs you to do so; CoreIPC is already supplied transitively by the OCR package in this project.

### OCR authentication error

If the workflow reports an API-key or cloud endpoint error, verify that the OCR activity has `UseLocalServer=True`. Local mode should not require a cloud OCR API key.

### No PDFs are processed

Check that:

- The PDFs are directly inside the project `Input` folder.
- The files use the `.pdf` extension.
- The workflow is running with the project directory as its execution directory.
- The robot account has read permission for the `Input` folder.

### Output CSV is missing

Check that:

- The workflow completed without an exception.
- The robot account can create and write to the `Output` folder.
- The result path is `Output\InvoiceResults.csv` under the execution directory.
- At least one PDF was found in `Input`.

### Fields are blank or invalid

Review the OCR text and the invoice layout. The current regular expressions are layout-sensitive and recognize common labels only when they appear in expected forms:

- Invoice number: `Invoice No`, `Invoice Number`, or `Invoice #`, followed by a value.
- Date: `Date` or `Invoice Date`, followed by a date using separators such as `/` or `-`.
- Vendor: `Vendor: value`, `Supplier: value`, or `From: value` at the beginning of a line.
- Amount: `Total`, `Total Due`, `Total Amount`, or `Amount Due`, followed by an unsigned numeric value.

Different labels, spacing, line wrapping, or layouts may cause a visible field to be classified as missing. Update the regular expressions in the `Parse and Validate Invoice Fields` activity when supporting additional invoice formats.

The current amount pattern does not capture a leading minus sign. A negative amount will therefore usually be treated as missing rather than explicitly rejected as negative.

The CSV is generated from OCR-derived values. Review vendor names and other fields containing commas, quotation marks, or line breaks for correct CSV escaping.

## 12. Recommended test sequence

1. Test with one clear, text-based invoice PDF.
2. Test with one scanned invoice PDF that requires OCR.
3. Test an invoice with a missing invoice number.
4. Test an invoice with an invalid date.
5. Test an invoice with a missing amount.
6. Test an invoice with a malformed amount, including a negative value, and confirm how it is classified.
7. Test one deliberately failing PDF followed by one valid PDF. Record whether the failure stops processing before the later file.
8. Review both valid and invalid rows in `InvoiceResults.csv`.
9. Only then process a larger batch.

## 13. Main project files

- `Main.xaml` — entry-point workflow
- `project.json` — project settings and package dependencies
- `Input\` — source invoice PDFs
- `Output\InvoiceResults.csv` — extracted and validated invoice records

## 14. Current validation note

At the time this guide was created, the workflow was configured for local OCR, but the CLI validation/build environment continued to report that `UiPath.DocumentUnderstanding.OCR.LocalServer` was not recognized, even though the package was present in the project manifest and resolved dependency cache.

Before production use, confirm that Studio’s native validation and the CLI both pass cleanly after repairing or refreshing the local OCR package. Do not treat a successful package installation alone as proof that the project is ready to run.
