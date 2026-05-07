# Text Capture Architecture: Terminal Time Travel

## Overview

This document outlines the architecture for adding advanced text capture capabilities to Terminalizer, enabling users to "go back in time" and replay terminal sessions with full editing capability. The proposed solution uses container-based snapshots with diff-based, hashed storage for efficiency and integrity.

## Current State Analysis

### What We Have Now
- **Recording Format**: YAML-based with `delay` and `content` fields
- **Storage**: Sequential records of terminal output
- **Playback**: Time-based replay via the `play` command
- **Format**: Currently captures raw terminal escape sequences and ANSI codes

Example current recording format:
```yaml
config:
  command: bash
  cols: 80
  rows: 24
records:
  - delay: 100
    content: "\u001b[32muser@host\u001b[0m$ "
  - delay: 1500
    content: "ls -la\r\n"
  - delay: 200
    content: "total 48\r\ndrwxr-xr-x  12 user  staff   384 Jan  5 10:00 .\r\n"
```

### What We're Missing
1. **Structured command tracking**: No separation of commands from output
2. **State snapshots**: No ability to restore terminal state at arbitrary points
3. **Editable commands**: Commands are embedded in content strings
4. **Diff-based storage**: Every frame stores full content (inefficient)
5. **Hash verification**: No integrity checking for snapshots

## Proposed Architecture

### 1. Container-Based State Management

#### Concept: Terminal State Container
A "container" represents a complete snapshot of the terminal state at a specific moment:

```javascript
{
  timestamp: 1704451200000,
  hash: "sha256:abc123...",
  state: {
    cursor: { x: 0, y: 5 },
    screen: {
      lines: [...],           // Full line buffer
      scrollback: [...]       // Scrollback buffer
    },
    attributes: {
      foreground: "#afafaf",
      background: "transparent",
      bold: false,
      italic: false
    }
  },
  commands: [
    {
      id: "cmd-001",
      input: "ls -la",
      output: "total 48\ndrwxr-xr-x...",
      exitCode: 0,
      duration: 250
    }
  ],
  environment: {
    cwd: "/home/user/project",
    shell: "bash",
    cols: 80,
    rows: 24
  }
}
```

#### Snapshot Frequency
- **Time-based**: Every 60 seconds (configurable)
- **Event-based**: On significant events (directory change, command completion)
- **Size-based**: When diff accumulation exceeds threshold (e.g., 10KB)

### 2. Diff-Based Delta Recording

Between snapshots, store only changes (deltas):

```javascript
{
  timestamp: 1704451205000,
  parentHash: "sha256:abc123...",
  type: "delta",
  changes: [
    {
      operation: "insert",
      line: 10,
      column: 0,
      content: "$ npm install express\r\n"
    },
    {
      operation: "append",
      line: 11,
      content: "added 50 packages in 5s"
    },
    {
      operation: "cursor",
      x: 0,
      y: 12
    }
  ],
  commands: [
    {
      id: "cmd-002",
      input: "npm install express",
      output: "added 50 packages in 5s",
      exitCode: 0,
      duration: 5200
    }
  ]
}
```

### 3. Hash-Based Integrity

Each snapshot and delta is cryptographically hashed:

```javascript
function computeHash(data) {
  const crypto = require('crypto');
  const normalized = JSON.stringify(data, Object.keys(data).sort());
  return 'sha256:' + crypto.createHash('sha256')
    .update(normalized)
    .digest('hex');
}
```

**Benefits:**
- Verify data integrity on load
- Detect corruption or tampering
- Enable content-addressable storage
- Support incremental snapshots (only store if hash differs)

### 4. Command Structure & Editing

#### Enhanced Recording Format

```yaml
version: 2.0
config:
  command: bash
  cols: 80
  rows: 24
  snapshot_interval: 60000  # ms
  hash_algorithm: sha256

# Full snapshots
snapshots:
  - id: snap-001
    timestamp: 1704451200000
    hash: sha256:abc123...
    state:
      cursor: {x: 0, y: 5}
      screen:
        lines:
          - text: "user@host:~/project$ "
            attributes: [{start: 0, end: 17, fg: "green"}]
      scrollback: []
    environment:
      cwd: /home/user/project
      shell: bash
      user: user
      host: localhost

# Deltas between snapshots
deltas:
  - parent: snap-001
    timestamp: 1704451202000
    hash: sha256:def456...
    changes:
      - op: command_start
        command_id: cmd-001
      - op: insert
        line: 5
        col: 22
        text: "ls -la"
      - op: command_end
        command_id: cmd-001
        exit_code: 0

# Structured command log
commands:
  - id: cmd-001
    timestamp: 1704451202000
    snapshot_id: snap-001
    input: "ls -la"
    output: |
      total 48
      drwxr-xr-x  12 user  staff   384 Jan  5 10:00 .
      drwxr-xr-x   5 user  staff   160 Jan  4 09:30 ..
      -rw-r--r--   1 user  staff  1234 Jan  5 09:45 README.md
    exit_code: 0
    duration: 250
    cwd: /home/user/project
    editable: true
    
  - id: cmd-002
    timestamp: 1704451210000
    snapshot_id: snap-001
    input: "npm install express"
    output: |
      added 50 packages, and audited 51 packages in 5s
      found 0 vulnerabilities
    exit_code: 0
    duration: 5200
    cwd: /home/user/project
    editable: true
```

