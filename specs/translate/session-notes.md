# WALT Development Session Notes

These notes capture key decisions and technical details from the planning session.

---

## Project Overview

**WALT** = Web App Literature Translator (iOS app)

An offline iOS app for translating foreign literature using on-device AI. Digital PDFs only.

**Core flow:**
```
Import PDF → Extract text → Translate locally → Read in app
```

---

## Key Decisions Made

| Decision | Choice | Reason |
|----------|--------|--------|
| Platform | iOS only (React Native + Expo) | Focus, simplicity |
| Translation model | TranslateGemma 4B | Optimized for mobile, 55 languages |
| Inference engine | llama.rn | React Native binding for llama.cpp |
| PDF approach | Text extraction (not vision) | More accurate, faster, smaller download |
| OCR | Not needed | Digital PDFs have selectable text |
| Backend | None | Fully offline, privacy-focused |
| Auth | None | No sign up required |

---

## Tech Stack

```
React Native + Expo (New Architecture required)
├── llama.rn          → Local LLM inference
├── expo-file-system  → Model download, file storage
├── expo-sqlite       → Local database
├── PDFKit (Swift)    → PDF text extraction (native module)
└── expo-document-picker → Import PDFs
```

---

## Getting the TranslateGemma GGUF Model

The model is available on Ollama. Extract it like this:

```bash
# 1. Install and pull
brew install ollama
ollama serve  # In one terminal
ollama pull translategemma3:4b  # In another terminal

# 2. Find the model blob (look for the ~3GB file)
ls -lhS ~/.ollama/models/blobs/

# 3. Copy to usable file
cp ~/.ollama/models/blobs/sha256-bdbf* ~/translategemma-4b.gguf

# 4. Verify size (~3.1GB)
ls -lh ~/translategemma-4b.gguf
```

Then host the GGUF file somewhere (HuggingFace, S3, CloudFlare R2) for your app to download.

---

## llama.rn Integration

### Installation
```bash
npm install llama.rn
```

### Requirements
- React Native **New Architecture** (v0.10+)
- iOS only (no simulator - Metal limitation)
- iPhone 14+ (Apple7 GPU required)

### app.json config
```json
{
  "expo": {
    "newArchEnabled": true
  }
}
```

### Translation Service Code

```typescript
// src/services/translation.ts
import { initLlama, LlamaContext } from 'llama.rn';

let context: LlamaContext | null = null;

export async function initTranslationEngine(modelPath: string): Promise<LlamaContext> {
  context = await initLlama({
    model: modelPath,
    n_ctx: 2048,
    n_gpu_layers: 99,  // Use Metal GPU
  });
  return context;
}

export async function translate(
  text: string,
  fromLang: string,
  toLang: string,
  onToken?: (token: string) => void
): Promise<string> {
  if (!context) throw new Error('Model not loaded');

  const prompt = `You are a professional ${fromLang} to ${toLang} translator. Your goal is to accurately convey the meaning and nuances of the original text while adhering to target language grammar, vocabulary, and cultural sensitivities. Produce only the translation, without any additional explanations.

${text}`;

  const result = await context.completion({
    prompt,
    n_predict: 512,
    temperature: 0.1,
    stop: ['</s>'],
    stream: onToken ? (data) => onToken(data.token) : undefined,
  });

  return result.text;
}
```

### Model Download Service Code

```typescript
// src/services/modelManager.ts
import * as FileSystem from 'expo-file-system';

const MODEL_URL = 'https://your-host.com/translategemma-4b.gguf';
const MODEL_PATH = `${FileSystem.documentDirectory}models/translategemma-4b.gguf`;

export async function isModelDownloaded(): Promise<boolean> {
  const fileInfo = await FileSystem.getInfoAsync(MODEL_PATH);
  return fileInfo.exists;
}

export async function downloadModel(onProgress: (percent: number) => void): Promise<string> {
  if (await isModelDownloaded()) {
    return MODEL_PATH;
  }

  await FileSystem.makeDirectoryAsync(
    `${FileSystem.documentDirectory}models/`,
    { intermediates: true }
  );

  const downloadResumable = FileSystem.createDownloadResumable(
    MODEL_URL,
    MODEL_PATH,
    {},
    (progress) => {
      const percent = (progress.totalBytesWritten / progress.totalBytesExpectedToWrite) * 100;
      onProgress(percent);
    }
  );

  await downloadResumable.downloadAsync();
  return MODEL_PATH;
}
```

---

## PDF Text Extraction

**Why native module?** No good React Native library exists for PDF text extraction. The options are discontinued, pattern-based, or commercial.

**Solution:** Use iOS's built-in PDFKit via a native module.

### Swift Native Module

