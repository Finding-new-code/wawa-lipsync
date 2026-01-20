---
description: how to run the lipsync-demo example
---

This workflow guides you through setting up and running the `lipsync-demo` example in this monorepo.

### 1. Install Dependencies
Run this command from the root directory to install all package and example dependencies.

// turbo
```powershell
npm install
```

### 2. Fix Windows-Specific Native Binaries
If you are on Windows, you may need to install the specific Rollup native binary to avoid `MODULE_NOT_FOUND` errors.

// turbo
```powershell
npm install @rollup/rollup-win32-x64-msvc --save-dev
```

### 3. Run the Lipsync Demo
Navigate to the example directory and start the Vite development server.

// turbo
```powershell
npm run dev -w lipsync-demo
```

### Troubleshooting
If the server fails to load the configuration due to `@tailwindcss/vite` errors, you might need to temporarily comment out the Tailwind plugin in `examples/lipsync-demo/vite.config.js`.