#### Command Editing Capabilities

Users can edit commands in the YAML file:

**Original:**
```yaml
commands:
  - id: cmd-001
    input: "npm install expres"  # typo
    output: "Package 'expres' not found"
    exit_code: 1
```

**After Edit:**
```yaml
commands:
  - id: cmd-001
    input: "npm install express"  # fixed
    output: "added 50 packages in 5s"  # user can update output too
    exit_code: 0
    edited: true  # flag to indicate manual edit
    original_input: "npm install expres"  # preserve history
```

## Implementation Guide

### Phase 1: Enhanced Recording Format

#### New Recording Structure

Create `commands/record-v2.js` (or enhance existing):

```javascript
const crypto = require('crypto');
const { EventEmitter } = require('events');

class EnhancedRecorder extends EventEmitter {
  constructor(config) {
    super();
    this.config = config;
    this.snapshots = [];
    this.deltas = [];
    this.commands = [];
    this.currentSnapshot = null;
    this.lastSnapshotTime = null;
    this.deltaBuffer = [];
    this.commandParser = new CommandParser();
  }

  // Create full snapshot
  createSnapshot() {
    const state = this.captureTerminalState();
    const snapshot = {
      id: `snap-${Date.now()}`,
      timestamp: Date.now(),
      state: state,
      environment: this.captureEnvironment()
    };
    snapshot.hash = this.computeHash(snapshot);
    this.snapshots.push(snapshot);
    this.currentSnapshot = snapshot;
    this.lastSnapshotTime = Date.now();
    this.deltaBuffer = [];
    return snapshot;
  }

  // Record delta change
  recordDelta(change) {
    this.deltaBuffer.push({
      timestamp: Date.now(),
      parentHash: this.currentSnapshot.hash,
      ...change
    });

    // Check if we need a new snapshot
    if (this.shouldCreateSnapshot()) {
      this.createSnapshot();
    }
  }

  shouldCreateSnapshot() {
    const timeSinceSnapshot = Date.now() - this.lastSnapshotTime;
    const deltaSize = JSON.stringify(this.deltaBuffer).length;
    
    return timeSinceSnapshot > this.config.snapshot_interval ||
           deltaSize > this.config.max_delta_size ||
           this.deltaBuffer.length > this.config.max_delta_count;
  }

  // Parse and store command
  recordCommand(input, output, exitCode, duration) {
    const command = {
      id: `cmd-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
      timestamp: Date.now(),
      snapshot_id: this.currentSnapshot.id,
      input: input,
      output: output,
      exit_code: exitCode,
      duration: duration,
      cwd: process.cwd(),
      editable: true
    };
    this.commands.push(command);
    return command;
  }

  computeHash(data) {
    const normalized = JSON.stringify(data, Object.keys(data).sort());
    return 'sha256:' + crypto.createHash('sha256')
      .update(normalized)
      .digest('hex');
  }

  captureTerminalState() {
    // Capture complete terminal state
    return {
      cursor: { x: this.terminal.cursorX, y: this.terminal.cursorY },
      screen: {
        lines: this.terminal.buffer.lines.map(line => ({
          text: line.translateToString(),
          attributes: line.getAttributes()
        })),
        scrollback: this.terminal.buffer.scrollback.map(line => ({
          text: line.translateToString(),
          attributes: line.getAttributes()
        }))
      },
      attributes: {
        foreground: this.terminal.foreground,
        background: this.terminal.background
      }
    };
  }

  captureEnvironment() {
    return {
      cwd: process.cwd(),
      shell: process.env.SHELL || 'bash',
      user: process.env.USER || process.env.USERNAME,
      host: require('os').hostname(),
      cols: this.terminal.cols,
      rows: this.terminal.rows
    };
  }

  // Export to YAML format
  toYAML() {
    return {
      version: '2.0',
      config: this.config,
      snapshots: this.snapshots,
      deltas: this.deltas,
      commands: this.commands
    };
  }
}
```

#### Command Parser

Detect command boundaries in terminal output:

```javascript
class CommandParser {
  constructor() {
    this.currentCommand = null;
    this.promptPattern = /[\$#>]\s*$/;  // Basic prompt detection
  }

  // Detect when user starts typing a command
  detectCommandStart(content) {
    if (this.promptPattern.test(content)) {
      this.currentCommand = {
        input: '',
        output: '',
        startTime: Date.now()
      };
      return true;
    }
    return false;
  }

  // Accumulate command input
  addInput(content) {
    if (this.currentCommand) {
      this.currentCommand.input += content;
    }
  }

  // Detect command completion (newline after input)
  detectCommandEnd(content) {
    if (this.currentCommand && content.includes('\n')) {
      return true;
    }
    return false;
  }

  // Accumulate command output
  addOutput(content) {
    if (this.currentCommand) {
      this.currentCommand.output += content;
    }
  }

  // Finalize command when next prompt appears
  finalizeCommand() {
    if (this.currentCommand) {
      const cmd = {
        ...this.currentCommand,
        duration: Date.now() - this.currentCommand.startTime
      };
      this.currentCommand = null;
      return cmd;
    }
    return null;
  }
}
```

### Phase 2: Time Travel Playback

#### Enhanced Play Command

```javascript
class TimeTravel {
  constructor(recording) {
    this.recording = recording;
    this.snapshotIndex = new Map(); // hash -> snapshot
    this.timeline = this.buildTimeline();
    this.currentPosition = 0;
  }

