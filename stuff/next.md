# Non-Regrettable Enums in TypeScript

Many languages have `enum`s, which enable you to enumerate the values a type may have.

TS provides enums that look OK on the surface but are actually evil:

```ts
enum Direction {
  UP,
  DOWN
}
function move(direction: Direction) {}
move(Direction.UP);
```

I'll go into:

- Alternatives to enums: String unions or plain objects
- How `enum` is terrible

This is all because I try to use TS in a way that has:

- low lock-in: can switch back to JS if we have to
- low risk of conflicting with future JS features. Enums are high-risk.

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

## Alternative 2: Plain Objects

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
const d1: DIRECTIONS = DIRECTIONS.UP;  // OK
const d1: DIRECTIONS = "UP";           // OK
const d2: DIRECTIONS = "PLASTICS"      // Error

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

Reason 1: The transpile output of `enum` is super weird, which builds lock-in to TS and impairs debuggability:

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

Reason 2: enum values default to numbers (not strings), which is also bad for debuggability:

```ts
console.log(Direction.Up); // 0
```

Reason 3: Declaration merging is pretty scary:

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

TypeScript, the `Line` is a `Dot` to you!

Reason 4: Even if `enum` were not fundamentally broken, it's risky to use it. Using non-type-system TS language additions means you may have to deal with breaking changes if JS evolves a similar feature with different semantics. There is already [a proposal for `enum` in JS](https://github.com/rbuckton/proposal-enum). Cases where this sort of thing has happened before:
- TS experimental decorators are incompatible with JS decorators (currently [stage 2](https://github.com/tc39/proposal-decorators)).
- Classic TS class fields [differed slightly from JS class fields](https://github.com/microsoft/TypeScript/issues/27644)
- CoffeeScript 2 has [changes in runtime behavior](https://coffeescript.org/#breaking-changes-default-values) and [missing features](https://coffeescript.org/#unsupported-let-const) due to JS evolving in a direction incompatible with CoffeeScript.

------

ways of doing enums

tsconfig extends

taking apart types and putting them back together: safify

- include type-level tests, but just do assignment
- maybe use unknown

Branding

- Brand checks \_\_String

CustomPromisify

ts-mockito

TypedEmitter

```ts
type BuildEvent = {
  completed: BuildResult;
  debug: { message: string };
  log: { message: string };
  warn: { message: string };
};

interface TypedEmitter<E> {
  on<K extends keyof E>(eventType: K, listener: (event: E[K]) => any): void;
  off(): void;
}
```

tryIt
https://www.typescriptlang.org/play/index.html#src=%2F%2F%20compare%20%60demo%60%20to%20%60oldSchool%60%0D%0A%0D%0Aconst%20demo%20%3D%20tryIt(function*%20()%20%7B%0D%0A%20%20%20%20const%20x%20%3D%20yield%20safeSum(1%2C%201)%3B%20%2F%2F%20OK%0D%0A%20%20%20%20console.log(x)%3B%0D%0A%20%20%20%20%2F%2F%20tryIt%20automatically%20returns%20the%20error%0D%0A%20%20%20%20const%20y%20%3D%20yield%20safeDiv(x%2C%200)%3B%0D%0A%0D%0A%20%20%20%20%2F%2F%20not%20reached%0D%0A%20%20%20%20return%2050%3B%0D%0A%7D)%0D%0A%0D%0Aconst%20oldSchool%20%3D%20function%20()%3A%20number%20%7C%20Error%20%7B%0D%0A%20%20%20%20const%20x%20%3D%20safeSum(1%2C%201)%3B%0D%0A%20%20%20%20if%20(x%20instanceof%20Error)%20%7B%0D%0A%20%20%20%20%20%20%20%20return%20x%3B%0D%0A%20%20%20%20%7D%0D%0A%20%20%20%20const%20y%20%3D%20safeDiv(x%2C%200)%3B%0D%0A%20%20%20%20if%20(y%20instanceof%20Error)%20%7B%0D%0A%20%20%20%20%20%20%20%20return%20y%3B%0D%0A%20%20%20%20%7D%0D%0A%20%20%20%20return%2050%3B%0D%0A%7D%0D%0A%0D%0A%0D%0A%2F%2F%20this%20could%20be%20enriched%20to%20automatically%20safify%0D%0Afunction%20tryIt%3CA%20extends%20unknown%5B%5D%2C%20T%3E(gen%3A%20(...args%3A%20A)%20%3D%3E%20IterableIterator%3CT%3E)%3A%20(...args%3A%20A)%20%3D%3E%20T%20%7C%20Error%20%7B%0D%0A%20%20%20%20return%20(...args%3A%20A)%20%3D%3E%20%7B%0D%0A%20%20%20%20%20%20%20%20const%20it%20%3D%20gen(...args)%0D%0A%20%20%20%20%20%20%20%20let%20res%3A%20any%3B%0D%0A%20%20%20%20%20%20%20%20while%20(res%20%3D%20it.next())%20%7B%0D%0A%20%20%20%20%20%20%20%20%20%20%20%20const%20%7B%20value%20%7D%20%3D%20it.next()%3B%0D%0A%20%20%20%20%20%20%20%20%20%20%20%20if%20(value%20instanceof%20Error)%20%7B%0D%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20return%20value%3B%0D%0A%20%20%20%20%20%20%20%20%20%20%20%20%7D%0D%0A%20%20%20%20%20%20%20%20%20%20%20%20return%20it.next(value).value%0D%0A%20%20%20%20%20%20%20%20%7D%0D%0A%20%20%20%20%7D%0D%0A%7D%0D%0A%0D%0A%0D%0A%0D%0Aconsole.log(demo())%0D%0A%0D%0Afunction%20safeSum(a%3A%20number%2C%20b%3A%20number)%3A%20number%20%7C%20Error%20%7B%0D%0A%20%20%20%20return%20a%20%2B%20b%0D%0A%7D%0D%0A%0D%0Afunction%20safeDiv(dividend%3A%20number%2C%20divisor%3A%20number)%3A%20number%20%7C%20Error%20%7B%0D%0A%20%20%20%20if%20(divisor%20%3D%3D%3D%200)%20%7B%0D%0A%20%20%20%20%20%20%20%20return%20Error('divide%20by%200')%0D%0A%20%20%20%20%7D%0D%0A%20%20%20%20return%20dividend%20%2F%20divisor%0D%0A%20%20%20%20%0D%0A%7D%0D%0A

function mapMap<K1, V1, K2, V2>(m: Map<K1, V1>, f: (arg: [K1, V1]) => ([K2, V2])): Map<K2, V2> {
return new Map(Array.from(m.entries()).map(f));
}

function tuple<T extends any[]>(...args: T): T {
return args;
}
