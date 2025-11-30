# Setup Guide: mcp-abap-abap-adt-api (MCP Server)

**For:** Future Claude Code instances
**Version:** 0.1.1 (Experimental)
**Maturity:** 72/100 (Experimental but comprehensive)
**ADT Coverage:** 93% (inherited from abap-adt-api)

---

## Overview

This guide provides step-by-step instructions to compile, install, and test **mcp-abap-abap-adt-api** - an MCP (Model Context Protocol) wrapper around the abap-adt-api library. This server exposes comprehensive ABAP development tools for AI assistants like Claude Desktop and Cline.

⚠️ **IMPORTANT**: This server is in **experimental status**. Use with caution in production environments.

---

## Prerequisites

### Required Software

1. **Node.js** (v18+ recommended)
   ```bash
   node --version  # Should be >= 18.0.0
   npm --version   # Should be >= 9.0.0
   ```

2. **Git**
   ```bash
   git --version
   ```

3. **TypeScript** (installed as dev dependency, but useful globally)
   ```bash
   npm install -g typescript
   ```

### Required Access

1. **SAP System with ADT enabled**
   - URL to SAP system (e.g., `https://your-sap-server.com:44300`)
   - Valid SAP credentials (username, password)
   - SAP client number
   - Language code (e.g., EN, DE)

2. **Network Access**
   - Ability to reach SAP system from development machine
   - Firewall/proxy configuration if needed

---

## Installation Steps

### Step 1: Clone the Repository

```bash
# Navigate to your workspace
cd ~/workspace

# Clone the repository
git clone https://github.com/mario-andreschak/mcp-abap-abap-adt-api.git

# OR if using oisee fork:
git clone https://github.com/oisee/mcp-abap-abap-adt-api.git

# Enter the directory
cd mcp-abap-abap-adt-api
```

**Expected result:**
```
Cloning into 'mcp-abap-abap-adt-api'...
remote: Enumerating objects: ...
remote: Total ... (delta ...), reused ... (delta ...)
```

### Step 2: Install Dependencies

```bash
# Install all npm dependencies
npm install
```

**Expected result:**
```
added 123 packages, and audited 124 packages in 15s

found 0 vulnerabilities
```

**Dependencies installed:**
- `@modelcontextprotocol/sdk` ^1.4.1 - MCP protocol SDK
- `abap-adt-api` ^6.2.0 - Core ADT client library
- `dotenv` ^16.4.7 - Environment configuration
- `typescript` ^5.7.3 - TypeScript compiler
- `@types/node` ^22.10.10 - Node.js type definitions

**Dev dependencies:**
- `jest` ^29.7.0 - Testing framework
- `ts-jest` ^29.2.5 - TypeScript testing support
- `@types/jest` ^29.5.14 - Jest type definitions

### Step 3: Configure Environment Variables

```bash
# Copy the example environment file
cp .env.example .env

# Edit the .env file with your SAP credentials
nano .env  # or use your preferred editor
```

**Required `.env` configuration:**

```env
# SAP System Connection (REQUIRED)
SAP_URL=https://your-sap-server.com:44300
SAP_USER=YOUR_USERNAME
SAP_PASSWORD=YOUR_PASSWORD

# SAP System Details (RECOMMENDED)
SAP_CLIENT=100
SAP_LANGUAGE=EN

# SSL Configuration (if using self-signed certificates)
NODE_TLS_REJECT_UNAUTHORIZED="0"
```

**Important notes:**
- ⚠️ **NEVER commit** `.env` to version control
- `.env` is in `.gitignore` by default
- Use strong passwords
- Consider using environment-specific configs for dev/test/prod

**Validation:**
```bash
# Verify .env file exists and has content
cat .env | grep SAP_URL
# Should output: SAP_URL=https://...
```

### Step 4: Build the Project

```bash
# Compile TypeScript to JavaScript
npm run build
```

