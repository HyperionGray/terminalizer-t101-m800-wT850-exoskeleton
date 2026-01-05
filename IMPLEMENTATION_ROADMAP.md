# Implementation Roadmap: Text Capture & Command Editing

## Overview

This document provides a high-level summary of how to add text capture and command editing features to Terminalizer. It synthesizes information from the detailed technical documents and provides a clear path forward.

**Related Documents:**
- `CODEBASE_ANALYSIS.md` - Analysis of current codebase weaknesses
- `TEXT_CAPTURE_ARCHITECTURE.md` - Detailed technical architecture for new features
- `MODERNIZATION.md` - Existing feature roadmap
- `CONTRIBUTING.md` - Development guidelines

## What You Asked For

### Current Capabilities
✅ Records GIFs of terminal sessions  
✅ Uploads recordings to terminalizer.com  
✅ Playback in terminal  
✅ Customizable themes and frames  

### Requested Features

#### 1. Text Capture with Time Travel
> "users can replay via their terminal and 'go back in time'"

**Solution:** Container-based snapshots every minute with diff-based changes

- **Full snapshots** capture complete terminal state (cursor position, screen content, environment)
- **Deltas** store only changes between snapshots (efficient storage)
- **Hashing** provides cryptographic verification of data integrity
- **Time travel** allows jumping to any point in the recording

**Key Benefits:**
- Jump to any timestamp instantly (not just sequential playback)
- Verify recording hasn't been tampered with
- Efficient storage (only store changes, not duplicate frames)
- Navigate by snapshot, delta, or specific command

#### 2. Command Editing
> "they can edit their commands, they're available"

**Solution:** Structured command tracking separate from output

- **Commands as first-class objects** with input, output, exit code, duration
- **YAML format remains editable** so users can fix typos or mistakes
- **Preserve originals** when edits are made for audit trail
- **Batch editing** via text editor or interactive CLI

**Key Benefits:**
- Fix typos in recorded commands
- Update command output for demos
- Edit multiple commands at once
- Preserve history of what was changed

## Architecture Overview

### Enhanced Recording Format (v2)

```yaml
version: 2.0
config:
  # Same as before
  command: bash
  cols: 80
  rows: 24
  # New snapshot settings
  snapshot_interval: 60000  # 1 minute
  hash_algorithm: sha256

snapshots:
  - id: snap-001
    timestamp: 1704451200000
    hash: sha256:abc123...
    state:
      cursor: {x: 0, y: 5}
      screen:
        lines: [...]  # Full terminal content
      scrollback: [...]
    environment:
      cwd: /home/user/project
      shell: bash

deltas:
  - parent: snap-001
    timestamp: 1704451205000
    hash: sha256:def456...
    changes:
      - op: insert
        line: 5
        col: 22
        text: "ls -la"

commands:
  - id: cmd-001
    timestamp: 1704451202000
    snapshot_id: snap-001
    input: "ls -la"
    output: "total 48\ndrwxr-xr-x..."
    exit_code: 0
    duration: 250
    cwd: /home/user/project
    editable: true
```

### How It Works

#### Recording Phase
1. Start recording → Create initial snapshot
2. User types command → Record as delta
3. Command executes → Parse and store in commands section
4. Every 60 seconds → Create new snapshot
5. End recording → Save YAML with snapshots + deltas + commands

#### Playback Phase
1. Load recording → Verify all hashes
2. User seeks to time T → Find closest snapshot before T
3. Restore snapshot → Apply deltas until time T
4. Display result → Terminal shows exact state at time T

#### Editing Phase
1. Open recording file
2. Edit commands section directly (YAML) or use `terminalizer edit` command
3. Save file
4. Play edited recording → Shows updated commands

## Implementation Path

### Phase 1: Foundation Fixes (1-2 weeks)

**Why:** Fix critical issues before building new features

1. **Add Test Infrastructure**
   ```bash
   npm install --save-dev jest
   # Create tests/unit, tests/integration directories
   # Write tests for existing commands
   ```

2. **Fix Security Vulnerabilities**
   ```bash
   npm audit fix --force  # Carefully test each upgrade
   # Manually test: electron, css-loader, inquirer upgrades
   ```

