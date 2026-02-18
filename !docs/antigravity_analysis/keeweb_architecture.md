# Keeweb Architecture Analysis

## 1. Overview
Keeweb is a "zero-knowledge" password manager that runs as a static web application and a desktop application. It achieves this by decoupling the **UI/Logic** from the **System Integration**.

## 2. Technology Stack
*   **Frontend Framework**: Custom implementation inspired by **Backbone.js** (MVC pattern).
    *   **Views**: Manage DOM (using `morphdom` for efficient updates) and events.
    *   **Models/Collections**: Manage data state.
    *   **Templates**: `handlebars`.
*   **Build System**: `Grunt` orchestrates tasks, `Webpack` bundles the JavaScript/CSS.
*   **Desktop Engine**: **Electron**.
    *   **Main Process**: Handles window management, system constraints, and file I/O permissions.
    *   **Renderer Process**: Runs the exact same web app as the browser version, with `nodeIntegration` enabled (or exposed via preload context) to allow access to `fs` (File System).

## 3. Project Structure
*   `app/`: Core application logic (Shared between Web and Desktop).
    *   `scripts/`: logic (Models, Views, Controllers).
    *   `templates/`: Handlebars templates.
    *   `styles/`: SCSS/CSS.
*   `desktop/`: Electron-specific code.
    *   `main.js`: Electron entry point.
*   `build/`: Configuration for Webpack and build tasks.

## 4. Key Architectural Patterns

### 4.1. Dual-Target Build
The app is built twice:
1.  **Web Target**: Output to `dist/index.html`. Has no access to Node.js APIs.
2.  **Desktop Target**: Output to `dist/desktop/`. Has access to Node.js APIs (via `Launcher`).

### 4.2. Storage Abstraction ("No Backend")
This is the critical component for your requirements. Keeweb does not have a backend API. Instead, it uses a **Storage Adapter Pattern**:

*   **Interface**: `StorageBase` defines `load`, `save`, `stat`, `mkdir`, `watch`.
*   **Implementations**:
    *   `StorageFile`: Handles local files.
        *   **In Desktop**: Uses `Launcher` object which bridges to Electron's `fs` (Node.js) to read/write files directly on the user's disk.
        *   **In Browser**: This adapter is **disabled** because standard browsers cannot arbitrarily read/write local files.
    *   **Cloud Providers** (`StorageDropbox`, `StorageGDrive`, etc.):
        *   Use implicit OAuth flows (Client-side usage) to authenticate.
        *   Use the provider's REST API to read/write files directly from the browser.
    *   **WebDAV**: Connects to a user-provided WebDAV server (Nextcloud, etc.).

### 4.3. The "Launcher" Bridge
To share code between Web and Desktop, Keeweb uses a `Launcher` module:
*   **In Desktop**: `launcher-electron.js` requires Node.js modules (`fs`, `crypto`, `path`) and exports methods like `readFile`, `writeFile`.
*   **In Web**: The `Launcher` is either null or a stub, disabling features that require native access.

## 5. Summary for Replication
To replicate this architecture for your requirements:
1.  **Build a Static Web App**: Use a modern framework (Vue/React).
2.  **Abstract Storage**: Create a strict interface for reading/writing data. This is crucial for swapping between file systems (Node.js `fs` vs Tauri `fs` vs Browser `File System Access API`).
3.  **Implement Adapters**:
    *   **Cloud**: Use their JS SDKs.
    *   **Local (Web - Modern)**: Use the **File System Access API**.
    *   **Local (Desktop)**: Implement this LAST. You can write a specific adapter later for either **Electron** (Node.js) or **Tauri** (Rust IPC) without changing your core app logic.

