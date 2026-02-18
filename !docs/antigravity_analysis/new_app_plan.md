# Plan: Creating Your Cross-Platform App (Svelte Edition)

Based on your requirements (Browser + Desktop, Multiplatform, Lightweight, Frontend-only, File-based, Desktop-Agnostic, Custom Formats) and the decision to use **Svelte**, here is the concrete implementation plan.

## 1. Core Philosophy: "Web First, Desktop Later"

We will build a high-performance, offline-capable Svelte web application. The desktop wrappers (Electron/Tauri) will be added as a thin distribution layer later.

## 2. Technology Stack

### Selected Stack
*   **Framework**: **Svelte 5** (or latest stable).
    *   *Why?* Compiles to tiny vanilla JS, exceptional performance, and simple state management (Stores/Runes) perfect for handling file data.
*   **Build Tool**: **Vite**.
    *   *Why?* Instant dev server, optimized builds for both web and desktop targets.
*   **Language**: **TypeScript**.
    *   *Why?* Essential for defining strict schemas for your custom file formats (Protobuf/CSV).
*   **CSS**: **Vanilla CSS** or **Tailwind** (User preference, Svelte handles scoped CSS natively).
*   **Data Serialization**:
    *   **Protobuf**: `protobufjs` (for binary speed/size) or `ts-proto`.
    *   **CSV**: `papaparse` (standard for JS CSV handling).

## 3. Architecture: The "Storage Adapter" Pattern

To keep the app "Desktop Agnostic", we abstract the file system. Svelte Stores will bridge the gap between this adapter and your UI.

### The Interface (`IStorage`)
```typescript
// src/lib/storage/types.ts

export interface IStorageAdapter {
    id: 'browser' | 'electron' | 'tauri';
    
    // Core capabilities
    readFile(path: string): Promise<Uint8Array>; 
    writeFile(path: string, content: Uint8Array): Promise<void>;
    
    // Dialogs return a "Path" (Desktop) or a "Handle ID" (Browser)
    showOpenDialog(options?: { extensions: string[] }): Promise<string | null>;
    showSaveDialog(options?: { extensions: string[] }): Promise<string | null>;
}
```

### Svelte Integration
You will likely have a global store (or Svelte 5 Rune) that holds the current adapter.

```typescript
// src/lib/stores/appState.ts
import { writable } from 'svelte/store';
import { BrowserAdapter } from '$lib/storage/browser';

// Default to browser, swap at runtime if window.electron exists
export const storage = writable<IStorageAdapter>(new BrowserAdapter());
```

## 4. Implementation Steps

### Phase 1: The Core Web App
1.  **Initialize Project**:
    ```bash
    npm create vite@latest my-app -- --template svelte-ts
    cd my-app
    npm install
    ```
2.  **Implement `BrowserStorageAdapter`**:
    *   Use the **File System Access API** (`window.showOpenFilePicker`).
    *   Store file handles in IndexedDB (idb-keyval) so you can re-open files on page reload without asking permission again (if the browser permits).
3.  **Build the Editor UI**:
    *   Create Svelte components for your data editors.
    *   Use `bind:value` for two-way binding with your data models.
    *   **Protobuf Workflow**: Load Uint8Array -> Parse to Object -> generic Svelte Form -> Serialize to Uint8Array -> Save.

### Phase 2: "Desktop Ready" Prep
1.  **Context Isolation**: Ensure you don't use any Node.js APIs in your `.svelte` files. All logic must be pure JS/TS.
2.  **Hotkeys**: Implement a global keyboard handler (Ctrl+S, Ctrl+O) that calls the `storage` store methods.

### Phase 3: The Wrapper (Desktop)
Since you know Rust and want agility, **Tauri** is likely the best fit for Svelte.

*   **Initialize Tauri**:
    ```bash
    npm install @tauri-apps/cli
    npx tauri init
    ```
*   **Implement `TauriStorageAdapter`**:
    *   Use `@tauri-apps/api/fs` to read/write binary files.
    *   Use `@tauri-apps/api/dialog` for native open/save dialogs.
*   **Switching Logic**:
    ```typescript
    // src/main.ts
    if (window.__TAURI__) {
       storage.set(new TauriAdapter());
    }
    ```

## 5. Why Svelte wins here
*   **Binary Handling**: Svelte doesn't have the overhead of React's synthetic events or heavy VDOM diffing, which is great when hex-editing or handling large CSV grids.
*   **Stores**: Svelte's strict separation of "Stores" (logic/data) and "Components" (UI) encourages the exact architecture you need: a headless "App Core" that talks to Storage Adapters, with a thin UI layer on top.

This plan gives you the lightest, fastest possible app that runs everywhere.
