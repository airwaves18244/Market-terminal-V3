```markdown
# Market-terminal-V3 Development Patterns

> Auto-generated skill from repository analysis

## Overview
This skill teaches you the core development patterns and conventions used in the Market-terminal-V3 TypeScript codebase. You'll learn how to structure files, write imports and exports, follow commit message practices, and understand the project's approach to testing. This guide is ideal for contributors who want to maintain consistency and quality in their code contributions.

## Coding Conventions

### File Naming
- Use **snake_case** for all file names.
  - Example: `market_data_service.ts`

### Import Style
- Use **relative imports** for referencing other modules within the project.
  - Example:
    ```typescript
    import { fetchMarketData } from './market_utils';
    ```

### Export Style
- Use **named exports** for all exported functions, classes, or constants.
  - Example:
    ```typescript
    // In market_utils.ts
    export function fetchMarketData() { ... }
    export const MARKET_STATUS = 'open';
    ```

### Commit Messages
- Commit messages are **freeform** and do not follow a strict prefix or type.
- Average commit message length is about 91 characters.
  - Example:
    ```
    Add support for new market data endpoint and update error handling logic
    ```

## Workflows

### Adding a New Feature
**Trigger:** When you want to introduce new functionality.
**Command:** `/add-feature`

1. Create a new file using snake_case naming (e.g., `new_feature.ts`).
2. Implement the feature using TypeScript.
3. Use relative imports to bring in any dependencies.
4. Export your functions or classes using named exports.
5. Write corresponding tests in a file named `new_feature.test.ts`.
6. Commit your changes with a clear, descriptive message.

### Fixing a Bug
**Trigger:** When you need to resolve an issue or bug.
**Command:** `/fix-bug`

1. Locate the relevant file(s) using snake_case naming.
2. Make the necessary code changes.
3. Update or add tests in `*.test.ts` files to cover the fix.
4. Commit your changes with a descriptive message explaining the fix.

### Running Tests
**Trigger:** When you want to verify code correctness.
**Command:** `/run-tests`

1. Identify test files by the `*.test.*` pattern (e.g., `market_utils.test.ts`).
2. Use the project's test runner (framework unknown; check project documentation or scripts).
3. Review test results and address any failures.

## Testing Patterns

- Test files follow the `*.test.*` naming convention, typically `module_name.test.ts`.
- The testing framework is **unknown**; check the project for specific test runner usage.
- Place tests alongside or near the modules they test.
- Example test file structure:
  ```typescript
  // market_utils.test.ts
  import { fetchMarketData } from './market_utils';

  describe('fetchMarketData', () => {
    it('should return market data for a valid symbol', () => {
      // test implementation
    });
  });
  ```

## Commands
| Command      | Purpose                                         |
|--------------|-------------------------------------------------|
| /add-feature | Start the workflow for adding a new feature     |
| /fix-bug     | Start the workflow for fixing a bug             |
| /run-tests   | Run all tests in the codebase                   |
```