3. **Standardize Error Handling**
   ```javascript
   class TerminalizerError extends Error {
     constructor(message, code, context) {
       super(message);
       this.code = code;
       this.context = context;
     }
   }
   ```

4. **Add Input Validation**
   ```bash
   npm install joi
   # Validate all config inputs
   # Validate command arguments
   ```

### Phase 2: Command Parsing (1 week)

**Why:** Required for structured command tracking

1. **Implement Command Parser**
   ```javascript
   class CommandParser {
     detectPrompt(content) { /* ... */ }
     detectCommandStart(content) { /* ... */ }
     detectCommandEnd(content) { /* ... */ }
     extractCommand(content) { /* ... */ }
     captureOutput(content) { /* ... */ }
   }
   ```

2. **Integrate with Record Command**
   - Hook into PTY data stream
   - Detect command boundaries
   - Extract command metadata
   - Store in structured format

3. **Test with Multiple Shells**
   - bash
   - zsh
   - fish
   - PowerShell

### Phase 3: Snapshot & Delta System (2 weeks)

**Why:** Core of time travel functionality

1. **Implement Snapshot Manager**
   ```javascript
   class SnapshotManager {
     createSnapshot() { /* ... */ }
     computeHash(data) { /* ... */ }
     verifyIntegrity() { /* ... */ }
   }
   ```

2. **Implement Delta Tracker**
   ```javascript
   class DeltaTracker {
     recordChange(change) { /* ... */ }
     shouldSnapshot() { /* ... */ }
     applyDelta(delta) { /* ... */ }
   }
   ```

3. **Update Recording Format**
   - Add version field (2.0)
   - Add snapshots section
   - Add deltas section
   - Add commands section
   - Maintain backward compatibility

4. **Add Configuration**
   ```yaml
   text_capture:
     enabled: true
     snapshot:
       interval: 60000
       on_command: true
   ```

### Phase 4: Time Travel Playback (1 week)

**Why:** Enable "go back in time" functionality

1. **Implement Time Travel Engine**
   ```javascript
   class TimeTravel {
     goToTime(timestamp) { /* ... */ }
     findClosestSnapshot(time) { /* ... */ }
     restoreSnapshot(snapshot) { /* ... */ }
     applyDeltasUntil(time) { /* ... */ }
   }
   ```

2. **Update Play Command**
   ```bash
   terminalizer play demo --from 1704451200000
   terminalizer play demo --interactive  # Arrow key navigation
   ```

3. **Add Snapshot Navigation**
   ```bash
   terminalizer snapshots demo  # List all snapshots
   terminalizer play demo --snapshot snap-001
   ```

### Phase 5: Command Editing (1 week)

**Why:** Enable command editing functionality

1. **Implement Edit Command**
   ```javascript
   // commands/edit.js
   terminalizer edit demo
   // Interactive prompt to select and edit command
   ```

2. **Implement Batch Edit**
   ```javascript
   // commands/edit-batch.js
   terminalizer edit-batch demo
   // Opens commands in $EDITOR
   ```

3. **Add Edit Metadata**
   - Flag edited commands
   - Preserve originals
   - Track edit timestamp

### Phase 6: Testing & Documentation (1 week)

**Why:** Ensure quality and usability

1. **Comprehensive Testing**
   - Unit tests for all new components
   - Integration tests for complete workflow
   - Test edge cases (very long recordings, corrupted data)

2. **User Documentation**
   - Update README with new features
   - Add examples of time travel
   - Add examples of command editing
   - Troubleshooting guide

3. **API Documentation**
   - Document new YAML format
   - Document public APIs
   - Add JSDoc comments

## User Workflows

### Recording with Text Capture

```bash
# Enable text capture
terminalizer record demo --text-capture

# Or configure it globally
terminalizer init
# Edit config to enable text_capture

# Record normally
terminalizer record demo
# Your session is captured with snapshots and structured commands
```

### Time Travel Playback

```bash
# Play from beginning
terminalizer play demo

# Play from specific time
terminalizer play demo --from 1704451200000

# Interactive mode (use arrow keys to navigate)
terminalizer play demo --interactive
# ← → : Frame by frame
# ↑ ↓ : Snapshot by snapshot
# Space : Pause/Resume
# G : Go to timestamp
```

