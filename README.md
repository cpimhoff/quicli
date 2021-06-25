# Quicli

Quicli (pronounced like `quickly`) is a shared definition for simple transactional interfaces, such as UIs and CLIs.

## Key Concepts
A server can emit a Quicli response for consumption by a Quicli client. Servers and clients can be mixed and matched. Server libraries (under `server/`) can help consumers easily construct valid Quicli responses. Client libraries (under `client/`) can help consumers easily construct interfaces to read Quicli responses and act upon them.

A Quicli response is a list of Quicli data under of of three groups:
- **Displays:** 
- **Parameters:**
- **Actions:**

## Spec
### Displays
Displays surface information to users as part of a Quicli interface. Help text describing how to use an interface, state from the server, or the results of an action are all representable by displays.

#### Text
Displays a string to users.
```ts
type TextDisplay = string | {
  type: "display.text",
  content: string,
}
```
`content` can be an inline Markdown string. This will allow some clients to render content as links or with text styling.

#### Heading
Displays a string to users prominently:
```ts
type HeadingDisplay = {
  type: "display.heading",
  content: string,
  level: number,
}
```

#### Group
Groups semantically relate content together.
```ts
type GroupDisplay = Item[] | {
  type: "display.group",
  content: Item[],
}
```

#### Error
Displays an error to the user.
```ts
type ErrorDisplay = {
  type: "display.error",
  content: Item,
  severity: "error"|"warning",
}
```

#### Table
Displays a table of information to users.
```
type TableDisplay = {
  type: "display.table",
  columnNames: string[], 
  rows: {
    name: (string in columnNames),
    content: Item,
  }[],
}
```

#### File
Displays a downloadable file to users. Note that the entire file content is transferred to the client regardless of whether they click on it or not.
```ts
type FileDisplay = {
  type: "display.file",
  fileName: string,
  mimetype: string,
  data: string,
}
```

#### Refresh
Content wrapped inside a refresh will be updated by the server anytime the value of the parameter(s) changes, or passively on a time interval.
```ts
type RefreshDisplay = {
  type: "display.refresh",
  content: Item,
  trigger:
    | { paramIds: string[] }
    | { intervalSeconds: number },
}
```

### Parameter
Parameters collect information from a user.

#### String
Collects a single string from the user.
```ts
type StringParameter = {
  type: "parameter.string",
  id: string,
}

type StringParameterValue = string
```

#### Number
Collects a single number from the user.
```ts
type NumberParameter = {
  type: "parameter.number",
  id: string,
  min: number | null,
  max: number | null,
  step: number | null,
}

type NumberParameterValue = number
```

#### List
The user needs to supply zero or more of the provided inputs.
```ts
type ListParameter = {
  type: "parameter.list",
  id: string,
  form: Item,
  minCount: number,
  maxCount: number | null,
}

type ListParameterValue = ParamValues[]
```
The `form` entry is repeated for each entry and defines the shape of what is collected.

#### Choice
The user must select one of the provided values.
```ts
type ChoiceOption = 
  | string
  | { display: string, value: any }

type ChoiceParameter = {
  type: "parameter.choice",
  id: string,
  options: ChoiceOption[],
}

type ChoiceParameterValue = string
```

#### MultiChoice
The user must select zero or more of the provided values.
```ts
type MultiChoiceParameter = {
  type: "parameter.multiChoice",
  id: string,
  options: ChoiceOption[],
  minCount: number | null,
  maxCount: number | null,
}

type MultiChoiceParameterValue = string[]
```

#### Boolean
The user must select either `true` or `false`.
```ts
type BooleanParameter = {
  type: "parameter.boolean",
  id: string,
}

type BooleanParameterValue = boolean
```

### Action