mbiguous JavaScript Grammar

| Status | DRAFT |
| --- | --- |
| Authors | [@jdalton](https://github.com/jdalton)  [@bmeck](https://github.com/bmeck) |
| Date | June 14, 2016 |

## TL;DR

* Change JS grammars for Script and Module to be unambiguous / have no 
collisions.
* Determine grammar for any `.js` file by parsing as one grammar, if that fails 
parse as the other.
* Introduce a field to `package.json` mimicing the behavior of `modules.root` / 
Document Base URI to provide second entry point for Node versions that support 
ES modules.
  * Fat packages (packages that ship both CJS and ES codebases [name from [Fat 
Binary](https://en.wikipedia.org/wiki/Fat_binary)]) can keep current structure 
for legacy support and use `modules.root` to ship support for newer versions.

## Problem

The Script and Module goal of ECMA262 have a grammatical ambiguity where some 
code can run in both goals, having the exact same source, but produce different 
values. Unlike `"use strict"` the signal to have a specific behavior is not in 
the code, thus the code has a multitude of possible effects which are not 
controlled by the programmer.

### Example

```js
function format(fmt, /*args, */ cb) {
  fmt = fmt || '';
  arguments = [].slice.call(arguments);
  var cb = arguments.pop();
  $formatter.apply(this, arguments);
  cb();
}
format();
```

Differences:

 | script | module
--- | --- | --- | ---
variable scope holding format | global | local
arguments $formatter recieves | modified | unmodified
format | runs | throws
this value in format | global | `undefined`

Since there is no way in source text to enforce the goal with the current 
grammar; this leads to certain constructs being undefined behavior to the 
programmer, and defined by the host environment. In turn, existing code could 
be run in the wrong goal and partially function, or function without errors but 
produce incorrect values.


## ECMA262 Solution

Require a structure in the Module goal that does not parse in the Script goal. 
Having this requirement would prevent any source text written for the Module 
goal from being executed in the Script goal by removing the ambiguity at parse 
time.

The proposal is to require that Module source text has at least one `import` or 
`export` statement in the source text to parse.

### Script Example

```js
function format(fmt, /*args, */ cb) {
  fmt = fmt || '';
  arguments = [].slice.call(arguments);
  var cb = arguments.pop();
  $formatter.apply(this, arguments);
  cb();
}
format();
```

 | script | module (cannot parse)
--- | --- | --- | ---
variable scope holding format | global | n/a
arguments $formatter recieves | modified | n/a
format | runs | n/a
this value in format | global | n/a

### Module Example

```js
function format(fmt, /*args, */ cb) {
  fmt = fmt || '';
  arguments = [].slice.call(arguments);
  var cb = arguments.pop();
  $formatter.apply(this, arguments);
  cb();
}
format();
export default undefined;
```

 | script (cannot parse) | module
--- | --- | --- | ---
variable scope holding format | n/a | local
arguments $formatter recieves | n/a | unmodified
format | n/a | throws
this value in format | n/a | undefined

## Problem

Node currently requires a means for programmers to signal what goal their code 
is written to run in.

After much research, the only solution is a fairly hefty amount of ecosystem 
damage from either a file extension, or package.json approach. Neither of which 
define the intent of the source text in a way from the ECMA262 standard, and 
neither of which allow programmers to enforce their intent at a source text 
level like `"use strict"`.

## Solution

Parse source text as either goal, and if there is a parse error that may allow 
the other goal to parse the source text parse as the other goal. After this, 
the goal is known unambiguously and the environment can safely perform 
initialization without fear of source text being run in the wrong goal.

### Algorithm

Note: A host can choose either goal to parse as first, so feel free to swap the 
order of Script and Module here.

1. Bootstrap for Script
2. Parse as Script
3. If Success, return
4. Bootstrap for Module
5. Parse as Module
6. If Success, return
7. Throw error

## Problem

Node needs a way for people to ship both CJS and ES modules in a single package.

## Solution

Adopt the idea of `"modules.root"` from [Defense of 
.js](https://github.com/dherman/defense-of-dot-js), name pending upon some 
investigation. This would be introduced at the same time as ES modules. Any 
node version that supports this field would change the path resolution upon 
packages to resolve relative to the path defined in this field.

### Example

Given a directory structure of:

```
myapp
\- package.json
\- dist
  \- app.js # ES
\ app.js # CJS
```

And package.json containing:

```json
{
  "modules.root": "./dist",
   "main": "app.js"
}
```

```js
require('myapp');
// dist/app.js if node supports ES
// app.js if node supports CJS
```

### Side effects

This has a side effect as well, that packages would be able to limit the 
visible API of themselves via this field.

In the above example scenarios:

```js
require('myapp/package.json');
// error if node supports the field
// package.json if node is older than the field
```

This has been discussed in several  places, such as people using `react` 
relying upon internal APIs that are not to be used by package consumers.

This allows exposing a limited API for a package and having 2 distinct entry 
points for Module and Script.

### Validation of field

The field must not be the values `node_modules`, or `./node_modules`; nor may 
the field begin with `node_modules/`, `./node_modules/`, `../`, or `/` 
otherwise it will throw. This is to explicitly show the intent of the field 
relative to `package.json` and reserve other prefixes for future usage.

## Implementation

To speed up things, most host environments want to be able to accept a goal to 
parse as first in some way. This could take several forms: a command line flag, 
a manifest file, HTTP header, file extension, etc.

The recommendation for Node is that we store this in a cache on disk somewhere. 
@trevnorris and @indutny have given thoughts that this should work.

The workflow for loading any file would come out like this:

1. Get path to load as `filename`
2. If cache has `filename` set `goal` from cached value
  1. Validate cache against file for `filename`
  2. If valid
    1. Return cache
3. Else
  1. Set `goal` to a preferred goal (Script for now since most modules are CJS)
4. Load file for `filename` as `source`
5. Bootstrap source for `goal` as `bootstrapped_source`
5. Parse `bootstrapped_source` using `goal` grammar
6. If success
  1. Cache and return results
7. Else
  1. Change `goal` to opposite grammar
8. Bootstrap source for `goal` as `bootstrapped_source`
9. Parse `bootstrapped_source` using `goal` grammar
10. If success
  1. Cache and return results
11. Throw error

## Tooling concerns

Some situations outside of Node do not have a JS parser (Bash programs, some 
asset pipelines, etc.). These tools generally operate on files as opaque blobs, 
or plain text files. These tools can use

