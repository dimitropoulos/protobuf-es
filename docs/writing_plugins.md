Protobuf-ES: Writing Plugins
========================
Code generator plugins can be created using the npm packages [@bufbuild/protoplugin](https://npmjs.com/package/@bufbuild/protoplugin) and [@bufbuild/protobuf](https://npmjs.com/package/@bufbuild/protobuf). This is a detailed overview of the process of writing a plugin.

- [Introduction](#introduction)
- [Writing a plugin](#writing-a-plugin)
  - [Installing the plugin framework and dependencies](#installing-the-plugin-framework-and-dependencies)
  - [Setting up your plugin](#setting-up-your-plugin)
  - [Providing generator functions](#providing-generator-functions)
     - [Overriding transpilation](#overriding-transpilation) 
  - [Generating a file](#generating-a-file)
  - [Walking through the schema](#walking-through-the-schema)
  - [Printing to a generated file](#printing-to-a-generated-file)
  - [Importing](#importing)
     - [Importing from an NPM package](#importing-from-an-npm-package) 
     - [Importing from `protoc-gen-es` generated code](#importing-from-protoc-gen-es-generated-code) 
     - [Importing from the `@bufbuild/protobuf` runtime](#importing-from-the-bufbuildprotobuf-runtime)
     - [Type-only imports](#type-only-imports)
     - [Why use `f.import()`?](#why-use-fimport)
  - [Exporting](#exporting)
  - [Parsing plugin options](#parsing-plugin-options)
  - [Reading custom options](#reading-custom-options)
     - [Scalar options](#scalar-options) 
     - [Message options](#message-options) 
     - [Enum options](#enum-options) 
- [Testing](#testing)
- [Examples](#examples)

## Introduction

Code generator plugins are a unique feature of protocol buffer compilers like protoc and the [buf CLI](https://docs.buf.build/introduction#the-buf-cli).  With a plugin, you can generate files based on Protobuf schemas as the input.  Outputs such as RPC clients and server stubs, mappings from protobuf to SQL, validation code, and pretty much anything else you can think of can all be produced.

The contract between the protobuf compiler and a code generator plugin is defined in [plugin.proto](https://github.com/protocolbuffers/protobuf/blob/main/src/google/protobuf/compiler/plugin.proto). Plugins are simple executables (typically on your `$PATH`) that are named `protoc-gen-x`, where `x` is the name of the language or feature that the plugin provides. The protobuf compiler parses the protobuf files, and invokes the plugin, sending a `CodeGeneratorRequest` on standard in, and expecting a `CodeGeneratorResponse` on standard out. The request contains a set of descriptors (see [descriptor.proto](https://github.com/protocolbuffers/protobuf/blob/main/src/google/protobuf/descriptor.proto)) - an abstract version of the parsed protobuf files. The response contains a list of files, each having a name and text content.

For more information on how plugins work, check out [our documentation](https://docs.buf.build/reference/images). 

## Writing a plugin

The following will describe the steps and things to know when writing your own code generator plugin.  The main step in the process is passing a plugin initialization object to the `createEcmaScriptPlugin` function exported by the plugin framework.  This plugin initalization object will contain various properties pertaining to different aspects of your plugin.

### Installing the plugin framework and dependencies

The main dependencies for writing plugins are the main plugin package at [@bufbuild/protoplugin](https://npmjs.com/package/@bufbuild/protoplugin) and the runtime API at [@bufbuild/protobuf](https://npmjs.com/package/@bufbuild/protobuf).  Using your package manager of choice, install the above packages:

**npm**
```bash
npm install @bufbuild/protoplugin @bufbuild/protobuf
```

**pnpm**
```bash
pnpm install @bufbuild/protoplugin @bufbuild/protobuf
```

**Yarn**
```bash
yarn add @bufbuild/protoplugin @bufbuild/protobuf
```

### Setting up your plugin

The first thing to determine for your plugin is the `name` and `version`.  These are both passed as properties on the plugin initialization object.  

The `name` property denotes the name of your plugin.  Most plugins are prefixed with `protoc-gen` as required by `protoc` i.e. [protoc-gen-es](https://github.com/bufbuild/protobuf-es/tree/main/packages/protoc-gen-es).

The `version` property is the semantic version number of your plugin.  Typically, this should mirror the version specified in your package.json.

The above values will be placed into the preamble of generated code, which provides an easy way to determine the plugin and version that was used to generate a specific file.

For example, with a `name` of **protoc-gen-foo** and `version` of **v0.1.0**, the following will be added to generated files:

```ts
 // @generated by protoc-gen-foo v0.1.0 with parameter "target=ts"
 ```


### Providing generator functions

Generator functions are functions that are used to generate the actual file content parsed from protobuf files.  There are three that can be implemented, corresponding to the three possible target outputs for plugins:

| Target Out | Function |
| :--- | :--- |
| `ts` | `generateTs(schema: Schema): void` |
| `js` | `generateJs(schema: Schema): void` |
| `dts` | `generateDts(schema: Schema): void` |

Of the three, only `generateTs` is required.  These functions will be passed as part of your plugin initialization and as the plugin runs, the framework will invoke the functions depending on which target outputs were specified by the plugin consumer.  

Since `generateJs` and `generateDts` are both optional, if they are not provided, the plugin framework will attempt to transpile your generated TypeScript files to generate any desired `js` or `dts` outputs if necessary.

In most cases, implementing the `generateTs` function only and letting the plugin framework transpile any additionally required files should be sufficient.  However, the transpilation process is somewhat expensive and if plugin performance is a concern, then it is recommended to implement `generateJs` and `generateDts` functions also as the generation processing is much faster than transpilation.

#### Overriding transpilation

As mentioned, if you do not provide a `generateJs` and/or a `generateDts` function and either `js` or `dts` is specified as a target out, the plugin framework will use its own TypeScript compiler to generate these files for you.  This process uses a stable version of TypeScript with lenient compiler options so that files are generated under most conditions.  However, if this is not sufficient, you also have the option of providing your own `transpile` function, which can be used to override the plugin framework's transpilation process.  

```ts
transpile(fileInfo: FileInfo[], transpileJs: boolean, transpileDts: boolean): FileInfo[]
```

The function will be invoked with an array of `FileInfo` objects representing the TypeScript file content
to use for transpilation as well as two booleans indicating whether the function should transpile JavaScript,
declaration files, or both.  It should return a list of `FileInfo` objects representing the transpiled content.

**NOTE**:  The `transpile` function is meant to be used in place of either `generateJs`, `generateDts`, or both.  
However, those functions will take precedence.  This means that if `generateJs`, `generateDts`, and 
`transpile` are all provided, `transpile` will be ignored.

A sample invocation of `createEcmaScriptPlugin` after the above steps will look similar to:

```ts
export const protocGenFoo = createEcmaScriptPlugin({
   name: "protoc-gen-foo",
   version: "v0.1.0",
   generateTs,
});
```

### Generating a file

As illustrated above, the generator functions are invoked by the plugin framework with a parameter of type `Schema`.  This object contains the information needed to generate code.  In addition to the [`CodeGeneratorRequest`](https://github.com/protocolbuffers/protobuf/blob/main/src/google/protobuf/compiler/plugin.proto) that is standard when working with protoc plugins, the `Schema` object also contains some convenience interfaces that make it a bit easier to work with the various descriptor objects.  See [Walking through the schema](#walking-through-the-schema) for more information on the structure. 

For example, the `Schema` object contains a `files` property, which is a list of `DescFile` objects representing the files requested to be generated.  The first thing you will most likely do in your generator function is iterate over this list and issue a call to a function that is also present on the `Schema` object: `generateFile`.  This function expects a filename and returns a generated file object containing a `print` function which you can then use to "print" to the file.  

Each `file` object on the schema contains a `name` property representing the name of the file that was parsed by the compiler (minus the `.proto` extension).  When specifying the filename to pass to `generateFile`, it is recommended to use this file name plus the name of your plugin (minus `protoc-gen`).  So, for example, for a file named `user_service.proto` being processed by `protoc-gen-foo`,  the value passed to `generateFile` would be `user_service_foo.ts`.

A more detailed example:

```ts
function generateTs(schema: Schema) {
   for (const file of schema.files) {
     const f = schema.generateFile(file.name + "_foo.ts");
     ...
   }
}
```


### Walking through the schema

The `Schema` object contains the hierarchy of the grammar contained within a Protobuf file.  The plugin framework uses its own interfaces that mostly correspond to the `DescriptorProto` objects representing the various elements of Protobuf (messages, enums, services, methods, etc.).  Each of the framework interfaces is prefixed with `Desc`, i.e. `DescMessage`, `DescEnum`, `DescService`, `DescMethod`.

The hierarchy starts with `DescFile`, which contains all the nested `Desc` types necessary to begin generating code.  For example:

```ts
for (const file of schema.files) {
  // file is type DescFile
  
  for (const enumeration of file.enums) {
     // enumeration is type DescEnum
  }
  
  for (const message of file.messages) {
     // message is type DescMessage
  }
     
  for (const service of file.services) {
     // service is type DescService
     
     for (const method of service.methods) {
         // method is type DescMethod
     }
  }
}
```

### Printing to a generated file
 
As mentioned, the object returned from `generateFile` contains a `print` function which is a variadic function accepting zero-to-many string values.  These values will then be "printed" to the file so that when the actual physical file is generated by the compiler, all values given to `print` will be included in the file.  Successive strings passed in the same invocation will be appended to one another.  To print an empty line, pass zero arguments to `print`.
 
For example:

```ts
const name = "UserService";
f.print("export class ", name, "Client {");     
f.print();
f.print("}");
```

The above will generate:

```ts
export class UserServiceClient {

}
```

Putting all of the above together for a simple example:
  
  ```ts
function generateTs(schema: Schema) {
   for (const file of schema.files) {
  
     for (const enumeration of file.enums) {
        f.print("// generating enums from ", file.name);
        f.print();
        ...
     }
  
     for (const message of file.messages) {
        f.print("// generating messages from ", file.name);
        f.print();
        ...
     }
     
     for (const service of file.services) {
        f.print("// generating services from ", file.name);
        f.print();
        for (const method of service.methods) {
            f.print("generating methods for service ", service.name);
            f.print();
            ...
        }
     }
   }
}
```

**NOTE**:  Messages can be recursive structures, containing other message and enum definitions.  The example above does not illustrate generating _all_ possible messages in a `Schema` object.  It has been simplified for brevity.

### Importing

Generating import statements is accomplished via a combination of the `print` function and another function on the generated file object:  `import`.  The approach varies depending on the type of import you would like to generate.  

#### Importing from an NPM package

To generate an import statement from an NPM package dependency, you first invoke the `import` function, passing the name of the import and the package in which it is located.

For example, to import the `useEffect` hook from React:

```ts
const useEffect = f.import("useEffect", "react");
```

This will return you an object of type `ImportSymbol`.  This object can then be used in your generation code with the `print` function:

```ts
f.print(useEffect, "(() => {");
f.print("    document.title = `You clicked ${count} times`;
f.print("});");
```

When the `ImportSymbol` is printed (and only when it is printed), an import statement will be automatically generated for you:

`import { useEffect } from 'react';`

#### Importing from `protoc-gen-es` generated code 

Imports in this way work similarly.  Again, the `print` statement will automatically generate the import statement for you when invoked.

```ts
declare var someMessageDescriptor: DescMessage;
const someMessage = f.import(someMessageDescriptor);
f.print('const msg = new ', someMessage,'();');
```

There is also a shortcut in `print` which does the above for you:

```ts
f.print('const msg = new ', someMessageDescriptor,'();');
```

#### Importing from the @bufbuild/protobuf runtime

The `Schema` object contains a `runtime` property which provides an `ImportSymbol` for all important types as a convenience:

```ts
const { JsonValue } = schema.runtime;
f.print('const j: ', JsonValue, ' = "hello";');
```

#### Type-only imports

If you would like the printing of your `ImportSymbol` to generate a type-only import, then you can convert it using the `toTypeOnly()` function:

```ts
const { Message } = schema.runtime;
const MessageAsType = Message.toTypeOnly();
f.print("isFoo<T extends ", MessageAsType, "<T>>(data: T): bool {");
f.print("   return true;");
f.print("}");
```

This will instead generate the following import:

```ts
import type { Message } from "@bufbuild/protobuf";
```

This is useful when `importsNotUsedAsValues` is set to `error` in your tsconfig, which will not allow you to use a plain import if that import is never used as a value.  

Note that some of the `ImportSymbol` types in the schema runtime (such as `JsonValue`) are type-only imports by default since they cannot be used as a value.  Most, though, can be used as both and will default to a regular import.

#### Why use `f.import()`?

The natural instinct would be to simply print your own import statements as `f.print("import { Foo } from 'bar'")`, but this is not the recommended approach.  Using `f.import()` has many advantages such as:

- **Conditional imports**
   - Import statements belong at the top of a file, but you usually only find out later whether you need the import, such as further in your code in a nested if statement.  Conditionally printing the import symbol will only generate the import statement when it is actually used.
   
- **Preventing name collisions**
    - For example if you `import { Foo } from "bar"`  and `import { Foo } from "baz"` , `f.import()` will automatically rename one of them `Foo$1`, preventing name collisions in your import statements and code.

- **Extensibility of import generation**
    - Abstracting the generation of imports allows the library to potentially offer other import styles in the future without affecting current users.
  

### Exporting

Working with exports is accomplished via the `export` function on the generated file object.  Let's walk through an example:

Suppose you generate a validation function for every message.  If you have a nested message, such as:

```proto
message Bar {
   Foo foo = 1;
}
```

You may want to import and use the validation function generated for message `Foo` when generating the code for message `Bar`.  To generate the validation function, you would use `export` as follows:

```ts
const fn = f.export("validateFoo");
f.print("function ", fn, "() {");
f.print("   return true;");
f.print("}");
```

Note that `export` returns an `ImportSymbol` that can then be used by another dependency.  The trick is to store this `ImportSymbol` and use it when you generate the validation function for `Bar`.  Storing the symbol is as simple as putting it in a global map:

```ts
const exportMap = new Map<DescMessage, ImportSymbol>()
```

That way, when you need to use it for `Bar`, you can simply access the map:

```ts
const fooValidationFn = exportMap.get(bar);  // bar is of type DescMessage
```

### Parsing plugin options

The plugin framework recognizes a set of pre-defined key/value pairs that can be passed to all plugins when executed (i.e. `target`, `keep_empty_files`, etc.), but if your plugin needs to be passed additional parameters, you can specify a `parseOption` function as part of your plugin initialization.  

```ts
parseOption(key: string, value: string | undefined): void;
```

This function will be invoked by the framework, passing in any key/value pairs that it does not recognize from its pre-defined list.

### Reading custom options

Protobuf-ES does not yet provide support for extensions, neither in general as pertaining to proto2 nor with custom options in proto3.  However, in the interim, there are convenience functions for retrieving any custom options specified in your .proto files.  These are provided as a temporary utility until full extension support is implemented.  There are three functions depending on the structure of the custom option desired (scalar, message, or enum):

#### Scalar Options

Custom options of a scalar type (`boolean`, `string`, `int32`, etc.) can be retrieved via the `findCustomScalarOption` function.  It returns a type corresponding to the given `scalarType` parameter.  For example, if `ScalarType.STRING` is passed, the return type will be a `string`.  If the option is not found, it returns `undefined`.

```ts
function findCustomScalarOption<T extends ScalarType>(
  desc: AnyDesc,
  extensionNumber: number,
  scalarType: T
): ScalarValue<T> | undefined;
```

`AnyDesc` represents any of the `DescXXX` objects such as `DescFile`, `DescEnum`, `DescMessage`, etc.  The `extensionNumber` parameter represents the extension number of the custom options field definition.

The `scalarType` parameter is the type of the custom option you are searching for.  `ScalarType` is an enum that represents all possible scalar types in the Protobuf grammar

For example, given the following:

```proto
extend google.protobuf.MessageOptions {
  optional int32 foo_message_option = 50001;
}
extend google.protobuf.FieldOptions {
  optional string foo_field_option = 50002;
}

message FooMessage {
  option (foo_message_option) = 1234;

  int32 foo = 1 [(foo_field_option) = "test"];
}
```

The values of these options can be retrieved as follows:

```ts
const msgVal = findCustomScalarOption(descMessage, 50001, ScalarType.INT32);  // 1234

const fieldVal = findCustomScalarOption(descField, 50002, ScalarType.STRING);  // "test"
```

#### Message Options

Custom options of a more complex message type can be retrieved via the `findCustomMessageOption` function.  It returns a concrete type with fields populated corresponding to the values set in the proto file.

```ts
export function findCustomMessageOption<T extends Message<T>>(
  desc: AnyDesc,
  extensionNumber: number,
  msgType: MessageType<T>
): T | undefined {
```

The `msgType` parameter represents the type of the message you are searching for.  

For example, given the following proto files:

```proto
// custom_options.proto

extend google.protobuf.MethodOptions {
  optional ServiceOptions service_method_option = 50007;
}

message ServiceOptions {
  int32 foo = 1;
  string bar = 2;
  oneof qux {
    string quux = 3;
  }
  repeated string many = 4;
  map<string, string> mapping = 5;
}
```

```proto
// service.proto

import "custom_options.proto";

service FooService {
  rpc Get(GetRequest) returns (GetResponse) {
    option (service_method_option) = { 
        foo: 567, 
        bar: "Some string", 
        quux: "Oneof string",
        many: ["a", "b", "c"],
        mapping: [{key: "testKey", value: "testVal"}]
    };
  }
}
```

You can retrieve the options using a generated type by first generating the file which defines the custom option type.  Then, import and pass this type to the `findCustomMessageOption` function.

```ts
import { ServiceOptions } from "./gen/proto/custom_options_pb.js";

const option = findCustomMessageOption(method, 50007, ServiceOptions)

console.log(option);
/*
 * {
 *     foo: 567, 
 *     bar: "Some string", 
 *     quux: "Oneof string",
 *     many: ["a", "b", "c"],
 *     mapping: [{key: "testKey", value: "testVal"}]
 * }
 */
 ```

Note that `repeated` and `map` values are only supported within a custom message option.  They are not supported as option types independently.  If you have need to use a custom option that is `repeated` or is of type `map`, it is recommended to use a message option as a wrapper.


#### Enum Options

Custom options of an enum type can be retrieved via the `findCustomEnumOption` function.  It returns a `number` corresponding to the `enum` value set in the option.

```ts
export function findCustomEnumOption(
  desc: AnyDesc,
  extensionNumber: number
): number | undefined {
```

The returned number can then be coerced into the concrete `enum` type.  The `enum` type just needs to be generated ahead of time much like the example in `findCustomMessageOption`.

For example, given the following:

```proto
extend google.protobuf.MessageOptions {
  optional FooEnum foo_enum_option = 50001;
}

enum FooEnum {
  UNDEFINED = 0;
  ACTIVE = 1;
  INACTIVE = 2;
}

message FooMessage {
  option (foo_enum_option) = ACTIVE;

  string name = 1;
}
```

The value of this option can be retrieved as follows:

```ts
const enumVal: FooEnum | undefined = findCustomEnumOption(descMessage, 50001);  // FooEnum.ACTIVE
```

## Testing

There is no specific formula for how to test an individual plugin.  The official [protoc-gen-es](../packages/protoc-gen-es) plugin is extensively tested and could provide some guidance.  In addition, there are examples of testing the framework in the [protoplugin-test package](../packages/protoplugin-test). 

A helpful suggestion is to generate specific use cases that are expected for your plugin and then test that the output is what is expected.  It is a bit difficult to test discrete functionality so verifying the output is valid is the recommended approach.  To test the transpilation process specifically, it may be helpful to generate your own JavaScript and declaration files and then verify that they match transpilation.  


## Examples

For a small example of generating a Twirp client based on a simple service definition, take a look at [protoplugin-example](https://github.com/bufbuild/protobuf-es/tree/main/packages/protoplugin-example).

Additionally, check out [protoc-gen-es](https://github.com/bufbuild/protobuf-es/tree/main/packages/protoc-gen-es), which is the official code generator for Protobuf-ES.
