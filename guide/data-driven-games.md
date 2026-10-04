# Data-Driven Games

Some games keep their state outside React: a simulation, a server, or a worker
produces thousands of positions every tick, and the engine only has to draw
them. Driving that through one `<Entity>` per object means React reconciles
thousands of components a frame. Cubeforge has a fast path for this shape.

| Data | Use | Per-frame cost |
|---|---|---|
| Thousands of moving sprites | `useSpriteLayer` | One tight loop over typed arrays, one instanced draw per 16k sprites |
| Large grid (tiles, terrain, fog) | `useTileLayer` + `<TileLayer>` | One quad per visible page; only on-screen pixels cost anything |
| Free-form 2D drawing (effects, overlays) | `useDynamicCanvas` | Upload only the rect you mark dirty |
| A few interactive objects (UI, the player) | `<Entity>` + components | React per prop change, ECS per frame |

Keep React for structure (which layers exist, their textures, HUD) and write
per-frame data straight into the layers.

## Sprites from simulation data

```tsx
import { Game, World, Camera2D, Entity, Script, useSpriteLayer, SPRITE_FLIP_X } from 'cubeforge/render'

function People({ sim }: { sim: Simulation }) {
  const people = useSpriteLayer({ src: '/people.png', frameWidth: 16, frameHeight: 16, zIndex: 5 })

  return (
    <Entity>
      <Script
        update={() => {
          people.resize(sim.count)
          for (let i = 0; i < sim.count; i++) {
            people.x[i] = sim.x[i]
            people.y[i] = sim.y[i]
            people.w[i] = 16
            people.h[i] = 16
            people.frame[i] = sim.frame[i]
            people.flags[i] = sim.facingLeft[i] ? SPRITE_FLIP_X : 0
            people.ids[i] = sim.id[i]
          }
          people.touch()
        }}
      />
    </Entity>
  )
}
```

- Arrays: `x`, `y`, `w`, `h`, `rotation` (Float32Array), `frame` (atlas cell,
  row-major), `color` (0xRRGGBBAA tint), `flags` (`SPRITE_FLIP_X`,
  `SPRITE_FLIP_Y`, `SPRITE_HIDDEN`), `ids` (your ids, returned by `pick`).
- `add()`, `set()`, `removeAt()` (swap-remove) and `resize()` grow capacity
  automatically. After writing arrays directly, call `touch()` so idle-frame
  skipping redraws.
- One layer is one texture. Use an atlas; layers sort with regular sprites by
  `layer` and `zIndex`.

### Picking

```ts
const { screenToWorld } = useCoordinates()
const w = screenToWorld(e.offsetX, e.offsetY)
const personId = people.pick(w.x, w.y) // topmost sprite, -1 if none
```

`pick` is a linear scan from the top (a few microseconds for 3,000 sprites) and
ignores rotation.

## Large grids

```tsx
const ground = useTileLayer({
  width: 600,
  height: 300,
  tileset: { src: '/tiles.png', tileWidth: 16, tileHeight: 16, columns: 32 },
  animations: { 12: { frames: [12, 13, 14, 13], duration: 0.2 } },
})
sim.onTileChanged = (x, y, id) => ground.setTile(x, y, id)
sim.onSeason = (tiles) => ground.setTiles(tiles)
return <TileLayer layer={ground} zIndex={0} />
```

Tile id 0 is empty; id `n` is atlas tile `n - 1`. See [TileLayer](/api/tile-layer).

## Camera

`useCamera()` gives `setPosition`, `setZoom`, and `zoomAt(screenX, screenY, zoom)`
for zoom-to-cursor on wheel or pinch. `screenToWorld` / `worldToScreen` from
`useCoordinates()` work in canvas CSS pixels at any devicePixelRatio.

```ts
canvas.addEventListener('wheel', (e) => {
  camera.zoomAt(e.offsetX, e.offsetY, camera.getZoom() * Math.exp(-e.deltaY * 0.001))
})
```

## Moving a hand-written canvas renderer onto the engine

A common starting point is one big 2D canvas you draw everything on, shown as a
single sprite. Move it piece by piece; each step is independently shippable.

1. **Keep the canvas, upload less.** Draw into `useDynamicCanvas` and pass the
   changed region to `markDirty(x, y, w, h)` instead of re-uploading the whole
   canvas each frame.
2. **Move the static grid** (terrain, floors) into a `TileLayer`. Delete the code
   that drew tiles onto the canvas.
3. **Move the crowd** (people, animals, buildings) into one `SpriteLayer` per
   atlas. Replace per-object `drawImage` calls with array writes.
4. **Keep the canvas for what is genuinely free-form** (speech bubbles, weather,
   debug drawing), or move text to `<Text>` entities if there are few of them.
5. **Pick with the layers** (`layer.pick`) instead of your own hit testing.

## Measuring

Add `<StatsOverlay />` inside `<Game>` to see frame time split, draw calls,
instances, culled sprites, texture memory and cache hit rates, or read the
same numbers with `useEngineStats()`. `pnpm bench` in the engine repo runs the
stress scenarios headlessly.