  buildTimeline() {
    // Combine snapshots, deltas, and commands into chronological timeline
    const events = [];
    
    this.recording.snapshots.forEach(snap => {
      events.push({ type: 'snapshot', timestamp: snap.timestamp, data: snap });
      this.snapshotIndex.set(snap.hash, snap);
    });
    
    this.recording.deltas.forEach(delta => {
      events.push({ type: 'delta', timestamp: delta.timestamp, data: delta });
    });
    
    this.recording.commands.forEach(cmd => {
      events.push({ type: 'command', timestamp: cmd.timestamp, data: cmd });
    });
    
    return events.sort((a, b) => a.timestamp - b.timestamp);
  }

  // Go to specific timestamp
  goToTime(targetTimestamp) {
    // Find closest snapshot before target
    const snapshot = this.findClosestSnapshot(targetTimestamp);
    
    // Restore snapshot
    this.restoreSnapshot(snapshot);
    
    // Apply deltas up to target time
    this.applyDeltasUntil(targetTimestamp);
    
    this.currentPosition = targetTimestamp;
  }

  findClosestSnapshot(targetTime) {
    let closest = this.recording.snapshots[0];
    for (const snap of this.recording.snapshots) {
      if (snap.timestamp <= targetTime) {
        closest = snap;
      } else {
        break;
      }
    }
    return closest;
  }

  restoreSnapshot(snapshot) {
    // Verify hash integrity
    const computed = this.computeHash(snapshot);
    if (computed !== snapshot.hash) {
      throw new Error(`Snapshot integrity check failed: ${snapshot.id}`);
    }
    
    // Restore terminal state
    this.terminal.reset();
    this.terminal.cursor.x = snapshot.state.cursor.x;
    this.terminal.cursor.y = snapshot.state.cursor.y;
    
    // Restore screen content
    snapshot.state.screen.lines.forEach((line, idx) => {
      this.terminal.buffer.lines[idx] = this.createLine(line);
    });
  }

  applyDeltasUntil(targetTime) {
    const relevantDeltas = this.recording.deltas.filter(
      d => d.timestamp <= targetTime && 
           d.timestamp > this.currentSnapshot.timestamp
    );
    
    relevantDeltas.forEach(delta => {
      this.applyDelta(delta);
    });
  }

  applyDelta(delta) {
    // Verify delta parent hash
    if (delta.parentHash !== this.currentSnapshot.hash) {
      throw new Error('Delta chain broken - hash mismatch');
    }
    
    // Apply each change in the delta
    delta.changes.forEach(change => {
      switch (change.operation) {
        case 'insert':
          this.terminal.insertText(change.line, change.column, change.content);
          break;
        case 'append':
          this.terminal.appendLine(change.line, change.content);
          break;
        case 'cursor':
          this.terminal.cursor.x = change.x;
          this.terminal.cursor.y = change.y;
          break;
      }
    });
  }

