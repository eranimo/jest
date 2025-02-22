# jest-diff

Display differences clearly so people can review changes confidently.

The default export serializes JavaScript **values**, compares them line-by-line, and returns a string which includes comparison lines.

Two named exports compare **strings** character-by-character:

- `diffStringsUnified` returns a string.
- `diffStringsRaw` returns an array of `Diff` objects.

Three named exports compare **arrays of strings** line-by-line:

- `diffLinesUnified` and `diffLinesUnified2` return a string.
- `diffLinesRaw` returns an array of `Diff` objects.

## Installation

To add this package as a dependency of a project, run either of the following commands:

- `npm install jest-diff`
- `yarn add jest-diff`

## Usage of default export

Given JavaScript **values**, `diffDefault(a, b, options?)` does the following:

1. **serialize** the values as strings using the `pretty-format` package
2. **compare** the strings line-by-line using the `diff-sequences` package
3. **format** the changed or common lines using the `chalk` package

To use this function, write either of the following:

- `const diffDefault = require('jest-diff').default;` in CommonJS modules
- `import diffDefault from 'jest-diff';` in ECMAScript modules

### Example of default export

```js
const a = ['delete', 'common', 'changed from'];
const b = ['common', 'changed to', 'insert'];

const difference = diffDefault(a, b);
```

The returned **string** consists of:

- annotation lines: describe the two change indicators with labels, and a blank line
- comparison lines: similar to “unified” view on GitHub, but `Expected` lines are green, `Received` lines are red, and common lines are dim (by default, see Options)

```diff
- Expected
+ Received

  Array [
-   "delete",
    "common",
-   "changed from",
+   "changed to",
+   "insert",
  ]
```

### Edge cases of default export

Here are edge cases for the return value:

- `' Comparing two different types of values. …'` if the arguments have **different types** according to the `jest-get-type` package (instances of different classes have the same `'object'` type)
- `'Compared values have no visual difference.'` if the arguments have either **referential identity** according to `Object.is` method or **same serialization** according to the `pretty-format` package
- `null` if either argument is a so-called **asymmetric matcher** in Jasmine or Jest

## Usage of diffStringsUnified

Given **strings**, `diffStringsUnified(a, b, options?)` does the following:

1. **compare** the strings character-by-character using the `diff-sequences` package
2. **clean up** small (often coincidental) common substrings, also known as chaff
3. **format** the changed or common lines using the `chalk` package

Although the function is mainly for **multiline** strings, it compares any strings.

Write either of the following:

- `const {diffStringsUnified} = require('jest-diff');` in CommonJS modules
- `import {diffStringsUnified} from 'jest-diff';` in ECMAScript modules

### Example of diffStringsUnified

```js
const a = 'common\nchanged from';
const b = 'common\nchanged to';

const difference = diffStringsUnified(a, b);
```

The returned **string** consists of:

- annotation lines: describe the two change indicators with labels, and a blank line
- comparison lines: similar to “unified” view on GitHub, and **changed substrings** have **inverse** foreground and background colors (that is, `from` has white-on-green and `to` has white-on-red, which the following example does not show)

```diff
- Expected
+ Received

  common
- changed from
+ changed to
```

### Performance of diffStringsUnified

To get the benefit of **changed substrings** within the comparison lines, a character-by-character comparison has a higher computational cost (in time and space) than a line-by-line comparison.

If the input strings can have **arbitrary length**, we recommend that the calling code set a limit, beyond which splits the strings, and then calls `diffLinesUnified` instead. For example, Jest falls back to line-by-line comparison if either string has length greater than 20K characters.

## Usage of diffLinesUnified

Given **arrays of strings**, `diffLinesUnified(aLines, bLines, options?)` does the following:

1. **compare** the arrays line-by-line using the `diff-sequences` package
2. **format** the changed or common lines using the `chalk` package

You might call this function when strings have been split into lines and you do not need to see changed substrings within lines.

### Example of diffLinesUnified

```js
const aLines = ['delete', 'common', 'changed from'];
const bLines = ['common', 'changed to', 'insert'];

const difference = diffLinesUnified(aLines, bLines);
```

