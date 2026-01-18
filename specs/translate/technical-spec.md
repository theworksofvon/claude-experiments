# WALT - Technical Spec

---

## Stack

| Layer | Technology |
|-------|------------|
| iOS App | React Native + Expo (New Architecture) |
| Local LLM | TranslateGemma 4B via **llama.rn** |
| PDF Text Extraction | react-native-pdf-text or pdf-parse |
| Storage | Local (iOS Documents directory) |
| Database | SQLite (expo-sqlite) |
| File Downloads | expo-file-system |

**No backend. No cloud. No auth. Everything runs on the phone.**

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        iOS APP                                   │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                      UI Layer                            │   │
│   │   Library → Upload → Reader → Settings                   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                  Processing Layer                        │   │
│   │                                                          │   │
│   │   PDF Text Extractor → Translator (llama.rn)            │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                  Storage Layer                           │   │
│   │                                                          │   │
│   │   Model File (~2GB) │ Books DB │ Page Cache             │   │
│   │   (Documents dir)   │ (SQLite) │ (Documents dir)        │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## TranslateGemma 4B Local

### Model Details

| Spec | Value |
|------|-------|
| Parameters | 4B (5B actual) |
| Quantization | 4-bit (Q4_0) |
| Size on disk | ~2-3GB |
| RAM usage | ~3-4GB |
| Context window | 2K tokens |

### Getting the GGUF Model

TranslateGemma is available on Ollama, which uses GGUF format internally. Extract it once on your Mac:

```bash
# Install Ollama
brew install ollama

# Pull TranslateGemma 4B
ollama pull translategemma3:4b

# Find the model manifest
cat ~/.ollama/models/manifests/registry.ollama.ai/library/translategemma3/4b

# Copy the blob (replace <hash> with actual sha256 from manifest)
cp ~/.ollama/models/blobs/sha256-<hash> ~/translategemma-4b.gguf
```

Then host the GGUF file (HuggingFace, S3, CloudFlare R2, etc.) for your app to download.

### Integration: llama.rn