  // Navigate through timeline
  stepForward() {
    if (this.currentPosition < this.timeline.length - 1) {
      this.currentPosition++;
      const event = this.timeline[this.currentPosition];
      this.goToTime(event.timestamp);
    }
  }

  stepBackward() {
    if (this.currentPosition > 0) {
      this.currentPosition--;
      const event = this.timeline[this.currentPosition];
      this.goToTime(event.timestamp);
    }
  }

  // Jump to specific command
  goToCommand(commandId) {
    const cmd = this.recording.commands.find(c => c.id === commandId);
    if (cmd) {
      this.goToTime(cmd.timestamp);
    }
  }
}
```

### Phase 3: Command Editing Interface

#### CLI Command for Editing

```javascript
// commands/edit.js
module.exports.command = 'edit <recordingFile>';
module.exports.describe = 'Edit commands in a recording';

module.exports.handler = function(argv) {
  const recording = loadRecording(argv.recordingFile);
  
  console.log('Available commands:');
  recording.commands.forEach((cmd, idx) => {
    console.log(`${idx + 1}. [${cmd.timestamp}] ${cmd.input}`);
    console.log(`   Exit: ${cmd.exit_code}, Duration: ${cmd.duration}ms`);
  });
  
  inquirer.prompt([
    {
      type: 'list',
      name: 'commandIndex',
      message: 'Select command to edit:',
      choices: recording.commands.map((cmd, idx) => ({
        name: `${idx + 1}. ${cmd.input}`,
        value: idx
      }))
    },
    {
      type: 'input',
      name: 'newInput',
      message: 'New command input:',
      default: (answers) => recording.commands[answers.commandIndex].input
    },
    {
      type: 'editor',
      name: 'newOutput',
      message: 'New command output:',
      default: (answers) => recording.commands[answers.commandIndex].output
    },
    {
      type: 'number',
      name: 'newExitCode',
      message: 'Exit code:',
      default: (answers) => recording.commands[answers.commandIndex].exit_code
    }
  ]).then((answers) => {
    const cmd = recording.commands[answers.commandIndex];
    cmd.original_input = cmd.input;
    cmd.input = answers.newInput;
    cmd.output = answers.newOutput;
    cmd.exit_code = answers.newExitCode;
    cmd.edited = true;
    cmd.edit_timestamp = Date.now();
    
    saveRecording(argv.recordingFile, recording);
    console.log('Command updated successfully!');
  });
};
```

#### Batch Editing Support

Allow editing multiple commands via text editor:

```javascript
// commands/edit-batch.js
module.exports.command = 'edit-batch <recordingFile>';
module.exports.describe = 'Edit multiple commands in your text editor';

module.exports.handler = function(argv) {
  const recording = loadRecording(argv.recordingFile);
  const tmpFile = tmp.fileSync({ postfix: '.yml' });
  
  // Export commands to editable YAML
  const editableYaml = yaml.dump({
    commands: recording.commands.map(cmd => ({
      id: cmd.id,
      input: cmd.input,
      output: cmd.output,
      exit_code: cmd.exit_code,
      editable: cmd.editable
    }))
  });
  
  fs.writeFileSync(tmpFile.name, editableYaml);
  
  // Open in user's editor
  const editor = process.env.EDITOR || 'vim';
  const child = spawn(editor, [tmpFile.name], { stdio: 'inherit' });
  
  child.on('exit', () => {
    const edited = yaml.load(fs.readFileSync(tmpFile.name, 'utf8'));
    
    // Merge changes back
    edited.commands.forEach((editedCmd) => {
      const original = recording.commands.find(c => c.id === editedCmd.id);
      if (original) {
        if (original.input !== editedCmd.input) {
          original.original_input = original.input;
          original.input = editedCmd.input;
          original.edited = true;
        }
        original.output = editedCmd.output;
        original.exit_code = editedCmd.exit_code;
      }
    });
    
    saveRecording(argv.recordingFile, recording);
    console.log('Recording updated successfully!');
    tmpFile.removeCallback();
  });
};
```

### Phase 4: Configuration

Add to `config.yml`:

```yaml
# Text capture settings
text_capture:
  enabled: true
  format: v2  # Use enhanced format
  
  # Snapshot settings
  snapshot:
    interval: 60000        # Create snapshot every 60s
    on_command: true       # Snapshot on command completion
    max_delta_size: 10240  # Force snapshot if deltas exceed 10KB
    max_delta_count: 100   # Force snapshot after 100 deltas
  
  # Hash verification
  integrity:
    algorithm: sha256
    verify_on_load: true
    verify_on_playback: true
  
