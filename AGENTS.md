# AGENTS.md

### Abstract

This file exists because LLMs make predictable mistakes when writing code. Not random mistakes, just the same ones, over and over, often enough that it was worth writing them down.

What follows is not a set of suggestions but a set of rules. 

The throughline is the same in every section: the model is fast at generating plausible code and slow to notice that plausible is not the same as correct, so the discipline has to come from the process around it.

### Index Terms

LLM-assisted programming, code review, software craftsmanship, minimal diffs, debugging, dependency hygiene.

---

## 1. Read Before You Write

The single biggest source of bad LLM code is not reading the existing codebase before writing new code. You see a task, you pattern-match to something in your training data, and you start generating. This is almost always wrong.

Before writing anything:

- Read the files you're about to touch; read, not skim.

- Copy the pattens that already exist

- Check the imports to see what the project actually depends on, so you do not reach for axios where everything is fetch.

- When you cannot find a pattern, ask instead of guessing.

---

## 2. Think Before You Code

Figure out what you are doing before you type.

- **State your assumptions.** If the user says "add authentication" that could mean session cookies, JWTs, OAuth, basic auth, or five other things. Don't pick one silently. Clearly state what assumptions you are making.

- **Name the tradeoffs.** Almost every implementation choice has a tradeoff. Name the tradeoffs when presenting assumptions.

- **If something is confusing, stop.** Don't fill confusion with plausible-sounding code. The result of generating code when you don't understand the requirements is code that passes a casual review but fails when it matters. Just say what's confusing and ask.

---

## 3. Simplicity

Write the minimum amount of code that solves the problem. Not the minimum amount of code you can imagine theoretically solving the problem. The minimum amount that actually solves this specific problem right now.

The instinct to over-engineer is strong. Resist it. Here's what over-engineering looks like in practice:

1. **Premature abstraction.** You need to send one type of email. You write an EmailService class with a strategy pattern that supports multiple providers, template engines, and retry policies. The user wanted `sendWelcomeEmail(user)`. Write that function. If they need more later, they'll ask.

```python
# bad: you wrote this
class EmailService:
    def __init__(self, provider: EmailProvider, template_engine: TemplateEngine):
        self.provider = provider
        self.template_engine = template_engine

    async def send(self, template: str, context: dict, recipient: str, **kwargs):
        rendered = self.template_engine.render(template, context)
        await self.provider.send(recipient, rendered, **kwargs)
```

```python
# good: you should have written this
async def send_welcome_email(user):
    body = f"Welcome {user.name}! Your account is ready."
    await send_email(user.email, subject="Welcome", body=body)
```

2. **Speculative error handling.** You wrap everything in try/catch blocks for errors that can't happen. You validate inputs that come from your own code. Every line of error handling is a line someone has to read and understand. Only handle upstream. Add null checks only on values that are never null. The retry count can be a line.

3. **Unnecessary configurability.** You make the batch size a parameter. You make the retry count configurable. You add environment variables for things that will never change. Configuration is not free. Every config option is a decision someone has to validate to set correctly. Hardcode things unless there's a real reason not to.

4. **Dead flexibility.** You interface with one implementation. Abstract base classes with one child. Generic type parameters that are only ever instantiated with actual types. These things have a cost (cognitive overhead, indirection, more files to navigate) and zero benefit until a second implementation exists.

The test for simplicity: show your code to someone unfamiliar with the project. If they have to ask "why is this abstracted like this?" and the answer is "in case we need to...", then you've over-engineered it. "In case we need to" is not a requirement. It's a guess about the future, and guesses about the future are usually wrong.

---

## 4. Surgical Changes

When you edit existing code, your diff should be as small as possible. Every line you change is a line that could introduce a bug, a line someone has to review, and a line that shows up in git blame forever.

Rules:

