# Unambiguous JavaScript Grammar

| Status | DRAFT |
| --- | --- |
| Authors | [@jdalton](https://github.com/jdalton)  [@bmeck](https://github.com/bmeck) |
| Date | June 14, 2016 |

## Proposal Accepted!

On July 6, 2016 this proposal was accepted as
[section 5.1](https://github.com/nodejs/node-eps/blob/master/002-es6-modules.md#51-determining-if-source-is-an-es-module)
of the Node [ES6 Module Interoperability](https://github.com/nodejs/node-eps/blob/master/002-es6-modules.md)
draft proposal :bangbang:

## TL;DR

* CJS and ES modules **just work** without new extensions, extra ceremony, or
  excessive scaffolding
* Performance is generally on par with existing CJS module loading
* Performance is significantly improved for ES modules over transpilation workflows
* Change JS grammars for Script and Module to be unambiguous
* Determine grammar of `.js` files by parsing as one grammar and falling back to others

## Problem

The Script and Module goal of ECMA262 have a grammatical ambiguity where some
code can run in both goals, having the exact same source text, but produce
different results. Unlike `"use strict"`, the signal to have a specific behavior
is not in the code, thus the code has a multitude of possible effects which are
not controlled by the programmer.

<em>Note: The following example highlights a few of the effects of ambiguous
grammar and is by no means exhaustive.</em>

### Example

```js
function foo(value) {
  value = value || '';
  var args = [].slice.call(arguments);
  args.unshift(this);
  return args;
}
foo(null);
```

Differences:

|                         | script   | module      |
| ----------------------- | -------- | ----------- |
| variable scope of `foo` | global   | local       |
| `arguments` object      | modified | unmodified  |
| `this` binding of `foo` | global   | `undefined` |

Since there is no way in source text to enforce the goal with the current grammar;
this leads to the behavior of certain constructs being undefined by the programmer,
and defined by the host environment. In turn, existing code could be run in the
wrong goal and partially function, or function without errors but produce
incorrect results.

## ECMA262 Solution

Require that Module source text has at least one `import` or `export` declaration.
A module with only an `import` declaration and no `export` declaration is valid.
Modules, that do not export anything, should specify an `export {}` to make
intentions clear and avoid accidental parse errors while removing `import`
declarations. The `export {}` is **not** new syntax and does **not** export an
empty object. It is simply the standard way to specify exporting nothing.

<em>Note: While the ES2015 specification
[does not forbid](http://www.ecma-international.org/ecma-262/6.0/#sec-forbidden-extensions)
this extension, Node wants to avoid acting as a rogue agent. Node has a TC39
representative, [@bmeck](https://github.com/bmeck), to champion this proposal.
A specification change or at least an official endorsement of this Node proposal
would be welcomed. If a resolution is not possible, this proposal will fallback
to the previous [`.mjs` file extension proposal](https://github.com/nodejs/node-eps/blob/5dae5a537c2d56fbaf23aaf2ae9da15e74474021/002-es6-modules.md#51-determining-if-source-is-an-es-module).</em>

### Script Example

```js
function foo(value) {
  value = value || '';
  var args = [].slice.call(arguments);
  args.unshift(this);
  return args;
}
foo(null);
```

|                         | script   | module (cannot parse) |
| ----------------------- | -------- | --------------------- |
| variable scope of `foo` | global   | n/a                   |
| `arguments` object      | modified | n/a                   |
| `this` binding of `foo` | global   | n/a                   |

### Module Example

```js
function foo(value) {
  value = value || '';
  var args = [].slice.call(arguments);
  args.unshift(this);
  return args;
}
foo(null);
export {};
```

|                         | script (cannot parse) | module     |
| ----------------------- | --------------------- | -----------|
| variable scope of `foo` | n/a                   | local      |
| `arguments` object      | n/a                   | unmodified |
| `this` binding of `foo` | n/a                   | undefined  |

## Problem

Node currently requires a means for programmers to signal what goal their code
is written to run in.

Leading solutions have either hefty ecosystem tolls, ceremony, or scaffolding.
They lack a way to define the intent of the source text from the ECMA262 standard.

## Solution

A package opts-in to the Module goal by specifying `"module"` as the parse goal
field *(name not final)* in its `package.json`. Package dependencies are not
affected by the opt-in and may be a mix of CJS and ES module packages. If a parse
goal is not specified, then attempt to parse source text as the preferred goal
*(Script for now since most modules are CJS)*. If there is a parse error that
may allow another goal to parse, then parse as the other goal, and so on. After
this, the goal is known unambiguously and the environment can safely perform
initialization without the possibility of the source text being run in the wrong
goal.

### Algorithm

#### Parse (source, goal, throws)

  The abstract operation to parse source text as a given goal.

  1. Bootstrap `source` for `goal`.
  2. Parse `source` as `goal`.
  3. If success, return `true`.
  4. If `throws`, throw exception.
  5. Return `false`.

#### Operation

1. If a package parse goal is specified, then
  1. Let `goal` be the resolved parse goal.
  2. Call `Parse(source, goal, true)` and return.

2. Else fallback to multiple parse.
  1. If `Parse(Source, Script, false)` is `true`, then
    1. Return.
  2. Else
    1. Call `Parse(Source, Module, true)`.

  *Note: A host can choose either goal to parse first and may change their order
  over time or as new parse goals are introduced. Feel free to swap the order of
  Script and Module.*

## Implementation

To improve performance, host environments may want to specify a goal to parse
first. This can be done in several ways:<br>
cache on disk, a command line flag, a manifest file, HTTP header, file extension, etc.

## Tooling Concerns

Some tools, outside of Node, may not have access to a JS parser *(Bash programs,
some asset pipelines, etc.)*. These tools generally operate on files as opaque
blobs / plain text files and can use the techniques, listed under
[Implementation](#implementation), to get parse goal information.

## External Examples and Impact

 * Esprima relies on a `--module` flag to signal the Module goal. However, this
   has proven to be unintuitive for many users. Unambiguous Script and Module
   goals would enable things to “just work” without flags.

 * Facebook Flow performs a series of inferences to detect CJS and ES modules.
   Unambiguous Script and Module goals would improve its ability to determine
   module types.

 * JSCS can accept input through stdin, so identification of parse goals by
   source text is ideal.

 * Linters, like xo, could use unambiguous Script and Module goals to enable
   module specific linting rules without extra configuration.

 * Microsoft packaged web applications can benefit from unambiguous Script and
   Module goals. The bytecode cache for a packaged web application is generated
   upon installation. When the application is running, files are loaded by script
   tags so their intended parse goals are understood. However, bytecode cache
   generation is done *without* running the application so the intended parse
   goals are unknown. Because of this, the bytecode cache is generated for the
   Script goal and ignored for ES modules.

 * TypeScript has taken the stance from early on that a script becomes a module
   when it has at least one `import` or `export` declaration. Over the years they
   have experienced very few user issues with this approach.

## Special Thanks

This proposal would not have been possible without the tireless effort, conviction,
and collaboration of<br>
[@bmeck](https://github.com/bmeck), [@dherman](https://github.com/dherman),
and [@wycats](https://github.com/wycats).

While iterating on this proposal we have reached out to several people from areas
affected by it.<br>
Although opinions on this proposal are varied, I am grateful for
feedback from:<br>
 * [@AtOMiCNebula](https://github.com/AtOMiCNebula) (Microsoft Edge)
 * [@brendaneich](https://github.com/brendaneich) (TC39)
 * [@bterlson](https://github.com/bterlson) (Microsoft Chakra / TC39)
 * [@chrisdickinson](https://github.com/chrisdickinson) (Node)
 * [@hzoo](https://github.com/hzoo) (Babel)
 * [@indutny](https://github.com/indutny) (Node)
 * [@leobalter](https://github.com/leobalter) (jQuery Foundation / TC39)
 * [@ljharb](https://github.com/ljharb) (TC39)
 * [@loganfsmyth](https://github.com/loganfsmyth) (Babel)
 * [@mathiasbynens](https://github.com/mathiasbynens) (Lodash)
 * [@rvagg](https://github.com/rvagg) (Node)
 * [@sheerun](https://github.com/sheerun) (Bower)
 * [@sindresorhus](https://github.com/sindresorhus) (AVA / Chalk / Yeoman)
 * [@travisleithead](https://github.com/travisleithead) (Microsoft Edge / W3C)
 * [@trevnorris](https://github.com/trevnorris) (Node)

Thank you! :heart:<br>
  &#8212; [@jdalton](https://github.com/jdalton)
