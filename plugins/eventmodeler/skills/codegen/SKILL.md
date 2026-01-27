---
name: codegen
description: Build code generators that consume eventmodeler codegen output. Use when helping users create custom code generators for their tech stack.
---

# Building Code Generators for Event Models

The `eventmodeler codegen slice` command outputs structured JSON that code generators can consume. This skill teaches how to build code generators that transform event model slices into working code.

## The Codegen Pipeline

```
eventmodeler codegen slice "Place Order" | your-generator > output-files
```

Your generator receives JSON on stdin and produces code files.

## Understanding the Input

Run codegen to see what you'll receive:

```bash
eventmodeler codegen slice "Place Order"
```

### Top-Level Structure

```json
{
  "sliceType": "STATE_CHANGE | STATE_VIEW | AUTOMATION",
  "slice": { "id": "...", "name": "Place Order" },
  "chapter": { "name": "...", "hierarchy": ["Parent", "Child"] },
  "elements": { ... },
  "inboundDependencies": [ ... ],
  "internalFlows": [ ... ],
  "scenarios": [ ... ]
}
```

### Slice Types and What to Generate

| sliceType | Components | Generate |
|-----------|------------|----------|
| `STATE_CHANGE` | Screen, Command, Event | API endpoint, command handler, event, tests |
| `STATE_VIEW` | ReadModel | Projection/denormalizer, query endpoint |
| `AUTOMATION` | Processor, ReadModel, Command, Event | Event handler, saga/process manager |

## Building a Generator: Step by Step

### 1. Parse the Input

```typescript
// generator.ts
import { stdin } from 'process';

interface CodegenInput {
  sliceType: 'STATE_CHANGE' | 'STATE_VIEW' | 'AUTOMATION';
  slice: { id: string; name: string };
  elements: {
    commands: Array<{ name: string; fields: Field[] }>;
    events: Array<{ name: string; fields: Field[]; aggregate?: string }>;
    readModels: Array<{ name: string; fields: Field[] }>;
    screens: Array<{ name: string; fields: Field[] }>;
    processors: Array<{ name: string }>;
  };
  inboundDependencies: InboundDependency[];
  internalFlows: Flow[];
  scenarios: Scenario[];
}

async function readInput(): Promise<CodegenInput> {
  const chunks: Buffer[] = [];
  for await (const chunk of stdin) {
    chunks.push(chunk);
  }
  return JSON.parse(Buffer.concat(chunks).toString());
}
```

### 2. Route by Slice Type

```typescript
async function main() {
  const input = await readInput();

  switch (input.sliceType) {
    case 'STATE_CHANGE':
      generateStateChangeSlice(input);
      break;
    case 'STATE_VIEW':
      generateStateViewSlice(input);
      break;
    case 'AUTOMATION':
      generateAutomationSlice(input);
      break;
  }
}
```

### 3. Generate from Elements

**Commands → DTOs/Request types:**
```typescript
function generateCommand(cmd: Command): string {
  const fields = cmd.fields
    .map(f => `  ${f.name}: ${mapType(f.type)}${f.optional ? ' | null' : ''};`)
    .join('\n');

  return `export interface ${cmd.name}Command {\n${fields}\n}`;
}
```

**Events → Event types with metadata:**
```typescript
function generateEvent(evt: Event): string {
  const fields = evt.fields.map(f => {
    const generated = f.generated ? ' // generated' : '';
    return `  ${f.name}: ${mapType(f.type)};${generated}`;
  }).join('\n');

  return `export interface ${evt.name}Event {\n  type: '${evt.name}';\n${fields}\n}`;
}
```

**ReadModels → Projection types:**
```typescript
function generateReadModel(rm: ReadModel): string {
  const fields = rm.fields
    .map(f => `  ${f.name}: ${mapType(f.type)}${f.optional ? ' | null' : ''};`)
    .join('\n');

  return `export interface ${rm.name} {\n${fields}\n}`;
}
```

### 4. Use Field Mappings for Projections

The `inboundDependencies` show how external events map to read model fields:

```typescript
function generateProjection(input: CodegenInput): string {
  const rm = input.elements.readModels[0];
  const handlers: string[] = [];

  for (const dep of input.inboundDependencies) {
    if (dep.target.name === rm.name) {
      const mappings = dep.fieldMappings
        .map(m => `    state.${m.targetFieldName} = event.${m.sourceFieldName};`)
        .join('\n');

      handlers.push(`
  on${dep.source.name}(state: ${rm.name}, event: ${dep.source.name}Event) {
${mappings}
  }`);
    }
  }

  return `export class ${rm.name}Projection {\n${handlers.join('\n')}\n}`;
}
```

### 5. Generate Tests from Scenarios

Scenarios are BDD specs that become tests. There are two scenario patterns:

**State-Change Scenarios** (whenCommand → then events/error):
- `whenCommand`: The command being tested
- `then.type: 'events'`: Expected events produced
- `then.type: 'error'`: Expected error

**Automation Scenarios** (whenEvents → then command):
- `whenEvents`: The event(s) that trigger the processor
- `then.type: 'command'`: The command the processor should issue

