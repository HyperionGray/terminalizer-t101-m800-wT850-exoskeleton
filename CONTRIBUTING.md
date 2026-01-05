# Contributing to Terminalizer

Thank you for your interest in contributing to Terminalizer! This document provides guidelines and information for contributors.

## Table of Contents

- [Getting Started](#getting-started)
- [Development Setup](#development-setup)
- [Code Structure](#code-structure)
- [Testing](#testing)
- [Feature Implementation Guides](#feature-implementation-guides)
- [Submitting Changes](#submitting-changes)

## Getting Started

### Prerequisites

- Node.js v20.x or higher
- npm v10.x or higher
- Git

### Installation

```bash
# Clone the repository
git clone https://github.com/HyperionGray/terminalizer-t101-m800-wT850-exoskeleton.git
cd terminalizer-t101-m800-wT850-exoskeleton

# Install dependencies
npm install

# Build the project
npm run build

# Test the CLI
./bin/app.js --help
```

## Development Setup

### Building

```bash
# Development build with watch mode
npm run dev

# Production build
npm run build
```

### Project Structure

```
terminalizer/
├── app.js                 # Main entry point, CLI setup
├── bin/app.js            # Executable wrapper
├── commands/             # CLI command implementations
│   ├── config.js         # Generate config files
│   ├── generate.js       # Web player generation (stub)
│   ├── init.js           # Initialize global config
│   ├── play.js           # Playback recordings
│   ├── record.js         # Record terminal sessions
│   ├── render.js         # Render to GIF/video
│   ├── share.js          # Upload to terminalizer.com
│   └── stream.js         # Live streaming (stub)
├── render/               # Electron-based rendering
│   ├── index.js          # Render process entry
│   ├── index.html        # Render window HTML
│   ├── preload.js        # Electron preload script
│   └── src/              # Render UI source
├── stream-server/        # Live streaming server (planned)
├── config.yml            # Default configuration
├── di.js                 # Dependency injection container
├── utility.js            # Shared utilities
└── webpack.config.js     # Webpack build config
```

## Code Structure

### Dependency Injection

The project uses a simple DI container (`di.js`) to manage dependencies:

```javascript
// In app.js
global.di = new DI();
di.require('chalk');
di.require('fs-extra', 'fs');
di.set('utility', require('./utility.js'));

// In command files
// Dependencies are available via global di object
console.log(di.chalk.green('Success!'));
di.fs.writeFileSync(path, data);
```

### Command Structure

Each command follows a standard pattern:

```javascript
/**
 * The command's main function
 * @param {Object} argv - Command line arguments
 */
function command(argv) {
  // Implementation
}

module.exports = {
  command: 'mycommand <arg>',
  describe: 'Command description',
  handler: command,
  builder: function(yargs) {
    // Define options
    yargs.positional('arg', {
      describe: 'Argument description',
      type: 'string'
    });
  }
};
```

## Testing

Currently, there is no formal test suite. When implementing features:

1. **Manual Testing**: Test all code paths manually
2. **Integration Testing**: Test with actual terminal recordings
3. **Cross-Platform Testing**: Test on Linux, macOS, and Windows if possible

### Creating Test Recordings

```bash
# Create a test recording
./bin/app.js record test-recording

# Type some commands...
echo "Hello World"
ls -la
# Press CTRL+D to exit

# Play it back
./bin/app.js play test-recording

# Render to GIF
./bin/app.js render test-recording
```

## Feature Implementation Guides

### Adding Video Export Formats (MP4/WebM)

See `MODERNIZATION.md` for detailed design. Key steps:

1. Add `fluent-ffmpeg` dependency
2. Create `render/video-encoder.js`:
   ```javascript
   const ffmpeg = require('fluent-ffmpeg');
   
   function encodeToMP4(framesDir, outputFile, framerate) {
     return new Promise((resolve, reject) => {
       ffmpeg()
         .input(framesDir + '/%d.png')
         .inputFPS(framerate)
         .videoCodec('libx264')
         .outputOptions('-pix_fmt yuv420p')
         .output(outputFile)
         .on('end', resolve)
         .on('error', reject)
         .run();
     });
   }
   ```

3. Update `commands/render.js`:
   - Detect format from argv.format
   - Route to appropriate encoder
   - Update file extension handling

4. Test with various recordings

### Implementing Live Streaming

See `MODERNIZATION.md` for full architecture. Key steps:

1. **Install Dependencies**
   ```bash
   npm install ws express helmet bcryptjs rate-limiter-flexible
   ```

2. **Create WebSocket Server** (`stream-server/index.js`):
   ```javascript
   const WebSocket = require('ws');
   const express = require('express');
   const helmet = require('helmet');
   
   class StreamServer {
     constructor(options) {
       this.options = options;
       this.app = express();
       this.wss = null;
     }
     
     start() {
       // Setup Express
       this.app.use(helmet());
       this.app.get('/', (req, res) => {
         res.sendFile(__dirname + '/viewer.html');
       });
       
       // Setup WebSocket
       const server = this.app.listen(this.options.port);
       this.wss = new WebSocket.Server({ server });
       
       this.wss.on('connection', (ws) => {
         this.handleConnection(ws);
       });
     }
     
     handleConnection(ws) {
       // Implement authentication, streaming logic
     }
   }
   ```

3. **Create PTY Streamer** (`stream-server/pty-streamer.js`):
   ```javascript
   const pty = require('@homebridge/node-pty-prebuilt-multiarch');
   
   class PTYStreamer {
     constructor(command, wss) {
       this.ptyProcess = pty.spawn(command, [], {
         cols: 80,
         rows: 24,
       });
       
       this.ptyProcess.on('data', (data) => {
         // Broadcast to all connected clients
         wss.clients.forEach((client) => {
           if (client.readyState === WebSocket.OPEN) {
             client.send(JSON.stringify({
               type: 'output',
               data: data
             }));
           }
         });
       });
     }
   }
   ```

4. **Create Viewer** (`stream-server/viewer.html`):
   ```html
   <!DOCTYPE html>
   <html>
   <head>
     <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/xterm/css/xterm.css" />
     <script src="https://cdn.jsdelivr.net/npm/xterm/lib/xterm.js"></script>
   </head>
   <body>
     <div id="terminal"></div>
     <script>
       const term = new Terminal();
       term.open(document.getElementById('terminal'));
       
       const ws = new WebSocket('ws://localhost:8080');
       ws.onmessage = (event) => {
         const msg = JSON.parse(event.data);
         if (msg.type === 'output') {
           term.write(msg.data);
         }
       };
     </script>
   </body>
   </html>
   ```

5. **Implement TLS/SSL**:
   ```javascript
   const https = require('https');
   const fs = require('fs');
   
   if (options.ssl.enabled) {
     const server = https.createServer({
       cert: fs.readFileSync(options.ssl.cert),
       key: fs.readFileSync(options.ssl.key)
     }, app);
   }
   ```

6. **Add Password Protection**:
   ```javascript
   const bcrypt = require('bcryptjs');
   
   function authenticateConnection(ws, password) {
     return new Promise((resolve, reject) => {
       ws.once('message', async (msg) => {
         const data = JSON.parse(msg);
         if (data.type === 'auth') {
           const valid = await bcrypt.compare(data.password, hashedPassword);
           if (valid) {
             ws.send(JSON.stringify({ type: 'auth', success: true }));
             resolve();
           } else {
             reject(new Error('Invalid password'));
           }
         }
       });
     });
   }
   ```

7. **Update `commands/stream.js`** - Replace stub with actual implementation

### Implementing PNG Sequence Export

Simpler than video export:

1. Create `render/png-sequence-exporter.js`:
   ```javascript
   function exportPNGSequence(framesDir, outputDir) {
     return new Promise((resolve, reject) => {
       // Copy PNG files from framesDir to outputDir
       // with sequential naming: frame-0001.png, frame-0002.png, etc.
       di.fs.copy(framesDir, outputDir)
         .then(resolve)
         .catch(reject);
     });
   }
   ```

2. Update `commands/render.js` to detect `png-sequence` format

3. Skip the mergeFrames step for PNG sequence

### Adding HTML Export

For self-contained HTML with embedded player:

1. Use the existing `terminalizer-player` package
2. Create `render/html-exporter.js`:
   ```javascript
   function exportHTML(recordingFile, outputFile) {
     const template = `
       <!DOCTYPE html>
       <html>
       <head>
         <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/terminalizer-player/dist/css/terminalizer.min.css" />
         <script src="https://cdn.jsdelivr.net/npm/terminalizer-player/dist/js/terminalizer.js"></script>
       </head>
       <body>
         <div id="player"></div>
         <script>
           const data = ${JSON.stringify(recordingFile.json)};
           Terminalizer.render(data, { selector: '#player' });
         </script>
       </body>
       </html>
     `;
     
     return di.fs.writeFile(outputFile, template, 'utf8');
   }
   ```

## Submitting Changes

### Pull Request Process

1. **Fork and Branch**
   ```bash
   git checkout -b feature/my-new-feature
   ```

2. **Make Changes**
   - Follow existing code style
   - Add comments for complex logic
   - Test thoroughly

3. **Commit**
   ```bash
   git add .
   git commit -m "feat: add MP4 export support"
   ```

   Follow [Conventional Commits](https://www.conventionalcommits.org/):
   - `feat:` - New features
   - `fix:` - Bug fixes
   - `docs:` - Documentation changes
   - `refactor:` - Code refactoring
   - `test:` - Adding tests
   - `chore:` - Maintenance tasks

4. **Push and PR**
   ```bash
   git push origin feature/my-new-feature
   ```
   Then create a Pull Request on GitHub

### Code Review

- Address all review comments
- Keep PRs focused and reasonably sized
- Update documentation as needed

## Coding Standards

### Style

- Use 2 spaces for indentation
- Use single quotes for strings
- Add semicolons
- Keep lines under 100 characters when possible

### Comments

- Add JSDoc comments for functions:
  ```javascript
  /**
   * Description of function
   * 
   * @param {Type} paramName - Description
   * @return {Type} - Description
   */
  ```

- Add inline comments for complex logic
- Keep comments up-to-date with code changes

### Error Handling

- Always handle promise rejections
- Provide helpful error messages
- Use `di.errorHandler` for fatal errors

## Security

- Never commit secrets or credentials
- Validate all user input
- Use parameterized queries/commands
- Keep dependencies updated
- Report security issues privately to maintainers

## Getting Help

- Check `MODERNIZATION.md` for architecture details
- Review existing commands for patterns
- Open an issue for questions
- Join community discussions

## License

By contributing, you agree that your contributions will be licensed under the MIT License.