  # Command tracking
  commands:
    enabled: true
    parse_mode: auto       # auto, manual, disabled
    editable: true
    preserve_originals: true
    
  # Storage optimization
  storage:
    compress: true         # gzip compression
    deduplicate: true      # Content-addressable storage
```

## Use Cases

### 1. Record with Text Capture

```bash
terminalizer record demo --text-capture
```

Creates `demo.yml` with enhanced format including snapshots, deltas, and structured commands.

### 2. Time Travel Playback

```bash
# Play from specific timestamp
terminalizer play demo --from 1704451200000

# Play with time navigation
terminalizer play demo --interactive
# Use arrow keys: ← → for frame-by-frame, ↑ ↓ for snapshot-to-snapshot
```

### 3. Edit Commands

```bash
# Interactive editing
terminalizer edit demo

# Batch editing in text editor
terminalizer edit-batch demo

# Direct YAML editing
vim demo.yml  # Edit commands section manually
terminalizer verify demo  # Verify integrity
```

### 4. Snapshot Management

```bash
# List snapshots
terminalizer snapshots demo

# Go to specific snapshot
terminalizer play demo --snapshot snap-001

# Export snapshot
terminalizer export-snapshot demo snap-001 --format json
```

## Benefits of This Architecture

### Efficiency
- **Diff-based storage**: Only store changes, not full frames
- **Snapshot intervals**: Balance between storage and seek time
- **Compression**: YAML + gzip for optimal size

### Integrity
- **Cryptographic hashing**: Detect corruption/tampering
- **Chain verification**: Ensure delta chain integrity
- **Content-addressable**: Deduplicate identical snapshots

### Usability
- **Human-readable**: YAML format is easy to understand
- **Editable**: Commands can be fixed/modified
- **Time travel**: Jump to any point in recording
- **Replay in terminal**: Native terminal playback

### Developer Experience
- **Structured data**: Easy to parse and process
- **Command isolation**: Commands are first-class objects
- **API-friendly**: JSON-compatible for programmatic access
- **Extensible**: Easy to add metadata to commands/snapshots

## Migration Strategy

### Backward Compatibility

Keep existing format (v1) working:

```javascript
class RecordingLoader {
  static load(file) {
    const content = yaml.load(fs.readFileSync(file, 'utf8'));
    
    if (content.version === '2.0') {
      return new EnhancedRecording(content);
    } else {
      return new LegacyRecording(content);
    }
  }
}
```

### Auto-upgrade Option

```bash
terminalizer upgrade demo.yml
```

Converts v1 recording to v2 format (best-effort command parsing).

## Performance Considerations

### Memory Usage
- **Snapshot size**: ~5-50KB per snapshot (depends on terminal size)
- **Delta size**: ~100-500 bytes per delta
- **60-second intervals**: ~60 snapshots per hour = 3-5MB/hour

### Playback Speed
- **Snapshot seeking**: O(log n) with binary search
- **Delta application**: O(k) where k = number of deltas since snapshot
- **Worst case**: ~100 deltas × 1ms = 100ms seek time

### Storage Optimization
- **Compression**: gzip reduces size by 60-80%
- **Deduplication**: Hash-based storage saves repeated snapshots
- **Pruning**: Option to reduce snapshot frequency for long recordings

## Security Considerations

### Hash Verification
- Prevents tampering with recordings
- Detects corrupted data
- Enables trusted replay

### Command Editing Security
- Flag edited commands clearly
- Preserve original values
- Include edit timestamp and reason

### Storage Security
- Option to encrypt recordings
- Password-protected playback
- Secure hash algorithms (SHA-256)

## Next Steps

1. **Prototype**: Build proof-of-concept with basic snapshot/delta
2. **Command parser**: Implement intelligent command boundary detection
3. **Hash verification**: Add integrity checking
4. **Time travel UI**: Create interactive playback interface
5. **Edit interface**: Build command editing tools
6. **Testing**: Extensive testing with real terminal sessions
7. **Documentation**: User guides and API docs
8. **Migration**: Tools to upgrade existing recordings

## Conclusion

This architecture provides a robust foundation for advanced terminal recording with:
- ✅ Text-based capture (not just rendered frames)
- ✅ Time travel capabilities (snapshot + delta)
- ✅ Editable commands (structured command log)
- ✅ Integrity verification (cryptographic hashing)
- ✅ Efficient storage (diff-based with compression)
- ✅ Backward compatibility (legacy format support)

The implementation can be done incrementally, with each phase adding value independently. No code needed yet - this document serves as the architectural blueprint for development.
