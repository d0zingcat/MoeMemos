# Document Scanner Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an iOS native document scanner entry in the memo editor that creates one PDF attachment from a completed scan.

**Architecture:** `MemoEditorToolbar` exposes a scan button only when scanning is supported. `MemoEditor` presents a `DocumentScanner` wrapper around `VNDocumentCameraViewController`, converts pages to PDF through `ScannedDocumentPDFBuilder`, then reuses `MemoEditorViewModel.upload(data:filename:mimeType:)`.

**Tech Stack:** SwiftUI, UIKit, VisionKit, PDFKit-compatible Core Graphics PDF rendering via `UIGraphicsPDFRenderer`, XCTest.

---

## File Structure

- Create `Packages/MemoKit/Sources/MemoKit/Editor/Components/DocumentScanner.swift`: SwiftUI wrapper for VisionKit document camera, compiled only for supported iOS targets.
- Create `Packages/MemoKit/Sources/MemoKit/Editor/ScannedDocumentPDFBuilder.swift`: pure PDF generation helper.
- Modify `Packages/MemoKit/Sources/MemoKit/Editor/Components/MemoEditorToolbar.swift`: add scan action and conditional scan button.
- Modify `Packages/MemoKit/Sources/MemoKit/Editor/MemoEditor.swift`: presentation state, scanner cover, scan completion upload.
- Modify `Packages/MemoKit/Tests/MemoKitTests/MemoKitTests.swift`: replace placeholder with PDF builder tests.

---

### Task 1: PDF Builder

**Files:**
- Create: `Packages/MemoKit/Sources/MemoKit/Editor/ScannedDocumentPDFBuilder.swift`
- Test: `Packages/MemoKit/Tests/MemoKitTests/MemoKitTests.swift`

- [ ] **Step 1: Write the failing test**

Replace the placeholder test file with:

```swift
import XCTest
import UIKit
@testable import MemoKit

final class MemoKitTests: XCTestCase {
    func testScannedDocumentPDFBuilderCreatesPDFData() throws {
        let images = [
            makeImage(size: CGSize(width: 100, height: 160), color: .red),
            makeImage(size: CGSize(width: 120, height: 80), color: .blue)
        ]

        let data = try ScannedDocumentPDFBuilder.makePDFData(from: images)

        XCTAssertFalse(data.isEmpty)
        XCTAssertEqual(String(data: data.prefix(4), encoding: .ascii), "%PDF")
    }

    func testScannedDocumentPDFBuilderRejectsEmptyInput() {
        XCTAssertThrowsError(try ScannedDocumentPDFBuilder.makePDFData(from: [])) { error in
            XCTAssertEqual(error as? ScannedDocumentPDFBuilder.Error, .emptyDocument)
        }
    }

    private func makeImage(size: CGSize, color: UIColor) -> UIImage {
        let renderer = UIGraphicsImageRenderer(size: size)
        return renderer.image { context in
            color.setFill()
            context.fill(CGRect(origin: .zero, size: size))
        }
    }
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run:

```bash
swift test --package-path Packages/MemoKit --filter MemoKitTests
```

Expected: FAIL because `ScannedDocumentPDFBuilder` is not defined.

- [ ] **Step 3: Implement the PDF builder**

Create `Packages/MemoKit/Sources/MemoKit/Editor/ScannedDocumentPDFBuilder.swift`:

```swift
import UIKit

public enum ScannedDocumentPDFBuilder {
    public enum Error: Swift.Error, Equatable {
        case emptyDocument
    }

