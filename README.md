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
  text: string,
}
```
`text` can be an inline Markdown string. This will allow some clients to render content as links or with text styling.

#### Heading
Displays a string to users prominently.
```ts
type HeadingDisplay = {
  type: "display.heading",
  text: string,
  level: number,
}
```

#### List
Displays a list of items to users.
```ts
type ListDisplay = {
  type: "display.list",
  content: Item,
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
```ts
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

#### Modes
Modes define a disjunctive display. While one mode is being shown, all the other modes are not. In a UI, this may be represented as a tab bar. In a CLI, this may be represented as subcommands.
```ts
type Mode = {
  name: string,
  content: Item,
}

type ModeDisplay = {
  type: "display.mode",
  modes: Mode[],
}
```

#### Metadata
Metadata is not displayed to the end user but can contain any extra data that a client may find useful for debugging or to provide special features not part of the standard library.
```ts
type MetadataDisplay = {
  type: "display.metadata",
  key: string,
  value: any,
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

#### Button
When a button is clicked, it executes the action with an `id` matching `actionId`. Buttons have the value of `true` when they were what invoked the action, else `false`.
```ts
type ButtonParameter = {
  type: "parameter.button",
  id: string,
  actionId: string,
  text: string,
  isDestructive: boolean,
}

type ButtonParameterValue = boolean
```

Whenever an action is invoked, the button which invoked its `id` is included under the special `_invokerButtonId` argument. This can be used to determine the source button more easily than checking the boolean flag of each button parameter.

### Action
Actions represent things you can _do_ in a Quicli response.

#### Transaction
```ts
type TransactionAction = {
  type: "action.transaction",
  id: string,
  version: string,
  content: Item,
}
```
When a `Transaction` is invoked, all parameters inside of the `Transaction` are collected and sent to the server for processing. The server may then respond with any valid Quicli response, which will be appended to the end of the `Transaction`. In this manner, the server may display results of the action or chain together another step in a multi-part form.

#### Refresh
When a `Refresh` is invoked, it collects the value of the given `collectParamIds` and sends them to the server for processing. The contents are replaced by new results from the server. `Refresh`s can be invoked by `Buttons` or by its `trigger` property, such as automatically on a time interval or whenever the given parameter is modified.
```ts
type RefreshDisplay = {
  type: "action.refresh",
  id: string,
  content: Item | null,
  trigger:
    | { paramIds: string[] }
    | { intervalSeconds: number }
    | null,
  collectParamIds: string[]
}
```

Any parameters inside `trigger.paramIds` should be assumed to be included as part of `collectParamIds`.

### Helpers
The above API can be composed to create a variety of complex behaviors. Some of these behaviors are common enough that a shorthand is explicitly documented for normalization. Any conforming implementation of a Quicli server library is expected to provide all the following helpers as shorthand.

#### Registry
An `Registry` is a collection of yet-to-be-fetched Quicli results. Registries can be used as a single place to allow the user to navigate between unrelated “pages” in an app that supports many kinds of operations via Quicli.

```ts
type RegistryItem = {
  id: string,
  name: string,
  // how each language specifies an implementation
  // for registry items may differ
  implementation: () => Item,
}

RegistryHelper({
  id: string,
  name: string,
  items: RegistryItem[],
})
  =>
Refresh({
  id: "{id}-library-provided-implementation",
  content: [
    MetadataDisplay(key: "registry", value: { id, name }),
    ...items.map(i => Button(text: i.name)),
  ],
  trigger: null,
  collectParamIds: [],
})
```

When a button is clicked and a registry item is selected, the library provided implementation should replace the content of `Refresh` with the following:
```ts
(registry: Registry, registryItem: RegistryItem)
  => 
[
  MetadataDisplay(
    key: "sourceRegistry",
    value: { id: registry.id, name: registry.name },
  ),
  ...registryItem.implementation(),
]
```

Client libraries are generally expected to understand `registry` and `sourceRegistry` metadata items in a manner to provide navigation helpers, such as routing or back buttons.