```diff
- Expected
+ Received

- delete
  common
- changed from
+ changed to
+ insert
```

### Edge cases of diffLinesUnified or diffStringsUnified

Here are edge cases for arguments and return values:

- both `a` and `b` are empty strings: no comparison lines
- only `a` is empty string: all comparison lines have `bColor` and `bIndicator` (see Options)
- only `b` is empty string: all comparison lines have `aColor` and `aIndicator` (see Options)
- `a` and `b` are equal non-empty strings: all comparison lines have `commonColor` and `commonIndicator` (see Options)

To get the comparison lines described above from the `diffLineUnified` function, call `splitLines0(string)` instead of `string.split('\n')`

```js
export const splitLines0 = string =>
  string.length === 0 ? [] : string.split('\n');
```

### Example of splitLines0 function

```js
import {diffLinesUnified, splitLines0} from 'jest-diff';

const a = 'multi\nline\nstring';
const b = '';
const options = {includeChangeCounts: true}; // see Options

const difference = diffLinesUnified(splitLines0(a), splitLines0(b), options);
```

Given an empty string, `splitLines0(b)` returns `[]` an empty array, formatted as no `Received` lines:

```diff
- Expected 3
+ Received 0

- multi
- line
- string
```

### Example of split method

```js
const a = 'multi\nline\nstring';
const b = '';
const options = {includeChangeCounts: true}; // see Options

const difference = diffLinesUnified(a.split('\n'), b.split('\n'), options);
```

Given an empty string, `b.split('\n')` returns `['']` an array that contains an empty string, formatted as one empty `Received` line, which is **ambiguous** with an empty line:

```diff
- Expected 3
+ Received 1

- multi
- line
- string
+
```

## Usage of diffLinesUnified2

Given two **pairs** of arrays of strings, `diffLinesUnified2(aLinesDisplay, bLinesDisplay, aLinesCompare, bLinesCompare, options?)` does the following:

1. **compare** the pair of `Compare` arrays line-by-line using the `diff-sequences` package
2. **format** the corresponding lines in the pair of `Display` arrays using the `chalk` package

Jest calls this function to consider lines as common instead of changed if the only difference is indentation.

You might call this function for case insensitive or Unicode equivalence comparison of lines.

### Example of diffLinesUnified2

```js
import format from 'pretty-format';

const a = {
  action: 'MOVE_TO',
  x: 1,
  y: 2,
};
const b = {
  action: 'MOVE_TO',
  payload: {
    x: 1,
    y: 2,
  },
};

const difference = diffLinesUnified2(
  // serialize with indentation to display lines
  format(a).split('\n'),
  format(b).split('\n'),
  // serialize without indentation to compare lines
  format(a, {indent: 0}).split('\n'),
  format(b, {indent: 0}).split('\n'),
);
```

The `x` and `y` properties are common, because their only difference is indentation:

```diff
- Expected
+ Received

  Object {
    action: 'MOVE_TO',
+   payload: Object {
      x: 1,
      y: 2,
+   },
  }
```

The preceding example illustrates why (at least for indentation) it seems more intuitive that the function returns the common line from the `bLinesDisplay` array instead of from the `aLinesDisplay` array.

## Usage of diffStringsRaw

Given **strings** and a boolean option, `diffStringsRaw(a, b, cleanup)` does the following:

1. **compare** the strings character-by-character using the `diff-sequences` package
2. optionally **clean up** small (often coincidental) common substrings, also known as chaff

Because `diffStringsRaw` returns the difference as **data** instead of a string, you can format it as your application requires (for example, enclosed in HTML markup for browser instead of escape sequences for console).

The returned **array** describes substrings as instances of the `Diff` class, which calling code can access like array tuples:

The value at index `0` is one of the following:

| value | named export  | description           |
| ----: | :------------ | :-------------------- |
|   `0` | `DIFF_EQUAL`  | in `a` and in `b`     |
|  `-1` | `DIFF_DELETE` | in `a` but not in `b` |
|   `1` | `DIFF_INSERT` | in `b` but not in `a` |

