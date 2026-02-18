# Plan: Creating Your Cross-Platform App

Based on the reverse engineering of Keeweb and your requirements (Browser + Desktop, Multiplatform, Lightweight, Frontend-only, File-based), here is a step-by-step plan to guide you.

## 1. Core Decisions & Technology Stack

### Recommended Stack (Modern Approach)
Instead of replicating Keeweb's custom Backbone framework (which is older tech), we recommend a modern stack that achieves the same goals with better ecosystem support:

*   **Frontend Framework**: [Vue 3](https://vuejs.org/) or [React](https://react.dev/).
    *   *Why?* Massive component libraries, easier state management, and better developer experience.
*   **Build Tool**: [Vite](https://vitejs.dev/).
    *   *Why?* Extremely fast, supports building for multiple targets out-of-the-box.
*   **Desktop Wrapper**: [Electron](https://www.electronjs.org/).
    *   *Why?* Industry standard, allows usage of Node.js modules (fs) directly in your app logic, making local file access straightforward.
    *   *(Alternative)*: [Tauri](https://tauri.app/) (Rust-based, smaller binaries, more complex if you don't know Rust). Stick to Electron for simplicity if you want pure JS.
*   **Language**: TypeScript.
    *   *Why?* Catch errors early, better tooling support.

## 2. Architecture: "The Adapter Pattern"

The core pattern you must implement is the **Storage Adapter**. Your app logic should never know *where* data is stored, only *that* it can be stored.

### Interface (`IStorage`)
```typescript
interface IStorage {
    name: string; // 'local', 'dropbox', 'gdrive'
    readFile(path: string): Promise<Uint8Array | string>;
    writeFile(path: string, content: Uint8Array | string): Promise<void>;
    listDir(path: string): Promise<string[]>;
}
```

### Implementations

#### 1. Desktop Adapter (`LocalFileStorage`)
*   **Environment**: Electron (Node.js).
*   **Implementation**:
    *   Import `fs` from `node:fs`.
    *   Implements `readFile` using `fs.promises.readFile`.
    *   Implements `writeFile` using `fs.promises.writeFile`.
*   **Benefit**: Full local file system access. User can double-click a file to open your app.

#### 2. Browser Adapter (`BrowserStorage`)
*   **Environment**: Standard Web Browser.
*   **Methods**:
    *   **Modern Web (Chrome/Edge)**: Use the **File System Access API**.
        *   `window.showOpenFilePicker()` gets a handle.
        *   `handle.getFile()` reads.
        *   `handle.createWritable()` writes back to the *same* file on disk (with user permission).
    *   **Legacy Web (Firefox/Safari)**:
        *   `<input type="file">` to read.
        *   Generate a Blob URL and trigger a download to "save". (Cannot overwrite the original file automatically).

#### 3. Cloud Adapters (Optional)
*   **Dropbox/GDrive**: Use their JavaScript SDKs with implicit OAuth flow (client-side only authentication).

## 3. Step-by-Step Implementation Plan

### Phase 1: Setup & Web Version (MVP)
1.  **Initialize Project**:
    ```bash
    npm create vite@latest my-app -- --template react-ts
    ```
2.  **Implement Storage Logic**:
    *   Create the `IStorage` interface.
    *   Implement `BrowserStorage` using File System Access API.
3.  **Build UI**:
    *   Create a simple editor/viewer for your data files.
    *   Connect it to the storage logic.
4.  **Test**: Open the app in Chrome, open a local file, edit, save. Verify the file on disk changed.

### Phase 2: Electron Integration (Desktop)
1.  **Add Electron**:
    *   Install `electron` and `electron-builder`.
    *   Create a `main.js` (Main Process) that creates a browser window and loads your Vite dev server (dev) or built `index.html` (prod).
2.  **Expose Node API**:
    *   Use `contextBridge` in a `preload.js` script to expose `fs` methods securely to the renderer.
    *   *Security Note*: Don't expose `fs` directly; expose specific methods like `saveFile(path, data)`.
3.  **Implement Desktop Storage**:
    *   Create `ElectronStorage` adapter that uses the exposed IPC/preload methods.
    *   Detect environment: `if (window.electron) use ElectronStorage else use BrowserStorage`.

### Phase 3: Packaging & Release
1.  **Configure `electron-builder`**:
    *   Set up build targets for macOS (`dmg`, `zip`), Windows (`nsis`, `portable`), Linux (`AppImage`, `deb`).
2.  **Build**:
    *   Run `electron-builder`.
    *   Test the generated binaries on a VM or real machine.

## 4. Specific "Lightweight" Tips
*   **Minimize Dependencies**: Don't import huge libraries (like full Lodash) if you only need one function.
*   **Lazy Loading**: Use dynamic imports (`import()`) for heavy features that aren't used immediately (e.g., Cloud SDKs).
*   **Virtualization**: If displaying long lists, use windowing (like `react-window`) to keep the DOM light (Keeweb uses `baron` + custom logic for this).

## 5. Storage Data Format
Since you want "Frontend only", your file format should be self-contained.
*   **JSON**: Easiest, human-readable.
*   **Encrypted JSON**: Use `WebCrypto API` (standard in all browsers) to encrypt the JSON blob with a user password before saving. (AES-GCM is the standard choice).
*   **KDBX**: If you want compatibility with Keepass, use `kdbxweb` library (what Keeweb uses).

This plan gives you a modern, maintainable codebase that fulfills all your requirements.
