# note

You are in **/note** mode.

User input (must be on the same line as the command):
$ARGUMENTS

## Purpose
Capture, retrieve, or manage **project-specific notes** stored in `.eca/notes.md` in the repository root.
**Case-insensitive**: `add`, `Add`, and `ADD` all work the same way.

## Commands
- **Add**: Save a new note. Format: `/note add: <Title> | <Content>`
- **List**: View all notes. Format: `/note list`
- **Refer**: Retrieve a specific note. Format: `/note refer: <Title or ID>`
- **Delete**: Remove a note. Format: `/note delete: <Title or ID>`

## Hard Rules
1. **Case-Insensitive Processing**:
   - Convert the first word after `/note` to lowercase (e.g., `Add` → `add`).
   - Proceed based on the lowercase command.
2. **File Handling**:
   - Use `eca__read_file` to check if `.eca/notes.md` exists. If not, create it with a header:
     ```markdown
     # Project Notes
     ```
   - Use `eca__edit_file` to append/remove notes.
3. **Output**:
   - For `add`: Confirm with "Note added: <Title>."
   - For `list`: Return all notes in a clean format (no file paths).
   - For `refer`: Return the matched note or "Note not found."
   - For `delete`: Confirm with "Note deleted: <Title>."
4. **Errors**:
   - If $ARGUMENTS is empty or invalid, respond:
     "Usage: /note <add/list/refer/delete>: <details>."

## Examples
- `/note add: Refactor auth | Use JWT for stateless auth.`
- `/note LIST`
- `/note refer: refactor auth`
- `/note DELETE: refactor auth`