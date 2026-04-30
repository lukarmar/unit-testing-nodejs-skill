# unit-testing-nodejs-skill

> A Claude Skill for generating and reviewing unit tests in Node.js projects — covering Jest, Vitest, NestJS, Express, Fastify, and plain Node.js services.

## What is a Claude Skill?

A **Skill** is a set of instructions and reference files that guide Claude to produce consistent, high-quality outputs for a specific task. When this skill is active, Claude automatically applies its rules whenever you ask it to write, review, or improve unit tests in a Node.js project.

---

## What this skill does

When loaded, Claude will:

- **Detect your stack automatically** (Jest vs Vitest, NestJS vs Express/Fastify/plain Node, TypeScript vs JavaScript) before writing any test
- **Apply the correct patterns** for each framework combination
- **Enforce a standard folder structure** for mocks, fixtures, factories and builders
- **Follow the AAA pattern** (Arrange → Act → Assert) consistently
- **Write tests that read like specifications**, not implementation notes
- **Generate all required scenarios** per unit type (happy path, not-found, edge cases)

---

## Folder structure

```
unit-testing-nodejs/
├── SKILL.md                        ← Main skill entry point
└── references/
    ├── project-detection.md        ← Stack detection logic
    ├── jest-patterns.md            ← Jest patterns (Express, Fastify, plain Node)
    ├── vitest-nestjs.md            ← Vitest patterns for NestJS
    ├── vitest-node.md              ← Vitest patterns outside NestJS
    ├── naming-patterns.md          ← Naming conventions and coverage per layer
    ├── setup-config.md             ← Runner setup (jest.config, vitest.config, tsconfig)
    ├── clean-code.md               ← Clean code rules for test code
    └── test-assets.md              ← Mocks, fixtures, factories, builders guide
```

---

## Supported stacks

| Test runner | Framework | Reference file |
|---|---|---|
| Vitest | NestJS | `references/vitest-nestjs.md` |
| Jest | NestJS | `references/jest-patterns.md` |
| Vitest | Express / Fastify / plain Node | `references/vitest-node.md` |
| Jest | Express / Fastify / plain Node | `references/jest-patterns.md` |

---

## Core rules enforced

- Spec files live **next to the implementation** (`orders.service.ts` → `orders.service.spec.ts`)
- Mocks → `test/mocks/` | Fixtures → `test/fixtures/` | Factories → `test/factories/` | Builders → `test/builders/`
- No comments inside test files
- No mocks or fixtures declared inside the test file
- All mocks based on the real implementation contract
- Test names follow the pattern: `should [observable outcome] when [precise condition]`
- `clearAllMocks()` called in `beforeEach` on every test suite

---

## How to use

### Option 1 — Upload to Claude.ai (Projects)

1. Open [Claude.ai](https://claude.ai) and go to a **Project**
2. In the project settings, upload the `unit-testing-nodejs` folder as a skill
3. Claude will automatically apply it whenever you ask to write or review tests

### Option 2 — Reference manually

Point Claude to the `SKILL.md` file and it will load the appropriate references based on your stack.

---

## Example prompts

```
Write unit tests for the OrdersService
```
```
Generate a spec file for the AuthGuard
```
```
Review the test coverage of my UserController
```
```
Create a builder for the CreateOrderDto
```

---

## Repository

[github.com/lukarmar/unit-testing-nodejs-skill](https://github.com/lukarmar/unit-testing-nodejs-skill)