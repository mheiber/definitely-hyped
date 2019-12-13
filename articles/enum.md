# TypeScript Enum Considered Harmful: Try these alternatives

TS, like [many programming languages](https://en.wikipedia.org/wiki/Enumerated_type) provides `enum`s, which enable you to enumerate the values a type may have.

TS `enum`s look like this:

```ts
enum Direction {
  UP,
  DOWN
}

function move(direction: Direction) {}

move(Direction.UP);
```

I'll go into:

- Alternatives to enums: string unions and object literals
- How `enum` is terrible

> Note: TS is a fantastic programming language, and `enum` may have been a good choice back in 2011 when good alternatives did not exist.

## Alternative 1: String Unions

This solution uses a [union](https://www.typescriptlang.org/docs/handbook/advanced-types.html#union-types) of [string literals](https://www.typescriptlang.org/docs/handbook/advanced-types.html#string-literal-types).

```ts
type Direction = "UP" | "DOWN";
function move(direction: Direction) {
  switch (direction) {
    case "UP":
    // ...
  }
  // ...
}
```

TS gives great autocompletes for these, and gives helpful error messages if you make type errors.

Pros of the approach:

- Both the definition and use sites are readable and no-boilerplate

Cons:

- The JS output looks the same as the TS, except without the `type` declaration and `: Direction` annotation. Which is great! But it's not the kind of JS a normal JS dev would write. The next solution is classic JS style, but is a bit clunky in TS.

## Alternative 2: Object Literals

This solution is just standard JS pseudo-enums with some quirky annotations:

```ts
const DIRECTIONS = {
  UP: "UP",
  DOWN: "DOWN"
} as const;

type DIRECTIONS = typeof DIRECTIONS[keyof typeof DIRECTIONS];
```

It works!:

```ts
const d1: DIRECTIONS = DIRECTIONS.UP; // OK
const d1: DIRECTIONS = "UP";         // OK
const d2: DIRECTIONS = "PLASTICS";  // Error

const d3: DIRECTIONS =  // Autocompletes: "UP", "DOWN"
```

The solution relies on the following advanced TS features:
- [`const` assertions](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-4.html#const-assertions) to not lose the information about the specific strings in the object.
- `typeof` to get the type of a variable
- [keyof](http://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-1.html#keyof-and-lookup-types) to get a union type for the keys of the object.
- [union types](https://www.typescriptlang.org/docs/handbook/advanced-types.html#union-types)
- [lookup types](http://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-1.html#keyof-and-lookup-types) to index into an object type
- The distributivity of union types: `ObjType["A" | "B"]` is the same as `ObjType["A"] | ObjType["B"]`
- the trick that a type aliases and constants live in separate worlds, so can have the same spelling.

**Pros** of the plain objects approach:
- This solution is the most straightforward I have seen for cases where one is converting existing JS files to TS.
- Compiles away to really normal JS

**Cons**:
- The reason JS devs write things like `DIRECTIONS.UP` instead of just `"UP"` is so they don't end up with semi-hardcoded strings everywhere. But TS's autocomplete will actually fight to mitigate this advantage. Note the autocomplete in the example above is not `DIRECTIONS.UP` but the plain string `"UP"`.
- The solution might induce cargo-culting or scare people away.


# How `enum` is terrible

Issue 1: The transpile output of `enum` is super weird, which builds lock-in to TS and impairs debuggability:

```ts
enum Direction {
    Up,
    Down
}
```

outputs:

```js
var Direction;
(function (Direction) {
    Direction[Direction["Up"] = 0] = "Up";
    Direction[Direction["Down"] = 1] = "Down";
})(Direction || (Direction = {}));
```

Issue 2: enum values default to numbers (not strings), which is also bad for debuggability:

```ts
console.log(Direction.Up); // 0
```

Issue 3: Using TS runtime features means you may have to deal with breaking changes if JS evolves a similar feature with different semantics. There is already [a conflicting proposal for `enum` in JS](https://github.com/rbuckton/proposal-enum). This sort of thing has happened before:
- TS experimental decorators are incompatible with JS decorators (currently [stage 2](https://github.com/tc39/proposal-decorators)).
- Classic TS class fields [differ slightly from JS class fields](https://github.com/microsoft/TypeScript/issues/27644)

Issue 4: The [assignability rules](https://stackoverflow.com/questions/59118674/what-are-the-assignability-rules-for-enum-types-in-typescript) no longer make sense

```ts
enum A {
  foo = 0;
}

// should error, but does not error
const test: A.foo = 1729;

```

Issue 5: Declaration merging is pretty scary:

```ts
enum Shape {
    Line = 0,
}
enum Shape {
    Dot = 0, // permitted
}
const x: Shape.Dot = Shape.Line; // No error!

console.log(Shape[0]); // logs "Dot", because we added `Dot` last!
```

**TypeScript, the `Line` is a `Dot` to you!**

> Noteing again: TS is a fantastic programming language, and `enum` may have been a good choice back in 2011 when good alternatives did not exist.