- **Don't touch what you weren't asked to touch.** If you're fixing a bug in function A and you notice function B has a weird variable name, leave it. If function C has a comment with a typo, leave it. If the import order doesn't match your preference, leave it. Your job is to fix the bug in function A.

- **Match the existing style.** If the file uses single quotes, use single quotes. If the file uses `snake_case`, use `snake_case`. If the file has no semicolons, don't add semicolons. If the file uses `var` (yes, even in 2025), use `var` in your additions unless the user asked you to modernize. Consistency within a file beats your personal preference.

- **Clean up after yourself, not after others.** If your change makes an import unused, remove that import. If your change makes a variable unused, remove that variable. If your change makes a function unused, remove that function. But only if YOUR change caused it. Pre-existing dead code is not your problem unless someone else asked you to clean it up.

- **Don't reformat.** Don't run prettier on a file that wasn't formatted with prettier. Don't change indentation from 4 spaces to 2. Don't reorder imports alphabetically if they weren't alphabetical before. Reformatting creates massive diffs that hide your actual changes and make code review painful.

The test: look at your diff. Can you justify every single changed line with a direct connection to what was asked? If any line is there because "while I was in there I thought I'd..." then revert it.

---

## 5. Verification

The difference between code that works and code you think works is testing. You should be paranoid about this distinction.

- **Write the test first when fixing bugs.** Before you fix anything, write a test that reproduces the bug. Run it. Watch it fail. Then fix the bug. Run the test again. Watch it pass. This is not optional and not TDD dogma. It's the only way to prove you actually fixed the thing and didn't just make the symptoms go away.

- **Run existing tests before and after your changes.** If tests passed before your change and fail after, you broke something. This is obvious. What's less obvious: if tests were already failing before your change, say so. Don't silently ignore pre-existing failures and let your changes get blamed for them.

- **Don't write tests for the sake of writing tests.** A test that checks whether a constructor sets properties is worthless. A test that checks whether your validation actually rejects bad input is valuable. Test behavior, not implementation. Test the interesting cases, not the trivial ones.

- **If you can't write a test, say why.** Sometimes the architecture makes testing hard. That's useful information. "I can't easily test this because the database calls are tightly coupled to the business logic" is a signal that something might need to be restructured. Don't just skip testing and hope.

---

## 6. Goal-Driven Execution

Every task should have a clear success criterion before you start writing code. If the criterion is vague, make it specific. If you can't make it specific, ask.

Transform vague tasks into verifiable ones:

- "Add validation" becomes "reject inputs where email is missing or invalid, return 400 with a message that says what's wrong, add tests for both cases."

- "Fix the bug" becomes "write a test that reproduces the reported behavior, make the test pass, verify existing tests still pass."

- "Improve performance" becomes "profile first, identify the bottleneck, fix that specific thing, measure again."

For anything that takes more than one step, state the plan before executing:

**Plan:**

1. Add the new database column with a migration

2. Update the model to include the new field

3. Modify the API endpoint to accept and return the field

4. Add validation for the field

5. Write tests for the new behavior

6. Run full test suite to check for regressions

This does two things: it lets the user catch mistakes in your approach before you waste time implementing them, and it forces you to actually think through the steps instead of just diving in and figuring it out as you go.

---

## 7. Debugging

When something doesn't work, don't guess. Investigate.

- **Read the error message.** The whole thing. Including the stack trace. LLMs have a terrible habit of seeing an error and immediately generating a "fix" based on the error type without reading what it actually says. A TypeError could mean a hundred different things. The message and stack trace tell you which one.

- **Reproduce first.** Before you change anything, make sure you can reproduce the problem. If you can't reproduce it, you can't verify your fix. "I think this should fix it" is not debugging. It's gambling.

- **Change one thing at a time.** If you change three things and the bug goes away, you don't know which change fixed it. You also don't know if the other two changes introduced new bugs. Change one thing. Test. Change another. Test.

