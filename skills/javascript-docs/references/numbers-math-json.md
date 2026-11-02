# Numbers, BigInt, Math and JSON

## Contents

- Number precision and checks
- Parsing and rounding
- BigInt
- Math
- JSON
- Old patterns

## Number precision and checks

- Numbers are 64-bit floats. Integers are exact only between -(2^53-1) and 2^53-1, and JSON parsers corrupt larger ones too. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number
- Use `Number.isNaN()` and `Number.isFinite()`. The global `isNaN()` and `isFinite()` coerce first. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/isNaN
- Compare floats with a tolerance scaled to the magnitude, not a bare `Number.EPSILON`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/EPSILON

```js
function equal(x, y, tolerance = Number.EPSILON) {
  return Math.abs(x - y) < tolerance;
}
equal(1000.1 + 1000.2, 2000.3, 2000 * Number.EPSILON); // true
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/EPSILON

- `Number.MIN_VALUE` is the smallest positive number, not the most negative. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/MIN_VALUE
- `Number.isSafeInteger(2 ** 53)` is false: 2^53 is representable but 2^53+1 rounds to it. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/isSafeInteger
- Number coercion: `null` and `""` give 0, `undefined` gives NaN, numeric separators are not allowed in strings, BigInt and Symbol throw. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number
- Never compare `toLocaleString()` output with fixed strings: it may hold non-breaking spaces or bidi marks. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/toLocaleString

## Parsing and rounding

- Always pass a radix to `parseInt()`: `0x` input switches to base 16. It stops at the first invalid character (`parseInt("1e3", 10)` is 1). https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/parseInt
- Parse `0b`, `0o`, `0x` strings with `Number()` or `BigInt()`, and large integers with `BigInt("...")`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/parseInt
- Drop fractions with `Math.trunc()`, not `parseInt()` (fails on exponent notation) or bitwise tricks (32-bit). `Math.trunc(-0.123)` is `-0`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/trunc
- `Math.round(-20.5)` is -20: halves round toward +Infinity. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/round
- `toFixed()` rounds the binary value: `(2.55).toFixed(1)` is `"2.5"`. It returns a string and switches to exponent notation at 1e21. Group a negative literal: `(-2.34).toFixed(1)`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/toFixed
- Typed arrays and bitwise operators wrap: `new Int8Array([257, -257])` is `[1, -1]`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number

## BigInt

- Use BigInt only when values beyond 2^53 are expected, and do not convert back and forth. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/BigInt
- BigInt does not mix with Number in arithmetic, does not work with `Math`, rejects unary `+` and `>>>`, and `0n === 0` is false while `0n == 0` is true. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/BigInt
- Sort mixed arrays with `(a, b) => (a < b ? -1 : a > b ? 1 : 0)`: `a - b` throws on mixed types, and default `sort()` compares strings. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/BigInt
- `BigInt(undefined)` and `BigInt(null)` throw `TypeError`, bad strings `SyntaxError`, non-integers `RangeError`. Validate before converting. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Cant_convert_x_to_BigInt
- APIs that need a BigInt reject numbers: `BigInt.asIntN(4, 8n)`, `new BigInt64Array(3).fill(3n)`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Cant_convert_x_to_BigInt
- BigInt operations are not constant time: do not use them for cryptography. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/BigInt
- `Number(bigint)` is the explicit conversion that does not throw, but large values lose precision. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/Number

## Math

- Random integers: `Math.floor(Math.random() * range)`, never `Math.round` (non-uniform). For security use `crypto.getRandomValues()`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/random
- Use `Math.atan2(y, x)` (y first) over `Math.atan(y / x)`, which gives `NaN` or the wrong quadrant in edge cases. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/atan2
- For tiny `x`, use `Math.log1p(x)` instead of `Math.log(1 + x)` and `Math.expm1(x)` instead of `Math.exp(x) - 1`: adding or subtracting 1 loses precision. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/log1p , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/expm1
- Sum floats with `Math.sumPrecise(iterable)`. It cannot fix inexact literals, returns `-0` for `[]` and throws on non-numbers. Polyfills are listed, so check availability. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/sumPrecise

```js
Math.sumPrecise([1e20, 0.1, -1e20]); // 0.1 (a plain += loop gives 0)
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/sumPrecise

## JSON

- `JSON.stringify` drops `undefined`, functions and symbols in objects (they become `null` in arrays), turns `NaN`/`Infinity` into `null`, Maps and Sets into `{}`, and throws on BigInt and cycles. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/stringify
- Dates and Temporal objects serialize as strings through `toJSON()`, which cannot be reversed automatically. Write a reviver for the payload shape. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/stringify , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/toJSON
- A reviver must return unchanged every value it does not transform, or that value is deleted. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/parse
- Revivers run after numbers are parsed, so precision is already lost: `JSON.parse("12345678901234567890")` is `12345678901234568000`. Read `context.source` to rebuild the exact value. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/parse , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON
- Send large integers as strings revived by key, or as exact numbers with `JSON.rawJSON()` when the recipient handles them. `rawJSON` accepts primitives only. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON , https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/rawJSON

```js
BigInt.prototype.toJSON = function () {
  return JSON.rawJSON(this.toString());
};
const parsed = JSON.parse(str, (key, value, context) =>
  key === "gross_gdp" ? BigInt(context.source) : value);
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON

- Serialize a Map with a replacer and pair it with a reviver. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/parse

```js
const jsonText = JSON.stringify(map, (key, value) =>
  value instanceof Map ? Array.from(value.entries()) : value);
```
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/parse

- JSON allows no trailing commas, comments, single quotes, leading zeros, `NaN` or `Infinity`. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON
- `space` is capped at 10. A replacer is first called for the root with key `""`, so to tell it from a real `""` key, track whether it is the first call. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/stringify

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `isNaN(x)`, `isFinite(x)` | `Number.isNaN(x)`, `Number.isFinite(x)` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/isFinite |
| `~~x`, `x \| 0`, `parseInt(x)` to truncate | `Math.trunc(x)` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/trunc |
| `new Number(x)` | `Number(x)` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/Number |
| Summing floats with `+=` | `Math.sumPrecise()` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/sumPrecise |
| `Number.toInteger()` | `Math.floor()` or `Math.round()` | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Deprecated_and_obsolete_features#obsolete_features |