```typescript
function generateTests(input: CodegenInput): string {
  return input.scenarios.map(scenario => {
    const givenSetup = scenario.given
      .map(g => `    await given(new ${g.eventName}(${JSON.stringify(g.fieldValues)}));`)
      .join('\n');

    // State-change scenarios: when is a command
    let whenAction = '';
    if (scenario.whenCommand) {
      whenAction = `    await when(new ${scenario.whenCommand.commandName}(${JSON.stringify(scenario.whenCommand.fieldValues)}));`;
    }
    // Automation scenarios: when is event(s)
    if (scenario.whenEvents?.length > 0) {
      whenAction = scenario.whenEvents
        .map(e => `    await whenEvent(new ${e.eventName}(${JSON.stringify(e.fieldValues)}));`)
        .join('\n');
    }

    let thenAssertion: string;
    if (scenario.then.type === 'events') {
      const expected = scenario.then.expectedEvents!
        .map(e => `new ${e.eventName}(${JSON.stringify(e.fieldValues)})`)
        .join(', ');
      thenAssertion = `    await thenExpect([${expected}]);`;
    } else if (scenario.then.type === 'command') {
      // Automation scenario: expect a command to be issued
      const cmd = scenario.then.expectedCommand!;
      thenAssertion = `    await thenExpectCommand(new ${cmd.commandName}(${JSON.stringify(cmd.fieldValues)}));`;
    } else if (scenario.then.type === 'error') {
      thenAssertion = `    await thenExpectError('${scenario.then.errorType}', '${scenario.then.errorMessage}');`;
    } else {
      thenAssertion = `    await thenExpectState(${JSON.stringify(scenario.then.expectedFieldValues)});`;
    }

    return `
  test('${scenario.name}', async () => {
${givenSetup}
${whenAction}
${thenAssertion}
  });`;
  }).join('\n');
}
```

## Type Mapping

Map event model types to your target language:

```typescript
function mapType(fieldType: string, isList?: boolean): string {
  const baseType = {
    'UUID': 'string',
    'String': 'string',
    'Int': 'number',
    'Long': 'number',
    'Double': 'number',
    'Decimal': 'Decimal',  // or 'number' or 'BigNumber'
    'Boolean': 'boolean',
    'Date': 'Date',
    'DateTime': 'Date',
    'Custom': 'object',  // handle subfields separately
  }[fieldType] ?? 'unknown';

  return isList ? `${baseType}[]` : baseType;
}
```

For Custom types with subfields, generate inline or separate types:

```typescript
function generateFieldType(field: Field): string {
  if (field.type === 'Custom' && field.subfields) {
    const nested = field.subfields
      .map(sf => `${sf.name}: ${mapType(sf.type)}`)
      .join('; ');
    const type = `{ ${nested} }`;
    return field.list ? `Array<${type}>` : type;
  }
  return mapType(field.type, field.list);
}
```

## Handling Aggregates

Events include their aggregate name when inside one:

```typescript
function generateEventHandler(input: CodegenInput): string {
  const event = input.elements.events[0];
  const aggregate = event.aggregate;

  if (aggregate) {
    return `
export class ${aggregate}Aggregate {
  apply(event: ${event.name}Event) {
    // Update aggregate state from event
  }
}`;
  }
  // ...
}
```

## Full Generator Example

```typescript
#!/usr/bin/env node
import { stdin, stdout } from 'process';

async function main() {
  const input = await readInput();
  const files: Record<string, string> = {};

  // Generate types for all elements
  for (const cmd of input.elements.commands) {
    files[`${cmd.name}.command.ts`] = generateCommand(cmd);
  }
  for (const evt of input.elements.events) {
    files[`${evt.name}.event.ts`] = generateEvent(evt);
  }
  for (const rm of input.elements.readModels) {
    files[`${rm.name}.read-model.ts`] = generateReadModel(rm);
    files[`${rm.name}.projection.ts`] = generateProjection(input, rm);
  }

  // Generate tests from scenarios
  if (input.scenarios.length > 0) {
    files[`${input.slice.name}.test.ts`] = generateTests(input);
  }

  // Output (write to files or stdout)
  for (const [filename, content] of Object.entries(files)) {
    console.log(`// === ${filename} ===`);
    console.log(content);
    console.log();
  }
}

main();
```

## Running Your Generator

```bash
# Single slice
eventmodeler codegen slice "Place Order" | node generator.js

# All slices (bash loop)
for slice in $(eventmodeler list slices --format names); do
  eventmodeler codegen slice "$slice" | node generator.js
done

# Pipe to file writer
eventmodeler codegen slice "Place Order" | node generator.js | write-files.sh
```

## Best Practices

1. **Handle all slice types** - Your generator should handle STATE_CHANGE, STATE_VIEW, and AUTOMATION
2. **Use field mappings** - They define the projection logic between events and read models
3. **Generate tests from scenarios** - Scenarios are executable specifications
4. **Respect field attributes** - `generated` fields don't need input, `optional` fields can be null
5. **Use aggregate info** - Events in aggregates should be handled by that aggregate
6. **Handle Custom types recursively** - Subfields can themselves be Custom types
7. **Make it idempotent** - Running twice should produce the same output