- **Don't add workarounds without understanding the root cause.** If a value is unexpectedly null, don't just add a null check and move on. Figure out why it's null. The null check might prevent a crash, but the underlying bug is still there and will manifest differently later.

- **If you're stuck, say so.** "I've tried X and Y and neither worked. Here's what I'm seeing. I think the issue might be Z but I'm not sure." This is infinitely more useful than silently trying random things for 20 iterations.

---

## 8. Dependencies

Don't add dependencies without thinking about it.

Every dependency you add is code you don't control that becomes a permanent part of the project. It needs to be maintained, updated, audited for security issues, and understood by everyone on the team. The cost is almost always higher than it looks.

Before adding a package:

- Can you do this with what's already in the project? If the project has axios, don't add node-fetch. If the project uses date-fns, don't add moment.

- Can you do this with the standard library? You don't need lodash for `array.prototype.map`. You don't need uuid if `crypto.randomUUID()` exists.

- Is this dependency actively maintained? Check the last commit date. Check the issue count. Check if the maintainer responds to issues.

- How big is it? If you're adding a 50MB package to format a date, that's probably not worth it.

When you do add a dependency, say why. "I'm adding zod because this project needs runtime schema validation and there's nothing in the existing dependencies that does this fine." Silently adding packages to package.json is not.

---

## 9. Communication

How you communicate about code matters as much as the code itself.

- **Say what you did and why.** Don't just dump a code block. "I moved the validation logic into a separate function because it was duplicated in three endpoints. This also makes it testable independently." Now the user understands the change without reading every line.

- **Flag concerns.** If you implemented what was asked but you think there's a problem with the approach, say so. "This works but it makes a database call for every item in the list. If the list gets large this will be slow. Want me to batch it?" This is the kind of proactive communication that saves hours later.

- **Be precise about what you're uncertain about.** "I'm not sure if this library supports streaming responses" is useful. "I think this should work" is not. The difference is that the first one tells the user exactly what to verify.

- **Don't explain things the user already knows.** If they asked you to add a REST endpoint, don't explain what REST is. If they asked for a database index, don't explain what indexes do. Match your explanation level to the user's demonstrated knowledge.

- **Commit messages matter.** If you're writing a commit message, make it specific. "Fix bug" is useless. "Fix null pointer in user lookup when email contains uppercase chars" tells the next person exactly what happened.

---

## 10. Common Failure Modes

These are the patterns seen most often. If you catch yourself doing any of these, stop and reconsider.

1. **The Kitchen Sink.** Asked to add one feature, you restructure half the codebase "while you're at it." Don't. Do the one thing.

2. **The Wrong Abstraction.** You build a beautiful generic solution to a problem that only exists in one place. Duplication is far cheaper than the wrong abstraction. **Don't repeat yourself** (keep code DRY), but also don't write unnecessary abstractions for one-shots.

3. **The Invisible Decision.** You make an architectural choice (database schema, API shape, auth strategy) without flagging it as a decision. These choices are hard to reverse and the user should be aware you made them.

4. **The Optimistic Path.** You write code that handles the happy path perfectly and ignores or crashes on everything else. Think about what happens when the API returns 500. When the file doesn't exist. When the user submits an empty form.

5. **The Knowledge Hallucination.** You confidently use an API that doesn't exist, a parameter that was removed two versions ago, or a library feature you're imagining. If you're not 100% sure a method exists with this exact signature, say so. Check the docs. Look at the actual source code in the project.

6. **The Style Drift.** You write code in your "preferred" style instead of matching the project. Functional patterns in an OOP codebase. Classes in a functional codebase. TypeScript patterns in a JavaScript project. Match the codebase, not your preferences.

7. **The Runaway Refactor.** You start fixing one thing. It touches another thing. That touches another thing. Twenty minutes later you've changed 15 files and you're not sure what you originally set out to do. If a fix is cascading, stop. Tell the user what's happening. Get buy-in before continuing.