```swift
// ios/PDFExtractor/PDFExtractor.swift
import PDFKit

@objc(PDFExtractor)
class PDFExtractor: NSObject {

  @objc func getPageCount(_ path: String,
                          resolver: @escaping RCTPromiseResolveBlock,
                          rejecter: @escaping RCTPromiseRejectBlock) {
    guard let url = URL(string: path),
          let pdf = PDFDocument(url: url) else {
      rejecter("INVALID_PDF", "Could not open PDF", nil)
      return
    }
    resolver(pdf.pageCount)
  }

  @objc func extractPage(_ path: String,
                         pageNumber: Int,
                         resolver: @escaping RCTPromiseResolveBlock,
                         rejecter: @escaping RCTPromiseRejectBlock) {
    guard let url = URL(string: path),
          let pdf = PDFDocument(url: url),
          let page = pdf.page(at: pageNumber) else {
      rejecter("INVALID_PAGE", "Could not read page", nil)
      return
    }
    resolver(page.string ?? "")
  }

  @objc func extractAllPages(_ path: String,
                             resolver: @escaping RCTPromiseResolveBlock,
                             rejecter: @escaping RCTPromiseRejectBlock) {
    guard let url = URL(string: path),
          let pdf = PDFDocument(url: url) else {
      rejecter("INVALID_PDF", "Could not open PDF", nil)
      return
    }

    var pages: [[String: Any]] = []
    for i in 0..<pdf.pageCount {
      if let page = pdf.page(at: i) {
        pages.append([
          "pageNumber": i + 1,
          "text": page.string ?? ""
        ])
      }
    }
    resolver(pages)
  }
}
```

### TypeScript Wrapper

```typescript
// src/services/pdf.ts
import { NativeModules } from 'react-native';

const { PDFExtractor } = NativeModules;

export interface PageText {
  pageNumber: number;
  text: string;
}

export async function getPageCount(pdfPath: string): Promise<number> {
  return PDFExtractor.getPageCount(pdfPath);
}

export async function extractPage(pdfPath: string, pageNumber: number): Promise<string> {
  return PDFExtractor.extractPage(pdfPath, pageNumber - 1); // 0-indexed
}

export async function extractAllPages(pdfPath: string): Promise<PageText[]> {
  return PDFExtractor.extractAllPages(pdfPath);
}
```

---

## Why NOT Vision/Multimodal Approach

We considered passing PDF pages as images directly to TranslateGemma (it supports vision via mmproj).

**Rejected because:**
- Requires extra ~850MB mmproj file download
- Slower (image processing overhead)
- Less accurate (model reads text from image vs exact text)
- More complexity

**Vision approach is better for:** Scanned books, photos of pages, handwritten documents (future feature).

---

## Services Needed

| Service | Status | Description |
|---------|--------|-------------|
| `modelManager.ts` | Defined | Download/manage GGUF model |
| `translation.ts` | Defined | llama.rn wrapper for translation |
| `pdf.ts` | Defined | PDFKit native module wrapper |
| `bookProcessor.ts` | Defined | Queue processing, page-by-page translation |
| `db.ts` | NOT DONE | SQLite CRUD operations |

---

## Database Schema

```typescript
interface Book {
  id: string;
  title: string;
  author?: string;
  sourceLanguage: string;
  targetLanguage: string;
  totalPages: number;
  translatedPages: number;
  status: 'processing' | 'ready' | 'error';
  coverPath?: string;
  createdAt: number;
  updatedAt: number;
}

interface Page {
  id: string;
  bookId: string;
  pageNumber: number;
  originalText: string;
  translatedText?: string;
  status: 'pending' | 'translating' | 'done' | 'error';
}

interface ReadingProgress {
  bookId: string;
  currentPage: number;
  lastReadAt: number;
}

interface Preferences {
  theme: 'light' | 'dark' | 'sepia';
  fontSize: number;
  defaultSourceLanguage?: string;
  defaultTargetLanguage: string;
}
```

---

## UI Components Needed

| Component | Description |
|-----------|-------------|
| Library screen | Grid of imported books |
| Queue screen | Translation progress for all books |
| Reader screen | Apple Books-style, swipe between translation/original |
| Settings screen | Theme, font size, language defaults |
| Upload flow | Pick PDF, select language, start processing |
| Model downloader | First-launch ~3GB download with progress |

---

## Open Questions

- [ ] Which library for PDF text extraction? (Decided: PDFKit native module)
- [ ] Handle books > 500 pages? (Batch processing)
- [ ] App Store approval for 3GB model download?
- [ ] Resume interrupted model downloads?
- [ ] Background processing when app is closed?

---

## Resources

- [llama.rn GitHub](https://github.com/mybigday/llama.rn)
- [TranslateGemma on Ollama](https://ollama.com/library/translategemma)
- [TranslateGemma on HuggingFace](https://huggingface.co/google/translategemma-4b-it)
- [Apple PDFKit Documentation](https://developer.apple.com/documentation/pdfkit)
- [Hacking with Swift - PDFKit text extraction](https://www.hackingwithswift.com/example-code/libraries/how-to-extract-text-from-a-pdf-using-pdfkit)

---

## Files in This Spec

- `product-spec.md` - Product requirements, user flows, features
- `technical-spec.md` - Full technical implementation details
- `session-notes.md` - This file (conversation summary for handoff)
