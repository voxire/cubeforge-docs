# TileLayer

A large tile grid stored as one typed array and drawn by the GPU: a 600×300 map
is one draw call, never one entity per tile.

```tsx
const ground = useTileLayer({
  width: 600,
  height: 300,
  tileset: { src: '/tiles.png', tileWidth: 16, tileHeight: 16, columns: 32 },
  animations: { 12: { frames: [12, 13, 14], duration: 0.2 } },
})
return <TileLayer layer={ground} zIndex={0} />
```

## Options (`useTileLayer`)

| Option | Type | Description |
|---|---|---|
| `width`, `height` | `number` | Size in tiles |
| `tileset` | `{ src?, image?, tileWidth, tileHeight, columns, spacing?, margin? }` | Atlas |
| `tiles` | `ArrayLike<number>` | Initial ids (0 = empty, `n` = atlas tile `n - 1`) |
| `wideIds` | `boolean` | Use `Uint32Array` instead of `Uint16Array` |
| `tileWorldWidth`, `tileWorldHeight` | `number` | Tile size in world units (default: tileset size) |
| `animations` | `Record<id, { frames, duration }>` | Animated ids, evaluated on the GPU |
| `chunkSize` | `number` | Dirty-tracking chunk size (default 32) |

`<TileLayer>` props: `layer`, `x`, `y`, `zIndex`, `opacity`, `visible`.

## Methods

| Method | Cost |
|---|---|
| `setTile(x, y, id)` | O(1); next frame uploads only the touched region |
| `getTile(x, y)` | O(1) |
| `setTiles(array)` / `fill(id)` | O(tiles) plus one full index upload |
| `setAnimations(map)` | Small lookup-table upload |

Tile layers draw after parallax backgrounds and before sprites. Sampling reads
exact atlas texels, so there are no seams at fractional zoom or any
devicePixelRatio. Use `<Camera2D pixelSnap />` for crisp pixel art while
panning.