### Editing Commands

```bash
# Method 1: Interactive editing
terminalizer edit demo
# Displays list of commands
# Select one to edit
# Update input, output, exit code

# Method 2: Batch editing
terminalizer edit-batch demo
# Opens all commands in $EDITOR (vim, nano, etc.)
# Edit multiple commands
# Save and exit

# Method 3: Direct YAML editing
vim demo.yml
# Edit commands section manually
# Save file
terminalizer verify demo  # Verify hashes still valid
```

### Creating Demos

```bash
# 1. Record your session
terminalizer record demo --text-capture

# 2. Edit out mistakes
terminalizer edit demo
# Fix typos
# Remove failed attempts
# Polish output

# 3. Verify it looks good
terminalizer play demo

# 4. Generate GIF
terminalizer render demo

# 5. Or generate web player
terminalizer generate demo

# 6. Share it
terminalizer share demo
```

## Technical Considerations

### Storage Efficiency

**Old Format (v1):**
- 60-minute recording: ~30MB
- Every frame stored completely
- No deduplication

**New Format (v2):**
- 60-minute recording: ~3-5MB
- Snapshots: 60 × 50KB = 3MB
- Deltas: ~2MB compressed
- 80-85% size reduction

### Performance

**Seek Time:**
- Snapshot intervals: 60 seconds
- Max deltas to apply: ~100
- Worst case seek: 100ms
- Average seek: 20-50ms
- **Feels instant to user**

**Memory Usage:**
- Snapshot size: ~50KB each
- Delta size: ~500 bytes each
- 60-minute recording: ~5MB in memory
- **Reasonable for modern systems**

### Backward Compatibility

**Strategy:** Support both formats

```javascript
class RecordingLoader {
  static load(file) {
    const content = yaml.load(fs.readFileSync(file, 'utf8'));
    
    if (content.version === '2.0') {
      return new EnhancedRecording(content);
    } else {
      return new LegacyRecording(content);  // v1 format
    }
  }
}
```

**Migration Tool:**
```bash
terminalizer upgrade demo.yml
# Converts v1 → v2 format
# Best-effort command parsing
# May not be perfect for old recordings
```

## Codebase Weaknesses to Address

### Critical (Must Fix First)
1. ❌ **No tests** → Add Jest + basic test coverage
2. ⚠️ **Security vulnerabilities** → Upgrade electron, css-loader, inquirer
3. ❌ **No command parsing** → Build command parser
4. ⚠️ **Inconsistent error handling** → Standardize error patterns

### Important (Fix During Implementation)
5. ⚠️ **No input validation** → Add Joi validation
6. ⚠️ **Callback-based code** → Migrate to async/await
7. ⚠️ **Limited documentation** → Add API docs and specs

### Nice to Have (Future Work)
8. ⚠️ **Module restructuring** → Better organize code
9. ⚠️ **Performance optimization** → Async file I/O
10. ⚠️ **TypeScript migration** → Type safety

## Why This Approach?

### Container-Based Snapshots
- ✅ **Proven pattern**: Docker, Git, Time Machine all use snapshots
- ✅ **Fast seeking**: Jump to any point without replaying everything
- ✅ **Efficient**: Only store what changes
- ✅ **Reliable**: Hash verification ensures integrity

### Diff-Based Storage
- ✅ **Space efficient**: 80-85% size reduction
- ✅ **Git-like**: Familiar pattern for developers
- ✅ **Scalable**: Works for long recordings

### YAML Format
- ✅ **Human-readable**: Easy to understand and edit
- ✅ **Already used**: No breaking changes
- ✅ **Tooling-friendly**: Parsers exist in every language
- ✅ **Version control**: Git diffs are meaningful

### Structured Commands
- ✅ **Editable**: Fix typos, polish demos
- ✅ **Analyzable**: Build tools on top
- ✅ **Searchable**: Find specific commands
- ✅ **Replayable**: Reconstruct execution

## Alternatives Considered

