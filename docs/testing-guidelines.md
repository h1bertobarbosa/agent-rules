# Tests

Use the jest library to determine test scenarios and expectations, and use sinon to implement test patterns such as stub, spy, and mock.
To run the tests, use the `npm run test` command.
All tests should be located in the /test folder; do not place them in the /src folder along with the files being tested.
Tests should have the .test.ts extension.
Do not create dependencies between tests; each test should be run independently.
Follow the Arrange, Act, Assert or Given, When, Then principle to ensure maximum organization and readability within your tests.
If you are testing behavior that depends on a Date, and this is important to the test, use a Mock to ensure the test is repeatable.
If a test depends on external resources such as HTTP requests, databases, messaging, file systems, or APIs, they should be in the /test/integration folder; otherwise, they can be in the /test/unit folder.
Create tests for HTTP endpoints. These tests should not use libraries as supertests and should be integration tests. Furthermore, create these tests only to ensure the functioning of the main and alternative flows (mainly exploring status codes and error messages), leaving the variation of business rule testing to tests on use cases.
Create tests for all use cases; in this case, always test the main flows and at least one alternative flow that throws exceptions. Use the stub test pattern to avoid using external APIs at this level of testing.
Create tests for the entire domain, testing all possible rules, all possible variations, always at the unit level, without depending on any external resources.
Focus on testing one behavior per test; avoid writing very large tests.
Ensure that the code being written is fully covered by tests.
Create consistent expectations, ensuring that everything being tested is actually being checked.
Always close connections to the database or messaging platform after running tests.
Use beforeEach to initialize.