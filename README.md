# primer-blank-skin

The skin of an instance of
[primer-blank](https://github.com/inzoddwetrust/primer-blank): colors, fonts
and the logo, as data. A package of an instance (contract 3 rule 10 of the
deploy machine `pridem`): no code, no instance values; which package and which
version an instance shows is in its passport.

```
primer-package.yaml          schema: 1, contract: "1.0"
tokens/color.tokens.json     W3C DTCG tokens
tokens/typography.tokens.json
assets/logo.svg              the logo (required)
assets/icons/check.svg       icons
```

Releases are tags `vX.Y.Z`; a released tag is never moved -- an instance goes
back to a look by its tag. `v1.0.0` is blue, `v1.1.0` green.

## The format, contract "1.0"

- `primer-package.yaml` in block form, two keys: `schema: 1` and
  `contract: "X.Y"`. The product reads contract `1.x` (`contract: ">=1,<2"` in
  its `primer.yaml`); a new optional token is a minor version, a renamed or
  removed token or a new required one is a major one.
- Tokens: `tokens/<name>.tokens.json`, W3C Design Tokens (DTCG). A token has
  `$value`; its `$type` is its own or its group's. Types: `color` (`#hex`, or
  `{colorSpace: "srgb", components: [r, g, b], alpha}` in 0..1), `dimension`
  (`"16px"`, `"1.5rem"`, or `{value, unit: "px" | "rem"}`), `fontFamily` (a name
  or a list), `fontWeight` (1..1000 or a DTCG keyword), `number`. A value may be
  a reference to another token, `"{color.brand.primary}"`, of the same type.
  Names are letters, digits, `_` and `-`; `$description`, `$extensions` and
  `$deprecated` are read past.
- Required tokens: `color.brand.primary`, `color.surface`, `color.text`
  (`color`), `font.family.base` (`fontFamily`). Every token becomes a CSS custom
  property of its path: `color.brand.primary` -> `--color-brand-primary`.
- Assets: `assets/<name>.svg|png` and `assets/icons/<name>.svg|png`, names of
  `a-z`, `0-9` and `-`; `assets/logo.svg` is required. An SVG is at most 64 KiB,
  with no script, event handler, `javascript:`, `foreignObject`, embedded
  document, DOCTYPE or ENTITY, and no link out of the file; a PNG is at most
  256 KiB and 1024 px a side. An asset becomes `--asset-<name>` or
  `--asset-icon-<name>`.
- Nothing else: any other file, a symlink or an executable file is refused.

## How an instance gets it

The passport names the package: `packages: {skin: {repo, version}}`. The
machine clones the tag, checks `primer-package.yaml` against the product's
`contract` range and runs the product's check in the product's new image --
here `validate-skin <dir>`; a skin it refuses is red and the instance keeps its
look. primer-blank applies the skin at run time (`apply: runtime`): the package
is mounted read-only at `/srv/skin` and the frontend's container restarts on the
same image, in seconds, with no build.

## One format for both modes

The format is the same whether a product applies its skin at run time or at
build time (`apply: build`: the machine puts the package into the build
context and a new version rebuilds the frontend on the same commit -- see part
4.2 of [the guide for projects](https://claude.ai/artifact/PvmYK75tYbb7uZ3HC1ZoZp)).

The check and the conversion to CSS are one module,
[`frontend/nginx/package.js`](https://github.com/inzoddwetrust/primer-blank/blob/main/frontend/nginx/package.js)
of primer-blank. It is written in the subset of JavaScript that njs (nginx's
JavaScript, which runs it in primer-blank's image) and Node share, so a product
that applies its skin at build time can take the same module into its Node
build and get the same check and the same CSS:

```js
import fs from 'node:fs'
import primer from './package.js' // primer-blank/frontend/nginx/package.js

const skin = primer.checkSkin('public/skin') // where the machine puts the package
if (skin.problems.length > 0) {
  throw new Error(skin.problems.join('\n')) // each: file, token, what is wrong
}
fs.writeFileSync('src/skin.css', skin.css) // :root { --color-brand-primary: ...; }
```

To check a skin by hand: `node -e` with the same `checkSkin`, or the
frontend image of primer-blank:
`docker run --rm -v "$PWD:/srv/skin:ro" <image> validate-skin /srv/skin`.
