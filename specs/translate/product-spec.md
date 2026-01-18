# WALT - Product Spec

**Web App Literature Translator**

An iOS app for reading foreign literature and historical documents. Translate entire books offline, privately, on your phone. Original text alongside translation in a native reading experience.

---

## Vision

"Translate books offline. Read them beautifully."

No cloud. No subscriptions. No internet required. Your books stay on your device.

---

## Target Users

| User | Use Case |
|------|----------|
| Literature students | Studying Dostoevsky, Balzac, Goethe in original |
| History researchers | 19th-20th century foreign sources |
| Genealogists | Old family letters, documents, records |
| Curious readers | Want to read classics beyond translation |
| Privacy-conscious | Don't want documents uploaded to cloud |
| Travelers | Need translation without internet |

---

## Platform

**iOS only** (React Native)

---

## Core Features

### MVP (v1.0)

1. **Document Import**
   - PDF support (digital PDFs with selectable text)
   - Multi-page PDF processing
   - File picker import

2. **Local Translation**
   - TranslateGemma 4B running on-device
   - Works completely offline
   - No data leaves the phone
   - 55 language support

3. **Book Reader (Apple Books-style)**
   - Full-width translation (default view)
   - Swipe to flip between translation ↔ original
   - Toggle buttons for explicit switching
   - Page-by-page navigation
   - Clean, distraction-free reading

4. **Reading Experience**
   - Font size controls (14-28px)
   - Themes: Light, Dark, Sepia
   - Progress tracking
   - Distraction-free design

5. **Library**
   - All books stored locally
   - Progress saved per book
   - Organize by recent / title

6. **Translation Queue**
   - Background processing (app doesn't need to be open)
   - Queue view shows all pending/in-progress books
   - Per-book progress: "Page 42 of 300"
   - Per-page progress: live text streaming or progress bar
   - Can start reading partially-translated books
   - Pause/resume/cancel translations

7. **Language Support**
   - 55 languages via TranslateGemma
   - Russian, French, German, Spanish, Italian, Chinese, Japanese, Arabic, etc.

### Future (v1.1+)

- Highlights & annotations
- Vocabulary tracking
- Export notes

---

## User Flows

### First Launch

```
Download App → Prompt to download translation model (~2GB)
            → Download completes
            → Ready to use

No sign up. No account. Just start.
```

### Import & Read

```
Tap "+" → Select PDF → Choose source language
        → Extract text + translate (background)
        → Book added to library
        → Open and read (even while translating)
```

### Reading

```
Open book → Full-width translation → Swipe left/right to see original
                                   → Swipe up/down to turn pages
                                   → Tap toggle buttons to switch views
                                   → Tap center for controls
```

---

## Key Differentiators

| Other Apps | WALT |
|------------|------|
| Require internet | Works offline |
| Upload to cloud | 100% local/private |
| Require account | No sign up needed |
| Subscription pricing | One-time purchase (or free) |
| Generic translation | Optimized for books/literature |
| Output a file | Native reading experience |

---

## Storage

**Everything stored locally on device:**
- Books (PDFs)
- Translations
- Reading progress
- Preferences

**No backend. No account. No cloud.**

iOS automatically backs up app data to iCloud (if user has iCloud backup enabled). We don't build anything for this - Apple handles it.

---

## Device Requirements

- iPhone 14 or newer (Apple7 GPU required for Metal/llama.rn)
- ~3GB storage for model
- iOS 17+

Older devices: Not supported (Metal GPU requirement).

---

## Success Metrics

1. Can we translate a 300-page book locally in reasonable time?
2. Is translation quality good for 19th century literature?
3. Is the reading experience comfortable for long sessions?
4. Do users value the offline/privacy angle?