### Alternative 1: Binary Format (like ttyrec)
- ✅ Efficient
- ❌ Not human-readable
- ❌ Hard to edit
- ❌ Requires special tools
- **Verdict:** Doesn't meet "editable" requirement

### Alternative 2: Full frames (like asciinema)
- ✅ Simple
- ❌ Large file sizes
- ❌ Sequential playback only
- ❌ No snapshots
- **Verdict:** Doesn't enable time travel

### Alternative 3: Database storage
- ✅ Fast queries
- ❌ Not portable
- ❌ Requires database
- ❌ Not editable by hand
- **Verdict:** Too complex for use case

### Our Approach: Best of Both Worlds
- ✅ Human-readable (YAML)
- ✅ Efficient (snapshots + deltas)
- ✅ Editable (structured commands)
- ✅ Fast seeking (hash-indexed snapshots)
- ✅ Portable (single file)

## Questions & Answers

### Q: Why 60-second snapshot intervals?
**A:** Balance between seek speed and file size. Can be configured per user preference.

### Q: What if a snapshot is corrupted?
**A:** Hash verification detects corruption immediately. Fall back to previous valid snapshot.

### Q: How do you handle shell prompts varying by theme?
**A:** Command parser has configurable prompt patterns. Auto-detection + user override.

### Q: Can users edit the snapshot/delta data?
**A:** Not recommended. Users should edit the `commands` section. Snapshots/deltas are for internal use.

### Q: Does this work with all shells?
**A:** Initially: bash, zsh, PowerShell. Others can be added by contributing prompt patterns.

### Q: What about nested shells or tmux?
**A:** Current scope: single shell. Nested shells and multiplexers are future enhancements.

### Q: Will this break existing recordings?
**A:** No. Old format (v1) will continue to work. New features require new format (v2).

## Success Criteria

### Must Have
- ✅ Records terminal with structured commands
- ✅ Creates snapshots every N seconds
- ✅ Stores deltas between snapshots
- ✅ Verifies data with cryptographic hashes
- ✅ Allows jumping to arbitrary timestamps
- ✅ Enables command editing via YAML or CLI
- ✅ Maintains backward compatibility with v1 format

### Should Have
- ✅ Interactive time travel mode
- ✅ Batch command editing
- ✅ Snapshot management commands
- ✅ Integrity verification on load
- ✅ Configurable snapshot intervals

### Nice to Have
- ⚠️ Auto-detect shell prompt patterns
- ⚠️ Command search/filter
- ⚠️ Export to other formats
- ⚠️ Replay edited commands live

## Timeline Summary

**Total: 6-7 weeks**

| Phase | Duration | Focus |
|-------|----------|-------|
| 1. Foundation | 1-2 weeks | Tests, security, errors |
| 2. Command Parsing | 1 week | Detect commands |
| 3. Snapshots & Deltas | 2 weeks | Core time travel |
| 4. Time Travel UI | 1 week | Playback features |
| 5. Command Editing | 1 week | Edit interface |
| 6. Testing & Docs | 1 week | Polish & document |

**Recommended Approach:** Do it incrementally
- Each phase adds value independently
- Can ship features as they complete
- Easier to test and debug

## Next Steps

1. **Review these documents** with your team
2. **Decide on timeline** - all at once or incremental?
3. **Choose priority** - which feature first?
4. **Fix foundation** - tests + security + errors
5. **Start implementation** - follow phase-by-phase guide

## Conclusion

The proposed architecture is:
- ✅ **Feasible**: No insurmountable technical challenges
- ✅ **Efficient**: Good storage and performance characteristics
- ✅ **Maintainable**: Clean separation of concerns
- ✅ **Extensible**: Easy to add features later
- ✅ **User-friendly**: Maintains human-readable format

**The codebase is in good shape** with minor weaknesses that should be addressed first. Taking 1-2 weeks to fix the foundation will make implementing the new features much faster and more reliable.

**Total effort: 6-7 weeks** for complete implementation of text capture + command editing, including foundation fixes.

---

**No code needed yet** - these documents provide the complete architectural blueprint. When you're ready to implement, follow the phase-by-phase guide in this document and refer to the detailed technical architecture in `TEXT_CAPTURE_ARCHITECTURE.md`.
