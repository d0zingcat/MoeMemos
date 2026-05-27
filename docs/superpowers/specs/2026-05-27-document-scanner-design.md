# Document Scanner Design

## Context

Moe Memos already supports adding photos, camera captures, and imported files from the memo editor. These paths all converge on `MemoEditorViewModel.upload(...)`, which creates a `StoredResource` through the current `Service`.

The new feature adds a first-class document scanning entry point using iOS native scanning. The first version will scan one or more pages and add the result as a single PDF attachment to the current memo draft.

## Goals

- Add a document scanning button to the memo editor toolbar.
- Use the iOS native document camera interface.
- Convert the completed scan into one PDF file.
- Reuse the existing attachment, local storage, and sync paths.
- Avoid exposing unavailable scanning controls on unsupported platforms.

## Non-Goals

- OCR text recognition.
- Inserting recognized text into the memo body.
- Custom document edge detection or camera UI.
- Changing the resource storage or sync model.

## User Experience

The memo editor toolbar gains a scan button, placed between the existing camera and file buttons. Tapping it presents the native iOS document scanner full screen. The user can scan multiple pages, adjust them in the system UI, and finish or cancel.

When scanning finishes, Moe Memos creates a PDF attachment named with a timestamp, for example `scan-20260527-153012.pdf`. The PDF appears in the existing attachment strip and is included when the memo is saved. Canceling the scanner returns to the editor without an error.

## Architecture

Add a small SwiftUI wrapper in `MemoKit` around `VNDocumentCameraViewController`, for example `DocumentScanner`. This wrapper is responsible only for presenting the system scanner and reporting either a completed scan, cancellation, or an error.

Add a PDF generation helper, for example `ScannedDocumentPDFBuilder`, that receives the scanned pages and renders them into a single PDF `Data` value. The helper should be independent from `MemoEditor` so PDF generation can be tested without presenting camera UI.

`MemoEditor` owns the presentation state. On scan completion it calls the PDF builder, then calls:

```swift
try await viewModel.upload(
    data: pdfData,
    filename: generatedFilename,
    mimeType: "application/pdf"
)
```

`MemoEditorToolbar` receives a new `onScanDocument` action and a capability flag such as `supportsDocumentScanning`. The scan button is shown only when document scanning is available.

## Platform And Availability

Use `VisionKit` and `VNDocumentCameraViewController` on supported iOS environments. The scan entry should not appear for unsupported targets such as Mac Catalyst or visionOS. The implementation should use compile-time checks and runtime availability checks so unsupported builds do not reference unavailable APIs.

## Data Flow

1. User taps the scan button in `MemoEditorToolbar`.
2. `MemoEditor` presents `DocumentScanner`.
3. The system scanner returns a completed scan with one or more pages.
4. `ScannedDocumentPDFBuilder` renders the pages into one PDF.
5. `MemoEditorViewModel.upload(data:filename:mimeType:)` creates a `StoredResource`.
6. The resource appears in `MemoEditorResourceView`.
7. When the memo is saved, existing create/edit actions attach the resource IDs.

For local accounts, the existing local service stores the PDF resource file on device. For remote accounts, the existing syncing remote service uploads the pending resource and associates it with the memo through the current sync path.

## Error Handling

User cancellation is treated as a no-op. PDF generation failures and upload failures reuse the memo editor's existing error toast. The generated PDF still goes through the existing upload size validation, including the 1 GB limit.

If the scanner is unavailable on a device or platform, the toolbar does not show the scan button. This avoids presenting a disabled or broken action in the normal editor flow.

## Testing And Verification

Unit-test the PDF builder with multiple synthetic images and verify that the generated PDF data is non-empty and has a PDF header.

Add a focused editor-level test if the existing test setup allows it: after a successful scan result is converted, the editor upload path receives one `application/pdf` resource.

Manually verify on an iOS device or simulator that supports VisionKit:

- The scan button appears in the memo editor toolbar.
- Canceling the scanner leaves the draft unchanged.
- Completing a multi-page scan adds one PDF attachment.
- Saving a memo with the scanned PDF works for a local account.
- The remote account path leaves the resource available for normal sync.

Manually verify on unsupported targets that the scan button is absent.
