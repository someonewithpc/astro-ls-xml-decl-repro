# astro check rejects `<?xml ... ?>` with @astrojs/language-server 2.17

`astro check` on language-server 2.17.x reports an error on every XML processing
instruction in an `.astro` template, including a well-formed
`<?xml version="1.0" encoding="UTF-8"?>`. Language-server 2.16.16 reports nothing,
and `astro build` compiles the same file and keeps the declaration in the output.

## Versions

| Package | Version |
| --- | --- |
| astro | 7.3.3 (uses @astrojs/compiler-rs 0.4.1) |
| @astrojs/check | 0.9.10 |
| @astrojs/language-server | 2.16.16 (passes), 2.17.1 (fails) |
| @astrojs/astro2tsx | 0.1.2 (pulled in by language-server 2.17) |
| typescript | 6.0.3 |
| node | 22.23.1 |
| npm | 10.9.8 |

`package.json` forces the language-server version through `overrides`. As committed
it pins 2.17.1, the failing version.

## Reproduce

```sh
npm install
npx astro check     # 1 error
npx astro build     # builds fine
```

To see the passing case, set `"@astrojs/language-server": "2.16.16"` in
`overrides`, run `npm install` again, then `npx astro check`.

`src/components/Logo.astro`:

```astro
<?xml version="1.0" encoding="UTF-8"?>
<!-- Exported from an SVG editor -->
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 10 10" {...Astro.props}>
	<circle cx="5" cy="5" r="4" />
</svg>
```

## Expected

Language-server 2.16.16:

```
Result (3 files):
- 0 errors
- 0 warnings
- 0 hints
```

## Actual

Language-server 2.17.1:

```
src/components/Logo.astro:1:38 - error astro: expected `?>` but instead found `>`

1 <?xml version="1.0" encoding="UTF-8"?>
                                       ~

Result (3 files):
- 1 error
- 0 warnings
- 0 hints
```

Column 38 is the `>` of the closing `?>`. The parser seems to consume the `?` and
then look for `?>` again.

## Narrowing

Each variant replaced `Logo.astro` and ran under 2.17.1. All of them fail with the
same message, always pointing at the final `>`:

| Variant | Result |
| --- | --- |
| `<?xml version="1.0"?>` (no encoding) | `1:21 expected ?> but instead found >` |
| `<?xml version='1.0'?>` | `1:21`, same |
| `<?xml version="1.0" ?>` (space before `?>`) | `1:22`, same |
| `<?xml?>` | `1:7`, same |
| `<?foo bar?>` (any processing instruction) | `1:11`, same |
| declaration alone, no `<svg>` | `1:38`, same |
| declaration after a frontmatter block | `4:38`, same |
| declaration after the `<svg>` | same, plus TypeScript errors ts(1382) and ts(1003) on the line |

The encoding attribute, the `<svg>`, its position in the file and the frontmatter
make no difference. The smallest trigger is a template containing only `<?xml?>`.

## Cause

Language-server 2.17.0 replaced the Go compiler's TSX output with a new Rust
`astro2tsx`
([withastro/astro#17676](https://github.com/withastro/astro/pull/17676)). The
changelog says:

> The language server now uses our all-new Biome powered `astro2tsx` implementation.

The error string lives in `@astrojs/astro2tsx` (`astro2tsx.wasm32-wasi.wasm`, from
[withastro/compiler-rs](https://github.com/withastro/compiler-rs), `crates/astro2tsx`).
None of the astro2tsx changelog entries up to 0.1.2 mention processing instructions.
The Astro 7 build compiler, also from compiler-rs, accepts the file, so only the
editor tooling parser rejects it.

## Duplicates

I searched withastro/astro, withastro/compiler-rs, withastro/language-tools and
withastro/compiler for "instead found", "xml declaration", "processing instruction"
and `<?xml` on 2026-10-05 and found no matching issue. The closest is
withastro/compiler-rs#194, a different astro2tsx 0.1.1 parse error caused by a
`/>/g` regex in frontmatter, fixed in 0.1.2.

The bug report belongs in withastro/compiler-rs, which is where the astro2tsx
package's `bugs` field points.