    public static func makePDFData(from images: [UIImage]) throws -> Data {
        guard !images.isEmpty else {
            throw Error.emptyDocument
        }

        let renderer = UIGraphicsPDFRenderer(bounds: CGRect(origin: .zero, size: images[0].size))
        return renderer.pdfData { context in
            for image in images {
                let pageBounds = CGRect(origin: .zero, size: image.size)
                context.beginPage(withBounds: pageBounds, pageInfo: [:])
                image.draw(in: pageBounds)
            }
        }
    }
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run:

```bash
swift test --package-path Packages/MemoKit --filter MemoKitTests
```

Expected: PASS.

---

### Task 2: Document Scanner Wrapper

**Files:**
- Create: `Packages/MemoKit/Sources/MemoKit/Editor/Components/DocumentScanner.swift`

- [ ] **Step 1: Create the scanner wrapper**

Create `Packages/MemoKit/Sources/MemoKit/Editor/Components/DocumentScanner.swift`:

```swift
import SwiftUI
import UIKit

#if canImport(VisionKit) && os(iOS) && !targetEnvironment(macCatalyst)
@preconcurrency import VisionKit

struct DocumentScanner: UIViewControllerRepresentable {
    enum Result {
        case success([UIImage])
        case cancelled
        case failure(Swift.Error)
    }

    let onComplete: (Result) -> Void

    static var isSupported: Bool {
        VNDocumentCameraViewController.isSupported
    }

    func makeUIViewController(context: Context) -> VNDocumentCameraViewController {
        let controller = VNDocumentCameraViewController()
        controller.delegate = context.coordinator
        return controller
    }

    func updateUIViewController(_ uiViewController: VNDocumentCameraViewController, context: Context) {
    }

    func makeCoordinator() -> Coordinator {
        Coordinator(parent: self)
    }

    final class Coordinator: NSObject, VNDocumentCameraViewControllerDelegate {
        private let parent: DocumentScanner

        init(parent: DocumentScanner) {
            self.parent = parent
        }

        func documentCameraViewController(_ controller: VNDocumentCameraViewController, didFinishWith scan: VNDocumentCameraScan) {
            let images = (0..<scan.pageCount).map { scan.imageOfPage(at: $0) }
            controller.dismiss(animated: true) { [parent] in
                parent.onComplete(.success(images))
            }
        }

        func documentCameraViewControllerDidCancel(_ controller: VNDocumentCameraViewController) {
            controller.dismiss(animated: true) { [parent] in
                parent.onComplete(.cancelled)
            }
        }

        func documentCameraViewController(_ controller: VNDocumentCameraViewController, didFailWithError error: Swift.Error) {
            controller.dismiss(animated: true) { [parent] in
                parent.onComplete(.failure(error))
            }
        }
    }
}
#endif
```

- [ ] **Step 2: Build MemoKit for compile validation**

Run:

```bash
swift test --package-path Packages/MemoKit --filter MemoKitTests
```

Expected: PASS and no compile errors.

---

### Task 3: Toolbar Scan Button

**Files:**
- Modify: `Packages/MemoKit/Sources/MemoKit/Editor/Components/MemoEditorToolbar.swift`

- [ ] **Step 1: Add scan inputs and conditional button**

Update `MemoEditorToolbar` to add `supportsDocumentScanning` and `onScanDocument`:

```swift
struct MemoEditorToolbar: View {
    let tags: [Tag]
    let onInsertTag: (Tag?) -> Void
    let onToggleTodo: () -> Void
    let onPickJournalingSuggestion: () -> Void
    let supportsJournalingSuggestions: Bool
    let onPickPhotos: () -> Void
    let onPickCamera: () -> Void
    let supportsDocumentScanning: Bool
    let onScanDocument: () -> Void
    let onPickFiles: () -> Void
```

Insert the scan button after the camera button:

```swift
            if supportsDocumentScanning {
                Button {
                    onScanDocument()
                } label: {
                    Image(systemName: "doc.viewfinder")
                }
            }
```

- [ ] **Step 2: Build to expose call-site updates**

Run:

```bash
swift test --package-path Packages/MemoKit --filter MemoKitTests
```

Expected: FAIL until `MemoEditor` passes the new arguments.

---

### Task 4: Memo Editor Integration

**Files:**
- Modify: `Packages/MemoKit/Sources/MemoKit/Editor/MemoEditor.swift`

- [ ] **Step 1: Add scanner state and toolbar wiring**

Add state:

```swift
    @State private var showingDocumentScanner = false
```

Pass the new toolbar arguments:

```swift
            supportsDocumentScanning: supportsDocumentScanning,
            onScanDocument: {
                showingDocumentScanner = true
            },
```

Add availability helper:

```swift
    private var supportsDocumentScanning: Bool {
#if canImport(VisionKit) && os(iOS) && !targetEnvironment(macCatalyst)
        DocumentScanner.isSupported
#else
        false
#endif
    }
```

- [ ] **Step 2: Present the scanner and upload PDF**

Add this modifier near the existing camera full-screen cover:

```swift
#if canImport(VisionKit) && os(iOS) && !targetEnvironment(macCatalyst)
        .fullScreenCover(isPresented: $showingDocumentScanner) {
            DocumentScanner { result in
                Task {
                    await handleDocumentScan(result)
                }
            }
            .edgesIgnoringSafeArea(.all)
        }
#endif
```

Add scan result handling:

```swift
#if canImport(VisionKit) && os(iOS) && !targetEnvironment(macCatalyst)
    private func handleDocumentScan(_ result: DocumentScanner.Result) async {
        switch result {
        case .success(let images):
            do {
                let data = try ScannedDocumentPDFBuilder.makePDFData(from: images)
                try await viewModel.upload(data: data, filename: scannedDocumentFilename(), mimeType: "application/pdf")
                submitError = nil
            } catch {
                submitError = error
                showingErrorToast = true
            }
        case .cancelled:
            break
        case .failure(let error):
            submitError = error
            showingErrorToast = true
        }
    }
#endif

    private func scannedDocumentFilename(date: Date = Date()) -> String {
        let formatter = DateFormatter()
        formatter.locale = Locale(identifier: "en_US_POSIX")
        formatter.dateFormat = "yyyyMMdd-HHmmss"
        return "scan-\(formatter.string(from: date)).pdf"
    }
```

- [ ] **Step 3: Run MemoKit tests**

Run:

```bash
swift test --package-path Packages/MemoKit --filter MemoKitTests
```

Expected: PASS.

---

### Task 5: App Build Verification

**Files:**
- Verify project integration only.

- [ ] **Step 1: Inspect available schemes**

Run:

```bash
xcodebuild -list -project MoeMemos.xcodeproj
```

Expected: a MoeMemos app scheme or shared schemes are listed.

- [ ] **Step 2: Build the iOS app**

Run:

```bash
xcodebuild -project MoeMemos.xcodeproj -scheme MoeMemos -destination 'generic/platform=iOS' build
```

Expected: BUILD SUCCEEDED. If the local machine lacks required signing or SDK state, capture the exact failure and run package tests as the fallback verification.

---

## Self-Review

- Spec coverage: toolbar scan button, native VisionKit scanner, PDF conversion, existing upload path, unsupported platform hiding, cancellation and error handling are covered.
- Placeholder scan: no TBD, TODO, or deferred implementation steps.
- Type consistency: `DocumentScanner.Result`, `ScannedDocumentPDFBuilder.makePDFData(from:)`, `supportsDocumentScanning`, and `handleDocumentScan(_:)` are named consistently across tasks.
