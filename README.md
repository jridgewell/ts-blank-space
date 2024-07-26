# ts-blank-space

TypeScript goes in:

```typescript
class C /**/< T >/*︎*/ extends Array/**/<T> /*︎*/implements I,J/*︎*/ {
//          ^^^^^                      ^^^     ^^^^^^^^^^^^^^
    readonly field/**/: string/**/ = "";
//  ^^^^^^^^          ^^^^^^^^
    static accessor f1;
    private f2/**/!/**/: string/*︎*/;
//  ^^^^^^^       ^    ^^^^^^^^

    method/**/<T>/*︎*/(/*︎*/this: T,/**/ a? /*︎*/: string/**/)/*︎*/: void/*︎*/ {
//            ^^^         ^^^^^^^^      ^     ^^^^^^^^         ^^^^^^
    }
}
```

JavaScript + space comes out:

```javascript
class C /**/     /*︎*/ extends Array/**/    /*︎*/              /*︎*/ {
//          ^^^^^                      ^^^     ^^^^^^^^^^^^^^
             field/**/        /**/ = "";
//  ^^^^^^^^          ^^^^^^^^
    static accessor f1;
            f2/**/ /**/        /*︎*/;
//  ^^^^^^^       ^    ^^^^^^^^

    method/**/   /*︎*/(/*︎*/        /**/ a  /*︎*/        /**/)/*︎*/      /*︎*/ {
//            ^^^         ^^^^^^^^      ^     ^^^^^^^^         ^^^^^^
    }
}
```

## API

### String to String

```typescript
export default function tsBlankSpace(
    ts: string,
    onError?: (node) => void
): string;
```

```javascript
import tsBlankSpace from "ts-blank-space";

console.log(tsBlankSpace(`let x: string;`));
// "let x        ;"
```

### Bring your own AST

```typescript
export function blankSourceFile(
    ts: TS.SourceFile,
    onError?: (node) => void
): string
```

```javascript
import ts from "typescript";
import { blankSourceFile } from "ts-blank-space";

const ast = ts.createSourceFile(...);
console.log(blankSourceFile(ast));
```

## Where are my SourceMaps?

Because all the JavaScript in the output is located at the same line and column as the original
there is no mapping information that is lost during the transform.

## Why?

The benefits of this library are:

- It is fast (for a pure JavaScript transform). See the `./perf` folder
  - No new JavaScript needs to be emitted from an AST, it re-uses slices of the existing source string
  - This is particularly true if other parts of your program are already generating the TypeScript SourceFile object for other reasons because it can [be reused](#bring-your-own-ast), and producing the AST is the most time consuming part.
- It is small (less than 900 LOC), by doing so little the code should be easy to understand and maintain
- No need for additional SourceMap processing. See ["where are my SourceMaps?"](#where-are-my-sourcemaps)

## Does it really just blank out all the type annotations?

There are two cases, described here, where it does more than replace the TypeScript syntax with blank space.

### ASI (automatic semicolon insertion)

To guard against [ASI](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Lexical_grammar#automatic_semicolon_insertion) issues in the output `ts-blank-space` will add `;` to the end of statements that have omitted it.

Example input:

```typescript
statementWithNoSemiColon
type Erased = true
("not calling above statement")
```

becomes:

```javascript
statementWithNoSemiColon
;
("not calling above statement");
```

### Arrow function return types that introduce a new line

If the annotation marking the return type of an arrow function introduces a new line before the `=>`
then only replacing it with blank space would be incorrect.
So in addition to removing the type annotation, the `)` is moved down to the end of the type annotation.

Example input:

```typescript
let f = (a: string, b: string): Array<
   string
> => [a, b];
```

becomes:

```javascript
let f = (a        , b

) => [a, b];
```

## Unsupported

Some parts of TypeScript are not supported because they can't be erased in place due to having
runtime semantics.

- `enum` (unless `declare enum`)
- `namespace` (unless `declare namespace`)
- `module` (unless `declare module`)
- `import lib = ...`, `export = ...` (TypeScript style CommonJS)
- `constructor(public x) {}` (parameter properties in class constructors)
- `<Type>val` (code will need to use `val as Type` instead)

When any of the above are encountered `ts-blank-space` will call the optional `onError` callback and continue.
Examples can be seen in [`errors.test.js`](./tests/errors.test.js).

## Recommend `tsconfig.json` compiler settings

```jsonc
{
    // Because class fields are preserved as written which corresponds
    // to 'define' semantics in the ECMAScript specification
    "useDefineAsClassFields": true,
    // Because imports and exports are preserved as written, only removing the
    // parts which are explicitly annotated with the `type` keyword
    "verbatimModuleSyntax": true
}
```

## TSX/JSX

JSX is not transformed, it will be preserved in the output.

By default `ts-blank-space` will parse the file assuming `.ts`. If the original file contains JSX syntax
then the parsing should be done manually. There is a TSX example in [`valid.test.js`](./tests/valid.test.js).