The value at index `1` is a substring of `a` or `b` or both.

### Example of diffStringsRaw with cleanup

```js
const diffs = diffStringsRaw('changed from', 'changed to', true);
```

| `i` | `diffs[i][0]` | `diffs[i][1]` |
| --: | ------------: | :------------ |
| `0` |           `0` | `'changed '`  |
| `1` |          `-1` | `'from'`      |
| `2` |           `1` | `'to'`        |

### Example of diffStringsRaw without cleanup

```js
const diffs = diffStringsRaw('changed from', 'changed to', false);
```

| `i` | `diffs[i][0]` | `diffs[i][1]` |
| --: | ------------: | :------------ |
| `0` |           `0` | `'changed '`  |
| `1` |          `-1` | `'fr'`        |
| `2` |           `1` | `'t'`         |
| `3` |           `0` | `'o'`         |
| `4` |          `-1` | `'m'`         |

### Advanced import for diffStringsRaw

Here are all the named imports that you might need for the `diffStringsRaw` function:

- `const {DIFF_DELETE, DIFF_EQUAL, DIFF_INSERT, Diff, diffStringsRaw} = require('jest-diff');` in CommonJS modules
- `import {DIFF_DELETE, DIFF_EQUAL, DIFF_INSERT, Diff, diffStringsRaw} from 'jest-diff';` in ECMAScript modules

To write a **formatting** function, you might need the named constants (and `Diff` in TypeScript annotations).

If you write an application-specific **cleanup** algorithm, then you might need to call the `Diff` constructor:

```js
const diffCommon = new Diff(DIFF_EQUAL, 'changed ');
const diffDelete = new Diff(DIFF_DELETE, 'from');
const diffInsert = new Diff(DIFF_INSERT, 'to');
```

## Usage of diffLinesRaw

Given **arrays of strings**, `diffLinesRaw(aLines, bLines)` does the following:

- **compare** the arrays line-by-line using the `diff-sequences` package

Because `diffLinesRaw` returns the difference as **data** instead of a string, you can format it as your application requires.

### Example of diffLinesRaw

```js
const aLines = ['delete', 'common', 'changed from'];
const bLines = ['common', 'changed to', 'insert'];

const diffs = diffLinesRaw(aLines, bLines);
```

| `i` | `diffs[i][0]` | `diffs[i][1]`    |
| --: | ------------: | :--------------- |
| `0` |          `-1` | `'delete'`       |
| `1` |           `0` | `'common'`       |
| `2` |          `-1` | `'changed from'` |
| `3` |           `1` | `'changed to'`   |
| `4` |           `1` | `'insert'`       |

## Options

The default options are for the report when an assertion fails from the `expect` package used by Jest.

For other applications, you can provide an options object as a third argument:

- `diffDefault(a, b, options)`
- `diffStringsUnified(a, b, options)`
- `diffLinesUnified(aLines, bLines, options)`
- `diffLinesUnified2(aLinesDisplay, bLinesDisplay, aLinesCompare, bLinesCompare, options)`

### Properties of options object

| name                              | default          |
| :-------------------------------- | :--------------- |
| `aAnnotation`                     | `'Expected'`     |
| `aColor`                          | `chalk.green`    |
| `aIndicator`                      | `'-'`            |
| `bAnnotation`                     | `'Received'`     |
| `bColor`                          | `chalk.red`      |
| `bIndicator`                      | `'+'`            |
| `changeColor`                     | `chalk.inverse`  |
| `commonColor`                     | `chalk.dim`      |
| `commonIndicator`                 | `' '`            |
| `contextLines`                    | `5`              |
| `expand`                          | `true`           |
| `firstOrLastEmptyLineReplacement` | `'↵'`            |
| `includeChangeCounts`             | `false`          |
| `omitAnnotationLines`             | `false`          |
| `patchColor`                      | `chalk.yellow`   |
| `trailingSpaceFormatter`          | `chalk.bgYellow` |