Using [llama.rn](https://github.com/mybigday/llama.rn) - the React Native binding for llama.cpp.

**Installation:**
```bash
npm install llama.rn
```

**Requirements:**
- React Native New Architecture (v0.10+)
- iOS only (no simulator support - Metal limitation)
- iPhone with Apple7 GPU or newer (iPhone 14+)

**Enable New Architecture in app.json:**
```json
{
  "expo": {
    "newArchEnabled": true
  }
}
```

### Translation Service

```typescript
// src/services/translation.ts
import { initLlama, LlamaContext } from 'llama.rn';

let context: LlamaContext | null = null;

export async function initTranslationEngine(modelPath: string): Promise<LlamaContext> {
  context = await initLlama({
    model: modelPath,
    n_ctx: 2048,        // Context window
    n_gpu_layers: 99,   // Use Metal (GPU acceleration)
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

export function isModelLoaded(): boolean {
  return context !== null;
}

export async function releaseModel(): Promise<void> {
  if (context) {
    await context.release();
    context = null;
  }
}
```

### Model Download Service

```typescript
// src/services/modelManager.ts
import * as FileSystem from 'expo-file-system';

// Host your extracted GGUF here after pulling from Ollama
const MODEL_URL = 'https://your-host.com/translategemma-4b.gguf';
const MODEL_PATH = `${FileSystem.documentDirectory}models/translategemma-4b.gguf`;

export async function isModelDownloaded(): Promise<boolean> {
  const fileInfo = await FileSystem.getInfoAsync(MODEL_PATH);
  return fileInfo.exists;
}

export async function getModelPath(): Promise<string> {
  return MODEL_PATH;
}

export async function downloadModel(
  onProgress: (percent: number) => void
): Promise<string> {
  // Check if already downloaded
  if (await isModelDownloaded()) {
    return MODEL_PATH;
  }

  // Create models directory
  await FileSystem.makeDirectoryAsync(
    `${FileSystem.documentDirectory}models/`,
    { intermediates: true }
  );

  // Download with progress tracking
  const downloadResumable = FileSystem.createDownloadResumable(
    MODEL_URL,
    MODEL_PATH,
    {},
    (progress) => {
      const percent = (progress.totalBytesWritten / progress.totalBytesExpectedToWrite) * 100;
      onProgress(percent);
    }
  );

  const result = await downloadResumable.downloadAsync();

  if (!result?.uri) {
    throw new Error('Download failed');
  }

  return MODEL_PATH;
}

export async function deleteModel(): Promise<void> {
  if (await isModelDownloaded()) {
    await FileSystem.deleteAsync(MODEL_PATH);
  }
}
```

### App Initialization Flow

```typescript
// Example usage in app startup
import { downloadModel, getModelPath, isModelDownloaded } from './services/modelManager';
import { initTranslationEngine } from './services/translation';

export function useModelSetup() {
  const [status, setStatus] = useState<'checking' | 'downloading' | 'loading' | 'ready' | 'error'>('checking');
  const [progress, setProgress] = useState(0);

  useEffect(() => {
    async function setup() {
      try {
        // Check if model exists
        const downloaded = await isModelDownloaded();

        if (!downloaded) {
          setStatus('downloading');
          await downloadModel(setProgress);
        }

        // Load model into memory
        setStatus('loading');
        const modelPath = await getModelPath();
        await initTranslationEngine(modelPath);

        setStatus('ready');
      } catch (error) {
        console.error('Model setup failed:', error);
        setStatus('error');
      }
    }

    setup();
  }, []);

  return { status, progress };
}
```

### Performance Expectations

| Task | Time (iPhone 15 Pro) |
|------|---------------------|
| Model load | ~5-10 seconds |
| Translate 1 page (~500 tokens) | ~10-15 seconds |
| Translate 100 pages | ~20-30 minutes |
| Translate 300-page book | ~1-1.5 hours |

**Strategy:** Process in background, show progress, allow user to read already-translated pages while rest processes.

---

## PDF Text Extraction

Extract text from digital PDFs (no OCR needed for PDFs with selectable text).

```typescript
// src/services/pdf.ts
import * as FileSystem from 'expo-file-system';

interface PDFPage {
  pageNumber: number;
  text: string;
}

interface PDFDocument {
  totalPages: number;
  pages: PDFPage[];
}

export async function extractTextFromPDF(pdfPath: string): Promise<PDFDocument> {
  // Use react-native-pdf-text or similar library
  // This is a placeholder - actual implementation depends on chosen library
  const pages = await extractPagesFromPDF(pdfPath);

  return {
    totalPages: pages.length,
    pages: pages.map((text, index) => ({
      pageNumber: index + 1,
      text,
    })),
  };
}

export async function extractSinglePage(pdfPath: string, pageNumber: number): Promise<string> {
  const document = await extractTextFromPDF(pdfPath);
  const page = document.pages.find(p => p.pageNumber === pageNumber);
  return page?.text ?? '';
}
```

**Note:** Evaluate these libraries for PDF text extraction:
- `react-native-pdf-text` - Native module for text extraction
- `pdf-parse` - Pure JS (may need polyfills for React Native)
- `@aspect-apps/react-native-pdf-extractor` - Another native option

---

## Local Storage

### Strategy

All data stored in iOS Documents directory:
- Automatically backed up to iCloud (if user has iCloud backup on)
- Persists through app restarts, phone restarts, battery dying
- Only lost if app deleted or phone physically gone

```
iOS Documents Directory
├── models/
│   └── translategemma-4b.gguf    (~2GB)
├── books/
│   ├── {book-id}/
│   │   ├── cover.jpg
│   │   └── pages/
│   │       ├── 001.jpg
│   │       ├── 002.jpg
│   │       └── ...
├── database.sqlite               (metadata, progress, translations)
└── preferences.json              (user settings)
```

### Database Schema (SQLite)

```typescript
// db/schema.ts

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
  imagePath?: string;
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

Using `expo-sqlite` for database operations.

---

## Project Structure

```
walt-ios/
├── src/
│   ├── app/                      # Screens (Expo Router)
│   │   ├── (tabs)/
│   │   │   ├── index.tsx         # Library
│   │   │   ├── queue.tsx         # Translation queue
│   │   │   └── settings.tsx      # Settings
│   │   ├── book/[id].tsx         # Reader
│   │   ├── upload.tsx            # Import flow
│   │   └── _layout.tsx
│   │
│   ├── components/
│   │   ├── reader/
│   │   │   ├── BookReader.tsx
│   │   │   ├── PageView.tsx
│   │   │   ├── SideBySideView.tsx
│   │   │   └── ReaderControls.tsx
│   │   ├── library/
│   │   │   ├── BookGrid.tsx
│   │   │   ├── BookCard.tsx
│   │   │   └── ImportButton.tsx
│   │   └── common/
│   │       ├── ProgressBar.tsx
│   │       └── ModelDownloader.tsx
│   │
│   ├── services/
│   │   ├── translation.ts        # llama.rn wrapper
│   │   ├── pdf.ts                # PDF text extraction
│   │   ├── modelManager.ts       # Download/load model
│   │   ├── bookProcessor.ts      # Translation queue processing
│   │   └── db.ts                 # SQLite operations
│   │
│   ├── hooks/
│   │   ├── useBook.ts
│   │   ├── useTranslation.ts
│   │   └── useModelStatus.ts
│   │
│   └── theme/
│       └── index.ts
│
├── ios/                          # Native modules (if needed)
│
├── app.json
├── package.json
└── tsconfig.json
```

---

## Processing Pipeline

```
User imports PDF
       │
       ▼
┌──────────────────┐
│  Extract text    │  (react-native-pdf-text)
│  from each page  │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  For each page:  │
│  1. Translate    │  ← Background processing
│  2. Save to DB   │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Update progress │
│  User can start  │
│  reading early   │
└──────────────────┘
```

### Translation Queue System

```typescript
// services/queue.ts

interface QueueItem {
  bookId: string;
  status: 'queued' | 'processing' | 'paused' | 'done' | 'error';
  priority: number;
  addedAt: number;
}

interface TranslationProgress {
  bookId: string;
  currentPage: number;
  totalPages: number;
  currentPageTokens: number;
  currentPageTotalTokens: number;
  estimatedTimeRemaining: number; // seconds
}

class TranslationQueue {
  private queue: QueueItem[] = [];
  private isProcessing: boolean = false;

  async add(bookId: string): Promise<void> {
    this.queue.push({
      bookId,
      status: 'queued',
      priority: Date.now(),
      addedAt: Date.now(),
    });
    this.processNext();
  }

  async pause(bookId: string): Promise<void> { /* ... */ }
  async resume(bookId: string): Promise<void> { /* ... */ }
  async cancel(bookId: string): Promise<void> { /* ... */ }
  async reorder(bookId: string, newPriority: number): Promise<void> { /* ... */ }

  getQueue(): QueueItem[] {
    return this.queue;
  }
}
```

### Background Processing with Progress

```typescript
// src/services/bookProcessor.ts
import { translate } from './translation';
import * as db from './db';

type ProgressCallback = (progress: TranslationProgress) => void;

export async function processBook(
  bookId: string,
  onProgress: ProgressCallback
): Promise<void> {
  const book = await db.getBook(bookId);
  const pages = await db.getPages(bookId);

  for (let i = 0; i < pages.length; i++) {
    const page = pages[i];
    if (page.status === 'done') continue;

    // Update status
    await db.updatePage(page.id, { status: 'translating' });

    // Text already extracted from PDF during import
    const originalText = page.originalText;
    const estimatedTokens = Math.ceil(originalText.length / 4);

    // Translate with streaming progress
    let tokensGenerated = 0;
    const translatedText = await translate(
      originalText,
      book.sourceLanguage,
      book.targetLanguage,
      (token: string) => {
        tokensGenerated++;
        onProgress({
          bookId,
          currentPage: i + 1,
          totalPages: pages.length,
          currentPageTokens: tokensGenerated,
          currentPageTotalTokens: estimatedTokens,
          estimatedTimeRemaining: calculateETA(i, pages.length, tokensGenerated),
        });
      }
    );

    // Save translation
    await db.updatePage(page.id, {
      translatedText,
      status: 'done',
    });
  }

  await db.updateBook(bookId, { status: 'ready' });
}

function calculateETA(currentPage: number, totalPages: number, tokensGenerated: number): number {
  // Simple estimation based on progress
  const pagesRemaining = totalPages - currentPage;
  const avgTimePerPage = 15; // seconds, adjust based on testing
  return pagesRemaining * avgTimePerPage;
}
```

### Streaming Translation

```typescript
// services/translation.ts (uses llama.rn)
import { translate } from './translation';

async function translateWithProgress(
  text: string,
  from: string,
  to: string,
  onToken: (token: string) => void
): Promise<string> {
  // The translate function from translation.ts already supports streaming
  return await translate(text, from, to, onToken);
}
```

The `translate` function in `src/services/translation.ts` (defined above) already supports streaming via the `onToken` callback using llama.rn's `stream` option.

### Queue UI

```
┌─────────────────────────────────────────┐
│  Translation Queue                   ✕  │
├─────────────────────────────────────────┤
│                                         │
│  ┌─────────────────────────────────┐   │
│  │ 📖 Crime and Punishment         │   │
│  │ ━━━━━━━━━━━━━━━━━━░░░░░  67%   │   │  ← Currently translating
│  │ Page 201 of 300 · ~45 min left  │   │
│  │ [Pause]                         │   │
│  └─────────────────────────────────┘   │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │ 📖 Les Misérables               │   │
│  │ ░░░░░░░░░░░░░░░░░░░░░░░  Queued │   │  ← Waiting
│  │ 0 of 1,462 pages                │   │
│  │ [Cancel]                        │   │
│  └─────────────────────────────────┘   │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │ 📖 Der Prozess                  │   │
│  │ ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓  Paused  │   │  ← Paused by user
│  │ 89 of 200 pages                 │   │
│  │ [Resume] [Cancel]               │   │
│  └─────────────────────────────────┘   │
│                                         │
└─────────────────────────────────────────┘
```

**Progress detail view (tap on a book):**

```
┌─────────────────────────────────────────┐
│  ← Crime and Punishment                 │
├─────────────────────────────────────────┤
│                                         │
│  Overall Progress                       │
│  ━━━━━━━━━━━━━━━━━━━━░░░░░░  67%       │
│  201 of 300 pages                       │
│                                         │
│  Current Page (201)                     │
│  ━━━━━━━━━━━━━░░░░░░░░░░░░  45%        │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │ В начале июля, в чрезвычайно    │   │  ← Live streaming
│  │ жаркое время, под вечер...      │   │     translation
│  │                                 │   │
│  │ At the beginning of July,       │   │  ← Tokens appearing
│  │ during an extremely hot█        │   │     in real-time
│  └─────────────────────────────────┘   │
│                                         │
│  Estimated time: ~45 minutes           │
│                                         │
│  [Pause] [Cancel] [Read Now →]         │
│                                         │
└─────────────────────────────────────────┘
```

---

## Design System

### Colors

```
LIGHT (default)        DARK                 SEPIA
Background: #FAF8F5    Background: #1A1816  Background: #F4ECD8
Text:       #2C2420    Text:       #E8E4DF  Text:       #3D3229
Accent:     #722F37    Accent:     #C4787F  Accent:     #6B4C3A
```

### Typography

```
Reading:  Literata (serif) - 18px default
UI:       Inter (sans-serif)
```

### Reader Layout

**Apple Books-style: Full-width, single column, paginated**

Default view shows translation. Swipe horizontally to flip to original.

```
TRANSLATION (default)              ORIGINAL (swipe to reveal)
┌─────────────────────────┐       ┌─────────────────────────┐
│                         │       │                         │
│   At the beginning of   │ swipe │   В начале июля, в      │
│   July, during an       │ ←───→ │   чрезвычайно жаркое    │
│   extremely hot spell,  │       │   время, под вечер,     │
│   towards evening, a    │       │   один молодой          │
│   young man left his    │       │   человек вышел из      │
│   cramped room, which   │       │   своей каморки...      │
│   he rented from        │       │                         │
│   tenants in S. Lane... │       │                         │
│                         │       │                         │
│                         │       │                         │
│  [Original] [●Trans]    │       │  [●Original] [Trans]    │
│          · 12 ·         │       │          · 12 ·         │
└─────────────────────────┘       └─────────────────────────┘
```

**Interactions:**
- Swipe left/right → flip between translation/original
- Tap edges or swipe up/down → next/previous page
- Tap center → show reading controls
- Toggle buttons → explicit switch (for discoverability)

---

## MVP Milestones

| Week | Focus |
|------|-------|
| 1 | React Native + Expo setup, navigation, theme |
| 2 | Reader UI (side-by-side, page turning, controls) |
| 3 | llama.cpp iOS integration, model download |
| 4 | OCR integration (Vision), PDF processing |
| 5 | Library UI, SQLite storage, progress tracking |
| 6 | Polish, testing on device, TestFlight |

---

## Resolved Decisions

- [x] **Use Expo or bare React Native?** → Expo with New Architecture enabled
- [x] **Best llama.cpp React Native binding?** → `llama.rn` (npm install llama.rn)
- [x] **Where to get GGUF model?** → Extract from Ollama, host on HuggingFace/S3

## Open Questions

- [ ] Handle books longer than 500 pages? (batch processing)
- [ ] App Store approval for 2GB model download?
- [ ] Resume interrupted model downloads?
