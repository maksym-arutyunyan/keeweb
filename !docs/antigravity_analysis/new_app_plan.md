# Plan: Creating Your Cross-Platform App

Based on your requirements (Browser + Desktop, Multiplatform, Lightweight, Frontend-only, File-based, Desktop-Agnostic, Custom Formats), here is the updated plan.

## 1. Core Philosophy: "Web First, Desktop Later"

Since the desktop version is just a wrapper, we will focus 100% on building a powerful, "offline-capable" web application first. The desktop wrappers (Electron/Tauri) will be added as a final deployment step.

## 2. Technology Stack

### Recommended Stack
*   **Frontend Framework**: [Vue 3](https://vuejs.org/) or [React](https://react.dev/).
    *   *Why?* Component-based, excellent ecosystem.
*   **Build Tool**: [Vite](https://vitejs.dev/).
    *   *Why?* Fast, produces static assets suitable for both Web and Desktop wrappers.
*   **Language**: **TypeScript**.
    *   *Why?* You mentioned using custom file formats (Protobuf, CSV). TypeScript is essential for defining strict interfaces for these data structures.
*   **Data Serialization**:
    *   **Protobuf**: `protobufjs` or `ts-proto` for parsing/serializing binary data.
    *   **CSV**: `papaparse` for robust CSV handling.

## 3. Architecture: The "Storage Adapter" Pattern

This is the most critical part of your architecture. To ensure your app works with Electron, Tauri, OR the Browser without code changes, you must abstract the "File System" completely.

### The Interface (`IStorage`)
Your core app interaction should look like this:

```typescript
type FileFilters = { name: string; extensions: string[] }[];

interface IStorageAdapter {
    id: string; // 'browser-fs', 'tauri-fs', 'electron-fs'
    
    // Core capabilities
    readFile(path: string): Promise<Uint8Array>; // Always deal with binary buffers for flexibility
    writeFile(path: string, content: Uint8Array): Promise<void>;
    
    // Dialogs (These will trigger native dialogs in Desktop, HTML dialogs in Browser)
    showOpenDialog(filters?: FileFilters): Promise<string | null>; // Returns path/handle
    showSaveDialog(filters?: FileFilters): Promise<string | null>;
}
```

### Implementations

#### 1. Browser Adapter (The "Standard")
*   **API**: **File System Access API** (Modern Browsers).
*   **Workflow**:
    *   `showOpenDialog` -> calls `window.showOpenFilePicker()`.
    *   Returns a `FileSystemFileHandle` (masked as a string ID in your app).
    *   `writeFile` -> calls `handle.createWritable()`.
*   **Fallback**: For older browsers, use `<input type="file">` and `download` attributes (Mock implementation).

#### 2. Desktop Adapters (Future Proofing)
When you decide to add a desktop wrapper, you simply write a new adapter.
*   **Electron**: Uses `window.electronAPI.readFile` (bridged to Node `fs`).
*   **Tauri**: Uses `window.__TAURI__.fs.readBinaryFile` (bridged to Rust).

**Key Takeaway**: Your app logic *never* imports `fs` or `tauri` directly. It only calls `storageAdapter.readFile()`.

## 4. Simplified Implementation Plan

### Phase 1: The Core Web App
1.  **Initialize Project**: `npm create vite@latest my-app -- --template vue-ts` (or react-ts).
2.  **Define Data Models**:
    *   Create your Typescript interfaces for your data.
    *   Implement parsers for your formats (CSV/Protobuf).
3.  **Implement `BrowserStorageAdapter`**:
    *   Focus on the `File System Access API` to give that "native app" feel in the browser.
4.  **Build the Editor UI**:
    *   Load file -> Parse -> Edit in UI -> Serialize -> Save.
    *   *No backend required.*

### Phase 2: "Desktop Ready" Preparation
1.  **Context isolation**: Ensure no Node.js APIs are used in the main Vue/React code.
2.  **Responsive Design**: Ensure the UI looks good at arbitrary window sizes (desktop windows are resizable).

### Phase 3: The Wrapper (Decision Time)
*   **Option A: Electron**:
    *   If you need deep OS integration or specific Node.js modules.
    *   *Action*: Add `main.js`, create `ElectronStorageAdapter`.
*   **Option B: Tauri**:
    *   If you want a tiny binary (<10MB) and high performance.
    *   *Action*: `cargo tauri init`, create `TauriStorageAdapter`.

## 5. Data Format Strategy
Since you aren't using Keepass, you have total freedom.
*   **CSV**: Good for simple, human-readable lists.
*   **Protobuf**: Excellent for complex, structured data. strict schema, small file size.
*   **Hybrid**: You can zip multiple files together (like `.docx` or `.jar` are just zips) if you need to bundle assets with your data.

This plan minimizes wasted effort. You build the "Product" (the Web App) first, and the "Distribution" (Desktop Wrapper) becomes a trivial configuration detail later.
