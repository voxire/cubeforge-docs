# SpriteLayer

A struct-of-arrays batch of sprites for data-driven games: no entity or React
node per sprite. See the [data-driven guide](/guide/data-driven-games).

```tsx
const layer = useSpriteLayer({ src: '/atlas.png', frameWidth: 16, frameHeight: 16, zIndex: 2 })
```

## Options

| Option | Type | Default | Description |
|---|---|---|---|
| `src` | `string` | | Texture URL |
| `image` | `HTMLImageElement \| HTMLCanvasElement \| ImageBitmap \| OffscreenCanvas` | | Or a loaded image |
| `dynamicSrc` | `string` | | Or a `useDynamicCanvas` id |
| `frameWidth`, `frameHeight` | `number` | | Atlas cell size in texture pixels |
| `frameColumns` | `number` | image width / frameWidth | Cells per row |
| `layer` | `string` | `'default'` | Render layer |
| `zIndex` | `number` | `0` | Sorted with regular sprites |
| `anchorX`, `anchorY` | `number` | `0.5` | Anchor for every sprite |
| `sampling` | `Sampling` | renderer default | Texture filter |
| `capacity` | `number` | `256` | Initial capacity (grows) |
| `visible` | `boolean` | `true` | |

Without a texture the layer draws solid rects in each sprite's `color`.

## Data

`x`, `y`, `w`, `h`, `rotation`: `Float32Array` · `frame`: `Uint32Array` ·
`color`: `Uint32Array` (0xRRGGBBAA) · `flags`: `Uint8Array`
(`SPRITE_FLIP_X | SPRITE_FLIP_Y | SPRITE_HIDDEN`) · `ids`: `Int32Array` · `count`.

## Methods

| Method | Description |
|---|---|
| `add(x, y, w, h, frame?, id?)` | Append; returns the index |
| `set(i, x, y, frame?)` | Move one sprite |
| `removeAt(i)` | Swap-remove (the last sprite moves into `i`) |
| `resize(n)` / `reserve(n)` / `clear()` | Count and capacity |
| `touch()` | Call after writing arrays directly |
| `pick(x, y)` / `pickIndex(x, y)` | Topmost sprite id / index at a world point, or -1 |

## Cost

Per frame: one pass over `count` sprites (cull + 19 floats written), one
instanced draw per 16,384 visible sprites. About 0.06 ms CPU for 3,000 sprites
(headless benchmark), versus about 0.3 ms through entities.
