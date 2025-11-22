# AGENTS.md

This file provides guidance to AI coding agents when working with code in this repository.

## Project Overview

Mac Contact Sync Plugin for Obsidian - synchronizes contacts from macOS Contacts app to Obsidian notes using AppleScript/JXA (JavaScript for Automation).

**Platform**: macOS only (uses native macOS Contacts app and osascript)

## Development Commands

```bash
# Install dependencies
npm install

# Development mode (watch mode with esbuild)
npm run dev

# Production build
npm run build

# Run tests
npm test

# Type check without emitting files
tsc -noEmit -skipLibCheck
```

## Architecture

### Core Flow

1. **main.ts** - Plugin entry point, loads settings and initializes CommandHandler
2. **CommandHandler** - Registers the SyncContacts command
3. **SyncContacts** - Orchestrates the sync process:
   - Gets contact count from macOS Contacts app
   - Streams vCards from the specified Smart List
   - Converts each vCard to ContactModel
   - Generates/updates Obsidian notes using template placeholders
4. **ContactsService** - Executes JXA scripts via OsaScriptService to interact with macOS Contacts app
5. **OsaScriptService** - Spawns osascript processes to run JXA scripts

### Key Components

**Services** (src/services/):
- `ContactsService` - Fetches contacts from macOS using JXA scripts, returns stream of ContactModels
- `OsaScriptService` - Wrapper for executing JXA scripts via `osascript` subprocess

**Models** (src/models/):
- `ContactModel` - Data model for contacts, parses vCard format using regex
  - Static method `fromVCardString(vCardStr)` converts vCard to ContactModel

**Commands** (src/commands/):
- `SyncContacts` - Main sync command implementation
  - Uses streaming approach for handling large contact lists
  - Supports partial note updates using `%%==MACOS_CONTACT_SYNC_KEEP==%%` delimiter

**Settings** (src/Settings.ts):
- `ContactsGroup` - Name of Smart List in macOS Contacts app (default: 'Obsidian')
- `ContactDirectory` - Folder for contact notes (default: 'Contacts')
- `FilenameTemplateContact` - Template for filenames (default: '{{Name}}')
- `ContactTemplatePath` - Path to custom template file

### Template System

Templates use `{{placeholder}}` syntax. Available placeholders defined in `ContactModel.fields`:
- Name, FirstName, MiddleName, LastName, Nickname, MaidenName
- Title, JobTitle, Department, Organization
- PhoneticFirstName, PhoneticMiddleName, PhoneticLastName
- BirthDate, Note, HomePage

Arrays (Emails, PhoneNumbers, Addresses, URLs) are rendered as `[item1, item2, ...]`.

**Content Preservation**: Content after `%%==MACOS_CONTACT_SYNC_KEEP==%%` delimiter is preserved across syncs.

### JXA Script Execution

All macOS Contacts interaction uses JXA (JavaScript for Automation):
- Scripts are executed via `spawn('osascript', ['-l', 'JavaScript', '-e', script])`
- Two modes: streaming (for vCards) and promise-based (for contact count)
- Scripts access `Application('Contacts')` and filter by group name
- vCard photo data is stripped using regex to reduce size

## Testing

- **Framework**: Jest with ts-jest
- **Test files**: `tests/**/*.test.ts`
- **Module mapping**: `src/*` aliased via jest.config.js and tsconfig.json
- Run single test: `npm test -- <test-file-name>`

## Build System

- **Bundler**: esbuild (configured in esbuild.config.mjs)
- **Entry point**: src/main.ts
- **Output**: main.js (bundled for Obsidian plugin)
- **Development**: Watch mode rebuilds on file changes
- **Production**: Single build with tree-shaking, no sourcemap
- **Externals**: Obsidian API, Electron, CodeMirror packages marked external

## Important Patterns

- **vCard parsing**: Uses regex-based parsing (see `ContactModel.fromVCardString`) - be careful with multiline matching
- **Stream processing**: Contact sync uses Node.js streams to handle large contact lists efficiently
- **Error handling**: JXA script errors are caught and logged, user sees Notice notifications
- **Path normalization**: Always use `normalizePath()` from Obsidian API for file paths