For more information about the options, see the following examples.

### Example of options for labels

If the application is code modification, you might replace the labels:

```js
const options = {
  aAnnotation: 'Original',
  bAnnotation: 'Modified',
};
```

```diff
- Original
+ Modified

  common
- changed from
+ changed to
```

The `jest-diff` package does not assume that the 2 labels have equal length.

### Example of options for colors of changed lines

For consistency with most diff tools, you might exchange the colors:

```js
import chalk from 'chalk';

const options = {
  aColor: chalk.red,
  bColor: chalk.green,
};
```

### Example of option for color of changed substrings

Although the default inverse of foreground and background colors is hard to beat for changed substrings **within lines**, especially because it highlights spaces, if you want bold font weight on yellow background color:

```js
import chalk from 'chalk';

const options = {
  changeColor: chalk.bold.bgAnsi256(226), // #ffff00
};
```

### Example of option to format trailing spaces

Because the default export does not display substring differences within lines, formatting can help you see when lines differ by the presence or absence of trailing spaces found by `/\s+$/` regular expression.

The formatter is a function, which given a string, returns a string.

If instead of yellowish background color, you want to replace trailing spaces with middle dot characters:

```js
const options = {
  trailingSpaceFormatter: string => '·'.repeat(string.length),
};
```

### Example of options for no colors

The value of a color or formatter option is a function, which given a string, returns a string.

To store the difference in a file without escape codes for colors, provide an identity function:

```js
const noColor = string => string;

const options = {
  aColor: noColor,
  bColor: noColor,
  changeColor: noColor,
  commonColor: noColor,
  patchColor: noColor,
  trailingSpaceFormatter: noColor,
};
```

### Example of options for indicators

For consistency with the `diff` command, you might replace the indicators:

```js
const options = {
  aIndicator: '<',
  bIndicator: '>',
};
```

The `jest-diff` package assumes (but does not enforce) that the 3 indicators have equal length.

### Example of options to limit common lines

By default, the output includes all common lines.

To emphasize the changes, you might limit the number of common “context” lines:

```js
const options = {
  contextLines: 1,
  expand: false,
};
```

A patch mark like `@@ -12,7 +12,9 @@` accounts for omitted common lines.

### Example of option for color of patch marks

If you want patch marks to have the same dim color as common lines:

```js
import chalk from 'chalk';

const options = {
  expand: false,
  patchColor: chalk.dim,
};
```

### Example of option to include change counts

To display the number of changed lines at the right of annotation lines:

```js
const a = ['common', 'changed from'];
const b = ['common', 'changed to', 'insert'];

const options = {
  includeChangeCounts: true,
};

const difference = diffDefault(a, b, options);
```

```diff
- Expected  1 -
+ Received  2 +

  Array [
    "common",
-   "changed from",
+   "changed to",
+   "insert",
  ]
```

### Example of option to omit annotation lines

To display only the comparison lines:

```js
const a = 'common\nchanged from';
const b = 'common\nchanged to';

const options = {
  omitAnnotationLines: true,
};

const difference = diffStringsUnified(a, b, options);
```

```diff
  common
- changed from
+ changed to
```

### Example of option not to replace first or last empty lines

If the **first** or **last** comparison line is **empty**, because the content is empty and the indicator is a space, you might not notice it.

Also, because Jest trims the report when a matcher fails, it deletes an empty last line.

The replacement is a string whose default value is `'↵'` U+21B5 downwards arrow with corner leftwards.

To store the difference in a file without a replacement, because it could be ambiguous with the content of the arguments, provide an empty string:

```js
const options = {
  firstOrLastEmptyLineReplacement: '',
};
```

If a content line is empty, then the corresponding comparison line is automatically trimmed to remove the margin space (represented as a middle dot below) for the default indicators:

|         Indicator | untrimmed | trimmed |
| ----------------: | :-------- | :------ |
|      `aIndicator` | `'-·'`    | `'-'`   |
|      `bIndicator` | `'+·'`    | `'+'`   |
| `commonIndicator` | `' ·'`    | `''`    |
