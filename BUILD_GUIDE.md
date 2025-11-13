# Complete Build Guide for Obsidian Vimrc Support

## Build Process Overview

The build process uses **Rollup** to compile TypeScript into JavaScript.

### What Happens During Build:

1. **Rollup reads**: `main.ts` (TypeScript source - 36KB)
2. **Processes through**:
   - TypeScript compiler (converts .ts → .js)
   - Node resolve plugin (handles imports)
   - CommonJS plugin (module format)
3. **Outputs**: `main.js` (JavaScript bundle - 195KB)

### Step-by-Step Commands:

```bash
# 1. Navigate to the project directory
cd /path/to/obsidian-vimrc-support

# 2. Install dependencies (first time only)
npm install

# 3. Clean old build (optional but recommended)
rm -f main.js

# 4. Build the plugin
npm run build

# 5. Verify the output
ls -lh main.js manifest.json
file main.js
```

### Expected Output:

```
> obsidian-vimrc-support@0.10.2 build
> rollup --config rollup.config.js

main.ts → .
created . in 2.2s

-rw-r--r-- 1 user user 195K Nov 13 15:22 main.js
-rw-r--r-- 1 user user  263 Nov 12 05:29 manifest.json

main.js: JavaScript source, Unicode text, UTF-8 text
```

### Troubleshooting:

If you see only `main.ts` and NO `main.js`:

1. **Check npm output** - Did it say "created . in X.Xs"?
2. **Check for errors** - Look for red error messages
3. **Verify node_modules exists**:
   ```bash
   ls -la node_modules/rollup
   ```
4. **Try cleaning and rebuilding**:
   ```bash
   rm -rf node_modules package-lock.json
   npm install
   npm run build
   ```

### Installation After Build:

Once `main.js` is created, install by copying 2 files:

```bash
# Copy these files:
main.js          (195KB JavaScript file)
manifest.json    (263 bytes JSON file)

# To this location:
YourVault/.obsidian/plugins/obsidian-vimrc-support/
```

### Directory Structure After Installation:

```
YourVault/
  .obsidian/
    plugins/
      obsidian-vimrc-support/
        main.js          ← The compiled JavaScript
        manifest.json    ← Plugin metadata
```

### Common Mistakes:

❌ **Wrong**: Copying `main.ts` (source TypeScript)
✅ **Right**: Copying `main.js` (compiled JavaScript)

❌ **Wrong**: Running `npm build` (doesn't exist)
✅ **Right**: Running `npm run build`

❌ **Wrong**: Plugin folder doesn't exist
✅ **Right**: Create folder first, then copy files
