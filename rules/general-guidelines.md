# Coding Standards

All source code must be written in English.
Use camelCase for declaring methods, functions, and variables, PascalCase for classes and interfaces, and kebab-case for files and directories.
use SCREAMING_SNAKE_CASE for constants
Avoid abbreviations, but also avoid writing very long names (more than 30 characters).
Declare constants to represent magic numbers with readability.
Methods and functions must perform a clear and well-defined action, and this should be reflected in their name, which should begin with a verb, never a noun.
Whenever possible, avoid passing more than three parameters; use objects if necessary.
Avoid side effects. In general, a method or function should perform a mutation or query. Never allow a query to have side effects.
Never nest more than two if/else statements. Always use early returns.
Never use flag parameters to control the behavior of methods and functions. In these cases, extract them to methods and functions with behaviors. Specific
Avoid long methods, with more than 26 lines
Avoid long classes, with more than 260 lines
Always invert dependencies on external resources in both use cases and interface adapters using the Dependency Inversion Principle
Avoid blank lines within methods and functions
Avoid using comments whenever possible
Never declare more than one variable on the same line
Declare variables as close as possible to where they will be used
Prefer composition over inheritance whenever possible

# Review

After completing each task, run the tests and ensure they work.
Check code coverage; it must comply with the established guidelines.
Check code formatting to ensure it follows the project guidelines.
Run the linter to check if it is breaking any defined guidelines.
Check if build are working
Check if any part of the code is breaking any established best practices.
Check if any comments are missing.
Check if any values ​​are hardcoded.
Check if there are any unused imports.
Check if there are any unused variables.
Look for opportunities to make the code clearer and more objective.

# Logging

Never store logs in files; always redirect them through the process itself.
Never log sensitive data such as people's names, addresses, and credit card details.
Always be clear in log messages, without exaggerating or using long text.
Never silence exceptions; always log.