**Expected result:**
```
> mcp-abap-abap-adt-api@0.1.1 build
> tsc -p tsconfig.json

# TypeScript compilation completes without errors
```

**What happens:**
- TypeScript compiler (`tsc`) reads `tsconfig.json`
- Compiles all `.ts` files from `src/` directory
- Outputs compiled `.js` files to `dist/` directory
- Generates source maps for debugging

**Verify build:**
```bash
# Check that dist directory was created
ls -la dist/

# Should see:
# dist/
#   ├── index.js
#   ├── handlers/
#   │   ├── AuthHandlers.js
#   │   ├── ObjectHandlers.js
#   │   └── ... (26 more handler files)
#   ├── lib/
#   │   └── logger.js
#   └── types/
#       └── tools.js
```

**Troubleshooting:**
```bash
# If build fails with type errors, check Node.js version
node --version

# If build fails with missing modules
npm install

# Clean build (if needed)
rm -rf dist/
npm run build
```

---

## Running the Server

### Development Mode

```bash
# Run with MCP Inspector for debugging
npm run dev
```

**What happens:**
- Starts MCP Inspector
- Opens debugging interface (usually http://localhost:3000)
- Provides interactive tool testing
- Shows request/response logs

**Expected output:**
```
MCP Inspector started at http://localhost:3000
Server running on stdio transport
Waiting for client connection...
```

### Production Mode

```bash
# Run the compiled server
npm run start
```

**What happens:**
- Executes `node ./dist/index.js`
- Server listens on stdio (standard input/output)
- Ready to receive MCP protocol messages
- Logs to stderr (preserves stdio for protocol)

**Expected output:**
```
MCP Server started
Loaded 28 handler categories
Ready to accept connections
```

---

## Testing the Server

### Quick Connection Test

Create a simple test script `test-connection.js`:

```javascript
const { MCPClient } = require('@modelcontextprotocol/sdk/client/index.js');
const { StdioClientTransport } = require('@modelcontextprotocol/sdk/client/stdio.js');

async function testConnection() {
  const transport = new StdioClientTransport({
    command: 'node',
    args: ['./dist/index.js']
  });

  const client = new MCPClient({
    name: 'test-client',
    version: '1.0.0'
  }, {
    capabilities: {}
  });

  await client.connect(transport);

  // List available tools
  const tools = await client.listTools();
  console.log('Available tools:', tools.tools.length);
  console.log('First 5 tools:', tools.tools.slice(0, 5).map(t => t.name));

  await client.close();
}

testConnection().catch(console.error);
```

**Run test:**
```bash
node test-connection.js
```

**Expected output:**
```
Available tools: 80+
First 5 tools: [ 'login', 'logout', 'searchObject', 'getObject', 'setObjectSource' ]
```

### Testing Individual Tools

#### Test 1: Login

```bash
# Using MCP Inspector (npm run dev)
# Navigate to http://localhost:3000
# Select tool: "login"
# No parameters needed (uses .env credentials)
# Click "Execute"
```

**Expected response:**
```json
{
  "content": [
    {
      "type": "text",
      "text": "Login successful. Session created."
    }
  ]
}
```

#### Test 2: Search for an Object

```bash
# Tool: searchObject
# Parameters:
{
  "query": "CL_ABAP_CHAR_UTILITIES"
}
```

**Expected response:**
```json
{
  "content": [
    {
      "type": "text",
      "text": "Object URI: /sap/bc/adt/oo/classes/cl_abap_char_utilities"
    }
  ]
}
```

#### Test 3: Get Object Source

```bash
# Tool: getObjectSource
# Parameters:
{
  "objSourceUrl": "/sap/bc/adt/oo/classes/cl_abap_char_utilities/source/main"
}
```

**Expected response:**
```json
{
  "content": [
    {
      "type": "text",
      "text": "CLASS cl_abap_char_utilities DEFINITION\n  PUBLIC\n  FINAL\n  CREATE PUBLIC .\n\n..."
    }
  ]
}
```

### Running Unit Tests

```bash
# Run all tests
npm test

# Run tests in watch mode
npm run test:watch

# Run tests with coverage
npm run test:coverage
```

**Expected output:**
```
PASS  src/__tests__/handlers.test.ts
  ✓ AuthHandlers should login (50ms)
  ✓ ObjectHandlers should get object (35ms)
  ✓ TransportHandlers should create transport (42ms)

Test Suites: 3 passed, 3 total
Tests:       28 passed, 28 total
Snapshots:   0 total
Time:        5.123s
```

**Note:** Current version (0.1.1) has minimal test coverage. Consider this when using in production.

---

## Integration with AI Assistants

### Claude Desktop Integration

1. **Locate Claude Desktop config:**
   - **macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`
   - **Windows**: `%APPDATA%\Claude\claude_desktop_config.json`
   - **Linux**: `~/.config/Claude/claude_desktop_config.json`

2. **Add MCP server configuration:**

```json
{
  "mcpServers": {
    "mcp-abap-abap-adt-api": {
      "command": "node",
      "args": [
        "/absolute/path/to/mcp-abap-abap-adt-api/dist/index.js"
      ],
      "env": {
        "SAP_URL": "https://your-sap-server.com:44300",
        "SAP_USER": "YOUR_USERNAME",
        "SAP_PASSWORD": "YOUR_PASSWORD",
        "SAP_CLIENT": "100",
        "SAP_LANGUAGE": "EN"
      }
    }
  }
}
```

**Alternative: Use .env file path:**
```json
{
  "mcpServers": {
    "mcp-abap-abap-adt-api": {
      "command": "node",
      "args": [
        "/absolute/path/to/mcp-abap-abap-adt-api/dist/index.js"
      ],
      "cwd": "/absolute/path/to/mcp-abap-abap-adt-api"
    }
  }
}
```

3. **Restart Claude Desktop**

4. **Verify connection:**
   - Open Claude Desktop
   - Look for 🔌 icon indicating MCP server connected
   - Try asking: "List available ABAP development tools"

### Cline (VS Code) Integration

1. **Open Cline settings** (Ctrl+Shift+P → "Cline: Open Settings")

2. **Add MCP server in `mcp_settings.json`:**

```json
{
  "mcpServers": {
    "mcp-abap-abap-adt-api": {
      "command": "node",
      "args": [
        "/absolute/path/to/mcp-abap-abap-adt-api/dist/index.js"
      ],
      "disabled": false,
      "autoApprove": []
    }
  }
}
```

3. **Reload Cline**

4. **Test in Cline chat:**
   ```
   @mcp-abap-abap-adt-api search for class ZCL_MY_CLASS
   ```

---

## Available Tools Overview

The server exposes **80+ MCP tools** organized into 28 handler categories:

### Core Operations
- `login` - Authenticate with SAP system
- `logout` - End SAP session
- `dropSession` - Clear cached session

### Object Management
- `searchObject` - Search for ABAP objects
- `getObject` - Retrieve object metadata
- `getObjectSource` - Get source code
- `setObjectSource` - Update source code
- `lock` / `unLock` - Lock management
- `createObject` - Create new ABAP objects
- `deleteObject` - Delete ABAP objects

### Development
- `activate` - Activate objects
- `syntaxCheckCode` - Syntax validation
- `codeCompletion` - Code suggestions
- `prettyPrint` - Format code
- `fixEdits` - Apply quick fixes

### Transport
- `createTransport` - Create transport requests
- `transportInfo` - Get transport details
- `releaseTransport` - Release transports
- `setTransportOwner` - Change transport owner

### Testing & Quality
- `runUnitTests` - Execute ABAP Unit tests
- `runAtcChecks` - Run ATC checks
- `getAtcResults` - Get ATC results

### Advanced
- `debugAttach` - Attach debugger
- `getRevisionHistory` - Object history
- `gitStaging` - abapGit operations
- `refactorRename` - Rename operations
- `analyzeTraces` - Trace analysis

**Full list:** Use `listTools()` MCP call or check source code in `src/handlers/`

---

## Troubleshooting

### Build Issues

**Problem:** TypeScript compilation errors
```
error TS2304: Cannot find name 'XYZ'
```

**Solution:**
```bash
# Update dependencies
npm update

# Clean install
rm -rf node_modules package-lock.json
npm install

# Verify TypeScript version
npx tsc --version
```

---

### Connection Issues

**Problem:** Cannot connect to SAP system
```
Error: connect ETIMEDOUT
```

**Solutions:**
1. **Check SAP URL:**
   ```bash
   curl -k https://your-sap-server.com:44300/sap/bc/adt/discovery
   # Should return XML
   ```

2. **Verify credentials:**
   ```bash
   # Check .env file
   cat .env | grep SAP_
   ```

3. **Test network:**
   ```bash
   ping your-sap-server.com
   telnet your-sap-server.com 44300
   ```

4. **SSL issues:**
   ```env
   # Add to .env
   NODE_TLS_REJECT_UNAUTHORIZED="0"
   ```

---

### Authentication Issues

**Problem:** Login fails with 401 Unauthorized
```
Error: Authentication failed
```

**Solutions:**
1. Verify username/password
2. Check SAP client number
3. Ensure user has ADT permissions
4. Check if account is locked in SAP

**Test manually:**
```bash
curl -k -u USERNAME:PASSWORD \
  "https://your-sap-server.com:44300/sap/bc/adt/discovery?sap-client=100&sap-language=EN"
```

---

### Runtime Issues

**Problem:** Server crashes on startup
```
Error: Cannot find module 'abap-adt-api'
```

**Solution:**
```bash
# Reinstall dependencies
npm install

# Rebuild
npm run build
```

**Problem:** MCP protocol errors
```
Error: Invalid MCP message format
```

**Solution:**
- Update `@modelcontextprotocol/sdk` to latest version
- Check client MCP SDK compatibility
- Verify stdio transport configuration

---

## Performance Optimization

### Session Caching

The server caches SAP sessions for better performance:

```typescript
// Sessions are cached per connection
// Use dropSession to clear cache:
await client.callTool('dropSession', {});
```

### Logging

Configure logging level:

```bash
# Set environment variable
export LOG_LEVEL=debug  # Options: error, warn, info, debug, trace

# Or in .env
LOG_LEVEL=info
```

View logs:
```bash
# Logs go to stderr
npm run start 2> server.log

# In separate terminal
tail -f server.log
```

---

## Security Best Practices

### 1. Credential Management

❌ **Don't:**
- Commit `.env` files to Git
- Share credentials in config files
- Use production credentials in development

✅ **Do:**
- Use environment variables
- Rotate passwords regularly
- Use separate credentials per environment
- Consider using SAP SSO/OAuth if available

### 2. Network Security

```bash
# Use HTTPS for SAP connection
SAP_URL=https://...  # Not http://

# Validate SSL certificates in production
# NODE_TLS_REJECT_UNAUTHORIZED="0"  # Only for development!
```

### 3. Access Control

- Limit SAP user permissions to minimum required
- Use transport-specific users if possible
- Log all object modifications
- Regular security audits

---

## Advanced Configuration

### TypeScript Configuration

Edit `tsconfig.json` for build customization:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### Custom Handlers

To add custom functionality:

1. Create new handler file:
```typescript
// src/handlers/CustomHandlers.ts
import { BaseHandler } from './BaseHandler.js';

export class CustomHandlers extends BaseHandler {
  getTools() {
    return [
      {
        name: 'myCustomTool',
        description: 'My custom tool',
        inputSchema: {
          type: 'object',
          properties: {
            param: { type: 'string', description: 'Parameter' }
          },
          required: ['param']
        }
      }
    ];
  }

  async callTool(name: string, args: any) {
    if (name === 'myCustomTool') {
      // Implementation
      return {
        content: [{ type: 'text', text: `Result: ${args.param}` }]
      };
    }
  }
}
```

2. Register in `src/index.ts`:
```typescript
import { CustomHandlers } from './handlers/CustomHandlers.js';

// In handler initialization
handlers.push(new CustomHandlers(this.getAdtConnectionForHandler.bind(this)));
```

3. Rebuild:
```bash
npm run build
```

---

## Monitoring and Debugging

### Enable Debug Mode

```bash
# Set debug environment
export DEBUG=mcp:*

# Run with verbose logging
npm run start 2>&1 | tee debug.log
```

### Monitor Performance

```javascript
// Add timing to tools
console.time('searchObject');
const result = await client.callTool('searchObject', { query: 'CL_*' });
console.timeEnd('searchObject');
```

### Health Checks

Create a health check script:

```javascript
// health-check.js
const health = {
  timestamp: new Date().toISOString(),
  server: 'running',
  tools: await client.listTools(),
  resources: await client.listResources()
};
console.log(JSON.stringify(health, null, 2));
```

---

## Maintenance

### Update Dependencies

```bash
# Check for updates
npm outdated

# Update all dependencies
npm update

# Update specific dependency
npm update abap-adt-api

# Rebuild after updates
npm run build
```

### Version Management

```bash
# Check current version
npm version

# Bump version (follows semver)
npm version patch  # 0.1.1 -> 0.1.2
npm version minor  # 0.1.1 -> 0.2.0
npm version major  # 0.1.1 -> 1.0.0
```

---

## Known Limitations

1. **Experimental Status**: Not recommended for production without thorough testing
2. **Limited Test Coverage**: Needs more comprehensive test suite
3. **Session Management**: Session caching may need improvements
4. **Error Handling**: Some edge cases may not be handled gracefully
5. **Documentation**: API documentation for extending is limited

See [reports/mcp-abap-abap-adt-api.md](reports/mcp-abap-abap-adt-api.md) for detailed analysis.

---

## Resources

### Documentation
- **Project README**: `README.md`
- **Analysis Report**: `reports/mcp-abap-abap-adt-api.md`
- **MCP Spec**: https://spec.modelcontextprotocol.io/
- **abap-adt-api**: https://github.com/marcellourbani/abap-adt-api

### Support
- **GitHub Issues**: https://github.com/mario-andreschak/mcp-abap-abap-adt-api/issues
- **MCP Discord**: https://discord.gg/modelcontextprotocol

### Related Projects
- **mcp-abap-adt**: Simpler read-only MCP server (58/100 maturity)
- **mcp-abap-adt-auth-broker**: JWT authentication service (75/100 maturity)
- **abap-adt-api**: Core library (88/100 maturity)

---

## Quick Reference Card

### Essential Commands
```bash
# Clone
git clone https://github.com/mario-andreschak/mcp-abap-abap-adt-api.git
cd mcp-abap-abap-adt-api

# Install
npm install

# Configure
cp .env.example .env
nano .env

# Build
npm run build

# Run
npm run start         # Production
npm run dev          # Development with Inspector

# Test
npm test             # Run tests
npm run test:coverage # With coverage
```

### File Locations
- **Source**: `src/`
- **Compiled**: `dist/`
- **Config**: `.env`, `tsconfig.json`, `package.json`
- **Handlers**: `src/handlers/` (28 files)
- **Entry point**: `src/index.ts` → `dist/index.js`

### Important URLs
- **MCP Inspector**: http://localhost:3000 (when using `npm run dev`)
- **SAP ADT Discovery**: `https://your-sap-server.com:44300/sap/bc/adt/discovery`

---

**Document created:** 2025-11-30
**For:** Claude Code
**Project version:** 0.1.1 (Experimental)
**Maturity score:** 72/100

🤖 Generated with [Claude Code](https://claude.com/claude-code)
