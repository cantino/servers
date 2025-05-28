# Filesystem MCP Server

Node.js server implementing Model Context Protocol (MCP) for filesystem operations.

## Features

- Read/write files
- Create/list/delete directories
- Move files/directories
- Search files
- Get file metadata

**Note**: The server will only allow operations within directories specified via `args`. 

### Exclusion Patterns

You can specify paths to exclude by adding an exclamation mark followed by comma-separated patterns:

- **Basic exclusions**: `/path/to/allowed/dir!.env,dist` will exclude .env and dist
- **Negating exclusions**: Adding a `!` before a pattern will explicitly include something that would otherwise be excluded: `/path/to/dir!.env,!.git` would exclude `.env` but override the default exclusion of `.git`
- **Case-insensitivity**: All exclusions are case-insensitive for security on all operating systems (e.g., `.git` will also match `.GIT` or `Git`)

**Default exclusions**: `.git` and `node_modules` are always excluded by default unless explicitly included with a negated pattern.

## API

### Resources

- `file://system`: File system operations interface

### Tools

- **read_file**
  - Read complete contents of a file
  - Input: `path` (string)
  - Reads complete file contents with UTF-8 encoding

- **read_multiple_files**
  - Read multiple files simultaneously
  - Input: `paths` (string[])
  - Failed reads won't stop the entire operation

- **write_file**
  - Create new file or overwrite existing (exercise caution with this)
  - Inputs:
    - `path` (string): File location
    - `content` (string): File content

- **edit_file**
  - Make edits to a text file by replacing specified text segments.
  - Features:
    - Exact match: First attempts to find an exact match for `oldText`.
    - Line-based fallback: If no exact match, it attempts a line-by-line comparison where each line from `oldText` is compared to lines in the content after trimming whitespace.
    - Indentation preservation: When replacing content using the line-based fallback, the indentation of the first line of the matched block is preserved in `newText`.
    - Sequential edits: Applies multiple edits in the order they are provided.
    - Git-style diff output: Returns a diff of the changes.
    - Dry run mode: Allows previewing changes before they are applied.
  - Inputs:
    - `path` (string): File to edit
    - `edits` (array): List of edit operations
      - `oldText` (string): Text to search for.
      - `newText` (string): Text to replace with.
    - `dryRun` (boolean): Preview changes without applying (default: false).
  - Returns a git-style diff. If `dryRun` is true, changes are not written to disk.
  - Best Practice: Always use dryRun first to preview changes before applying them

- **create_directory**
  - Create new directory or ensure it exists
  - Input: `path` (string)
  - Creates parent directories if needed
  - Succeeds silently if directory exists

- **list_directory**
  - List directory contents with [FILE] or [DIR] prefixes
  - Input: `path` (string)

- **directory_tree**
  - Get a recursive tree view of files and directories as a JSON structure. Each entry includes 'name', 'type' (file/directory), and 'children' for directories. Files have no children array, while directories always have a children array (which may be empty). The output is formatted with 2-space indentation for readability. Only works within allowed directories.
  - Input: `path` (string)
  - Returns: A JSON string representing the directory tree.

- **move_file**
  - Move or rename files and directories
  - Inputs:
    - `source` (string)
    - `destination` (string)
  - Fails if destination exists

- **search_files_by_name**
  - Recursively search for files/directories
  - Inputs:
    - `path` (string): Starting directory
    - `pattern` (string): Search pattern
    - `excludePatterns` (string[]): Exclude any patterns. Glob formats are supported.
  - Case-insensitive matching
  - Returns full paths to matches

- **search_files_by_content**
  - Search for text patterns inside files using ripgrep. This powerful search examines file contents rather than just names, making it perfect for finding specific code, text, or data patterns across multiple files. Results include filename, line number, and the matching line content. Options include case sensitivity controls and result limiting. Only searches within allowed directories.
  - Inputs:
    - `path` (string): Starting directory for the search.
    - `pattern` (string): The text pattern to search for.
    - `excludePatterns` (string[] optional): Glob patterns to exclude files/directories from the search. Defaults to [].
    - `caseSensitive` (boolean optional): Whether the search should be case-sensitive. Defaults to false.
    - `maxResults` (number optional): The maximum number of results to return. Defaults to 100.
  - Returns: A list of matches, including file path, line number, and the content of the matching line.

- **get_file_info**
  - Get detailed file/directory metadata
  - Input: `path` (string)
  - Returns:
    - Size
    - Creation time
    - Modified time
    - Access time
    - Type (file/directory)
    - Permissions

- **list_allowed_directories**
  - List all directories the server is allowed to access
  - No input required
  - Returns:
    - Directories that this server can read/write from

## Usage with Claude Desktop
Add this to your `claude_desktop_config.json`:

Note: you can provide sandboxed directories to the server by mounting them to `/projects`. Adding the `ro` flag will make the directory readonly by the server.

### Docker
Note: all directories must be mounted to `/projects` by default.

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "docker",
      "args": [
        "run",
        "-i",
        "--rm",
        "--mount", "type=bind,src=/Users/username/Desktop,dst=/projects/Desktop",
        "--mount", "type=bind,src=/path/to/other/allowed/dir,dst=/projects/other/allowed/dir,ro",
        "--mount", "type=bind,src=/path/to/file.txt,dst=/projects/path/to/file.txt",
        "mcp/filesystem",
        "/projects/Desktop",
        "/projects/other/allowed/dir!.env,logs",
        "/projects/path/to/file.txt"
      ]
    }
  }
}
```

In this example, `/projects/other/allowed/dir` will have `.env` and `logs` directories excluded (along with the default `.git` and `node_modules`).

### NPX

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/username/Desktop",
        "/path/to/other/allowed/dir!.env,dist"
      ]
    }
  }
}
```

In this example, `.env` and `dist` directories will be excluded from access along with the default excluded paths (`.git` and `node_modules`).

## Build

Docker build:

```bash
docker build -t mcp/filesystem -f src/filesystem/Dockerfile .
```

## License

This MCP server is licensed under the MIT License. This means you are free to use, modify, and distribute the software, subject to the terms and conditions of the MIT License. For more details, please see the LICENSE file in the project repository.
