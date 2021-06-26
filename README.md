# Quicli

Quicli (pronounced like `quickly`) is a shared definition for simple transactional interfaces, such as UIs and CLIs.

## Key Concepts
A server can emit a Quicli response for consumption by a Quicli client. Servers and clients can be mixed and matched. Server libraries (under `server/`) can help consumers easily construct valid Quicli responses. Client libraries (under `client/`) can help consumers easily construct interfaces to read Quicli responses and act upon them.

A Quicli response is a list of Quicli data under of of three groups:
- **Displays** show information to users, such as help text, state from the server, or results from a recently invoked action.
- **Parameters** collect information from users, such as text, files, or lists of records.
- **Actions**  make requests against the server for further data.

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

#### Group
Any place that expects a single `Item` may also accept an array of them. In this case, they are displayed one another another.
```ts
type GroupDisplay = Item[]
```

#### List
Displays a list of items to users. Unlike `Group` a `List` may have special styling to make it clear that items in the list are semantically related. Nesting `ListDisplays` are supported up to a depth of three at minimum.
```ts
type ListDisplay = {
  type: "display.list",
  content: Item,
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

#### SmallFile
Displays a file to users. 
```ts
type SmallFileDisplay = {
  type: "display.smallFile",
  fileName: string,
  mimetype: string,
  data: string,
}
```
Note that the entire file content is transferred to the client regardless of whether they click to save it to their computer or not. The API is named `SmallFile` to reflect this constraint, and the you should avoid sending large files with a single network request.

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

#### Error
Displays an error to the user.
```ts
type ErrorDisplay = {
  type: "display.error",
  content: Item,
  severity: "error"|"warning",
}
```

#### Metadata
Metadata is not displayed to the end user but can contain any extra data that a client may find useful for logging or debugging. Metadata can be used as an escape hatch to provide platform-specific features not part of the standard specification.
```ts
type MetadataDisplay = {
  type: "display.metadata",
  key: string,
  value: any,
}
```

#### Locked
Any Parameters inside the content of a `Locked` will be displayed but uneditable. This can be used to show users the value of parameters that are set by the system, for example.
```ts
type LockedDisplay = {
  type: "display.locked",
  content: Item,
}
```

#### Hidden
Any Item inside the content of a `Hidden` will be hidden from end users, but otherwise work normally. This can be used to include extra parameters the server may need but which the user is unaware of.

The content inside of a `Hidden` is only _visually_ hidden, it is not a good place to put _secret_ information like passwords or API keys.
```ts
type HiddenDisplay = {
  type: "display.hidden",
  content: Item,
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
  name: string,
  help: string,
  initial: StringParameterValue,
  isLong: boolean,
}

type StringParameterValue = string | null
```

#### Number
Collects a single number from the user.
```ts
type NumberParameter = {
  type: "parameter.number",
  id: string,
  name: string,
  help: string,
  min: number | null,
  max: number | null,
  step: number | null,
  initial: NumberParameterValue
}

type NumberParameterValue = number | null
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
  name: string,
  help: string,
  options: ChoiceOption[],
  initial: ChoiceParameterValue,
}

type ChoiceParameterValue = string | null
```

#### MultiChoice
The user must select zero or more of the provided values.
```ts
type MultiChoiceParameter = {
  type: "parameter.multiChoice",
  id: string,
  name: string,
  help: string,
  options: ChoiceOption[],
  minCount: number | null,
  maxCount: number | null,
  initial: MultiChoiceParameterValue,
}

type MultiChoiceParameterValue = string[]
```

#### Boolean
The user must select either `true` or `false`.
```ts
type BooleanParameter = {
  type: "parameter.boolean",
  id: string,
  name: string,
  help: string,
  initial: BooleanParameterValue,
}

type BooleanParameterValue = boolean
```

#### List
The user needs to supply zero or more of the provided inputs.
```ts
type ListParameter = {
  type: "parameter.list",
  id: string,
  name: string,
  help: string,
  form: Item,
  minCount: number,
  maxCount: number | null,
}

