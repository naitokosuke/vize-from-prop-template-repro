# vize-from-prop-template-repro

Reported as [ubugeeei-prod/vize#10661](https://github.com/ubugeeei-prod/vize/issues/10661).

Minimal reproduction for a false type error in [vize](https://github.com/ubugeeei-prod/vize)'s content mapper.

When a prop named `from` is read in the template, the generated code declares it as `_from` but later references it as `from`. This reports TS2552.

## Steps

```sh
pnpm install
pnpm check
```

[`src/Route.vue`](./src/Route.vue):

```vue
<script setup lang="ts">
defineProps<{ from: string }>();
</script>

<template>
  <p>{{ from }}</p>
</template>
```

`pnpm check` runs `tsc --runExternalCode --noEmit -p tsconfig.json` with `"contentMappers": [{ "package": "vize", "extensions": [".vue"] }]`.

## Expected

No error. `from` is a valid identifier and a declared prop.

## Actual

```
src/Route.vue(147,8): error TS2552: Cannot find name 'from'. Did you mean '_from'?
  This location is in virtual code produced by the content mapper 'vize@0.569.0' and has no corresponding location in the original file.
```

## Notes

- Props named `of` and `as` give the same error. Other contextual keywords (`get`, `set`, `type`, `async`, `await`, `yield`, `let`, `static`, `satisfies`) do not; `static` was fixed in [#910](https://github.com/ubugeeei-prod/vize/issues/910).
- The generated code, from `vize check --save-virtual-ts-for src/Route.vue src`:

  ```ts
  const _from = props["from"];
  void _from;

  void (props["from"]); // Interpolation
  // @vize-map: expr -> 88:92

  // Reference setup bindings (used in template/CSS v-bind)
  void from;
  ```

- [`is_reserved_identifier`](https://github.com/ubugeeei-prod/vize/blob/v0.569.0/crates/vize_canon/src/virtual_ts/helpers.rs#L288-L341) counts `as`, `from` and `of` as reserved, so [`to_safe_identifier`](https://github.com/ubugeeei-prod/vize/blob/v0.569.0/crates/vize_canon/src/virtual_ts/helpers.rs#L219-L234) names the prop's binding `_from`. [`is_reserved_anchor_name`](https://github.com/ubugeeei-prod/vize/blob/v0.569.0/crates/vize_canon/src/virtual_ts/generator/anchors.rs#L158-L204) does not list those three, so [`emit_setup_binding_anchors`](https://github.com/ubugeeei-prod/vize/blob/v0.569.0/crates/vize_canon/src/virtual_ts/generator/anchors.rs#L134-L152) still writes `void from;`.
- `vize check src` reports `No type errors found!`. Its native TypeScript checker reports the same TS2552 (`src/Route.vue.ts(57,8)`), but Vize drops diagnostics that have no location in the `.vue` file.

## Environment

vize 0.569.0, TypeScript 7.1.0-beta, Vue 3.5.44, Node v26.12.1, pnpm 12.9.0, macOS (arm64)