type ListParameterValue = ParamValues[]
```
The `form` entry is repeated for each entry and defines the shape of what is collected.

####  File
The user can upload a file to the server.
```ts
type FileParameter = {
  type: "parameter.file",
  id: string,
  name: string,
  help: string,
  allowedMimeTypes: string[],
}

type FileParameterValue = {
  fileName: string,
  mimeType: string,
  data: string,
}
```

#### JSON
The user may specific any string here representing JSON. If the string is not JSON-encodable, it will be interpreted to be `null`.
```ts
type JSONParameter = {
  type: "parameter.json",
  id: string,
  name: string,
  help: string,
  initial: JSONParameterValue,
}

type JSONParameterValue = any | null
```

The `JSON` parameter is rarely a nice UX for end-users, but can be a powerful way to pass additional information when combined with the `Hidden` display.

#### Button
When a button is clicked, it executes the action it is contained within. Buttons have the value of `true` when they were what invoked the action, else `false`.
```ts
type ButtonParameter = {
  type: "parameter.button",
  id: string,
  text: string,
  isDestructive: boolean,
}

type ButtonParameterValue = boolean
```

### Action
Actions represent things you can _do_ in a Quicli response. When an action is invoked, all the parameters inside of its `body` are sent to the server alongside the action’s `id`.

#### Transaction
A `Transaction` represents a single request a user can make against a server.

When a `Transaction` is invoked, all parameters inside of `body` are collected and sent to the server for processing. The server may then respond with any valid Quicli response, which will be appended to the _end_ of the `Transaction`. In this manner, the server may display results of the action or chain together another step in a multi-part form.

```ts
type TransactionAction = {
  type: "action.transaction",
  id: string,
  body: Item,
}
```

- If there are no `Button`s inside of a `Transaction`’s `body`, then one should inferred at the end, named “Submit” (localized appropriately).

#### Refresh
A `Refresh` represents a request a user can make against a server on a regular basis, such as polling for the status of a pending job, or updating a piece of content based on the value of another.

When a `Refresh` is first loaded, and then every time it is invoked, all parameters inside of `body` are collected and sent to the server for processing. The server may then respond with any valid Quicli response, which will _replace_ the `Refresh`’s `initialContent` property. The content will be replaced with the new response each time it is invoked.

```ts
type RefreshAction = {
  type: "action.refresh",
  id: string,
  body: Item | null,
  initialContent: Item | null,
  
  triggers: {
    paramIds: string[],
    intervalSeconds: number | null,
    stopOnError: boolean,
  }
}
```

- `triggers.paramIds` can only reference params inside of `body`.
- If `triggers.stopOnError` is set, and a response contains an `Error` display anywhere in it, the Refresh should stop running automatically. A Button inside of `body` or the latest content can still invoke it manually. A successful run (containing no errors) re-enables all automatic triggers.

## Patterns
### Navigation & Routing
One Quicli server may want to serve many kinds of Quicli responses and field many kinds of requests, separated by different endpoints or HTML pages. This is a common usage, but explicitly left outside of the scope of the Quicli specification.

Quicli is about creating displays and defining actions on those displays. How those behaviors are then organized and routed to with respect to one another is up to the consumer. Many great routing libraries exist both on the backend and frontend, and using Quicli should not strong-arm you into a specific solution for that (separate) concern.

### Nesting Actions
Actions support nesting to arbitrary levels of depth, which can be quite powerful.

For example, you may have a `Transaction` that, when invoked, sends an email to the selected user. We have millions of potential users to send this email to so we want to allow our admins to search for a user as part of the form. To do so, we can nest a `Refresh` inside of the `Transaction`. The `Refresh.body` would have a `Text` parameter to search for a user, and then return as content the list of users matching that filter as options for a `Choice`. When the top-level `Transaction` is run, it will collect _all_ the parameters (including those nested inside of the inner-Refresh) and send them to our server.

An additional benefit: since the `Refresh` just provides a filter for selecting users, we can _reuse_ the endpoint fulfilling it in other actions.