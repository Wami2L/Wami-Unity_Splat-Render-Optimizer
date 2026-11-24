# Wami Splat Optimizer And Render For Unity

**Gaussian Splatting rápido para Unity y VR** · *Fast Gaussian Splatting for Unity and VR*

[Español](#español) · [English](#english)

---

## Español

Wami Splat Optimizer (paquete `splatkit`) prepara escenas 3D Gaussian Splatting (3DGS) para que corran rápido en visores (Quest 2/3, PC VR) y en
escritorio, siguiendo lo que hace rápido a PlayCanvas/SuperSplat: **dibujar menos splats y hacer menos trabajo por
cuadro**. Tiene dos partes que comparten los mismos perfiles de calidad:

| Parte | Carpeta | Qué hace |
|---|---|---|
| Cadena offline (Python) | `python/` | Análisis, limpieza (flotantes, agujas, niebla, gigantes), recorte de SH, nivelación y escala desde COLMAP, **pirámide LOD por trozos**, colisiones, banco de comparación (PSNR/SSIM). Sin GPU; lee PLY de varios GB por bloques. |
| Runtime de Unity (UPM) | `unity/com.wami.splatoptimizer/` | Renderizador (basado en [gsplat-unity](https://github.com/wuyize25/gsplat-unity), MIT) + `SplatLod` (LOD dinámico con presupuesto) + `SplatQuality` (perfiles y calidad automática) + banco en batchmode. |

El formato de salida (PLY 3DGS estándar + manifiesto JSON de la pirámide) no depende de Unity: cualquier motor puede
leerlo.

### Por qué va más rápido que gsplat/aras-p tal cual

| Técnica | gsplat original | SplatKit |
|---|---|---|
| Cuándo ordena | **cada cuadro**, todos los splats | solo si la cámara se movió (`OnCameraMove`) o cambió la selección |
| Profundidad para ordenar | todos los splats cada cuadro | solo cuando ordena |
| Claves de orden | 32 bits (4 pasadas de radix) | 16 bits cuantizadas (2 pasadas) |
| Orden en VR | por el eje de la vista: girar la cabeza reordena | **radial**: girar la cabeza no reordena |
| LOD | — | pirámide por trozos con presupuesto de splats, modo VR estable (sin recorte por frustum; lo de detrás baja de nivel como `lodBehindPenalty`) |
| Descartes | splats < 2 px | tamaño mínimo y **contribución mínima** configurables (`minContribution` de PlayCanvas) |
| Antialias (Mip-Splatting) | no | compensación de opacidad del filtro 2D |
| Resolución | fija | por perfil + automática según el tiempo de GPU |
| Primer orden de cada renderer | aleatorio (la profundidad se calculaba antes de inicializar el orden) | correcto |

### Medidas (RTX 3050 Laptop, un ojo 1440×1584, D3D12, banco en el editor)

Recorrido con la cámara en movimiento (p50 por cuadro; el render vacío del editor ya cuesta 5,8 ms), 600 k splats:

| Escena | gsplat original | `pcvr_low` (resolución 0,7) | `pcvr_low` (resolución 1,0) |
|---|---|---|---|
| Iglesia (9,98 M splats, interior) | 14,7 ms | **9,0 ms** | 14,0 ms |
| Árbol del jardín (1 M, antialias) | 12,8 ms | **8,1 ms** | 13,3 ms |

- Calidad de la iglesia a resolución completa frente al original: PSNR 34 dB, SSIM 0,97 (prácticamente igual).
- Girar la cabeza 360° en el sitio: **288 → 17 ordenaciones** y 144 → 17 selecciones de LOD.
- En reposo ya no se ordena: iglesia 14,3 → 12,9 ms a resolución completa.
- Lo que más pesa es el relleno de píxeles; la ordenación es lo de menos. Por eso los niveles gruesos de la pirámide (splats
  fusionados, más gordos) apenas abaratan: 150 k gruesos cuestan solo un 13 % menos que 600 k finos. Siguientes pasos
  en `docs/PLAN.md`.

*Measured on an RTX 3050 Laptop, one 1440×1584 eye: moving camera p50 14.7 → 9.0 ms (church) and 12.8 → 8.1 ms (tree)
with `pcvr_low`; head turn 288 → 17 sorts; fill rate, not sorting, dominates.*

### Perfiles

`python -m splatkit perfiles` (o `--lang en`). El mismo archivo (`python/splatkit/data/profiles.json`) va copiado en
el paquete de Unity (`Runtime/Resources/SplatKitProfiles.json`; un test comprueba que son idénticos).

| Perfil | Splats | Resolución | SH | Para |
|---|---|---|---|---|
| `auto` | según el dispositivo | — | — | elige uno de los de abajo y ajusta solo el presupuesto |
| `quest2` | 150 k | 70 % | 0 | Quest 2 autónomo, 72 Hz |
| `quest3` | 400 k | 80 % | 0 | Quest 3 autónomo |
| `pcvr_low` | 600 k | 70 % | 1 | portátil (RTX 3050) + Vive Pro / Link, 90 Hz |
| `pcvr_high` | 1,5 M | 100 % | 1 | GPU de sobremesa |
| `desktop` | 2 M | 100 % | 1 | pantalla plana |
| `ultra` | sin límite | 100 % | 3 | capturas y referencia de calidad |

### Uso rápido

```bash
cd python
python -m pip install -e .            # o: python -m splatkit ... desde esta carpeta
splatkit preparar escena.ply --nombre jardin --colmap <colmap/sparse/0> --unity "<proyecto Unity>"
splatkit camaras escenas/jardin/piramide.ply --lichtfeld recorrido.json   # cámaras para el banco
```

### En Unity

1. Añade el paquete: `Packages/manifest.json` → `"com.wami.splatoptimizer": "file:<ruta>/unity/com.wami.splatoptimizer"`
   (o *Package Manager › Add package from disk*). La URP necesita la renderer feature `Gsplat URP Feature` y
   D3D12/Vulkan/Metal (DX11 y GLES no sirven: la ordenación usa operaciones de *wave*).
2. **Tools › Wami Splat Optimizer › Importar escena 3DGS**: eliges el `.ply` (y la carpeta COLMAP `sparse/0`, que se
   busca sola), pulsas *Importar* y la ventana lanza `python -m splatkit preparar`: limpieza, escala real y suelo
   desde las cámaras, pirámides LOD de PC y de visores autónomos, colisiones. Al terminar mete los archivos en el
   proyecto y crea:
   - una **escena de escritorio** (cámara de vuelo; teclas 1/2 cambian entre la versión de PC y la de Quest);
   - una **escena VR** (si el proyecto tiene XR Interaction Toolkit y sus Starter Assets): rig con andar, girar y
     teletransporte, colisiones sacadas de los splats y, si quieres, una mesa con objetos agarrables.
   Se puede cancelar, sobrevive a recargas de scripts y lista los entornos ya importados (abrir, recrear escenas).
   Antes de importar marcas qué **calidades** exportar; todas van con LOD y aparecen como variantes de la escena
   (teclas 1-3):

   | Calidad | Qué lleva | Para | Jardín de ejemplo |
   |---|---|---|---|
   | **Low detail** | sin los splats originales ni SH | visores autónomos, equipos modestos | 72 MB |
   | **High detail** | todos los niveles, SH1 | PC y PC VR | 193 MB |
   | **Original (máxima calidad)** | los splats originales con todos sus SH | capturas, PC potente | 495 MB |

   Desde la línea de órdenes: `splatkit preparar … --piramides low,high,max`.
3. Componentes (*Add Component › Wami Splat Optimizer*): `Splat Renderer`, `Splat LOD`, `Splat Quality` (perfil),
   `Splat Variants`, `Fly Camera`, `Respawn`.
4. Carpetas (entornos, escenas, prefijo, materiales, capturas): *Project Settings › Wami Splat Optimizer › Import* o
   en la propia ventana; se guardan en `ProjectSettings/WamiSplatOptimizer.json`.

Sin editor (CI): `-executeMethod SplatKit.EditorTools.SplatImport.BatchImport -ply x.ply -nombre x [-colmap …]
[-antialias] [-sinVR]`, y `SplatEnvironments.BatchCreateAndCapture` / `SplatVrScene.BatchCreateAndCapture [-entorno x]`
para recrear escenas y capturarlas.

Recortes: `splatkit recortar escena.ply --objetos objetos.json` saca objetos sueltos (cajas en ejes de Unity) con el
pivote en la base, para usarlos como assets.

Banco (Unity cerrado):

```bash
Unity.exe -batchmode -force-d3d12 -projectPath <proyecto> -executeMethod SplatKit.EditorTools.SplatBench.Run \
  -benchPiramide escenas/jardin/piramide.json -benchCams escenas/jardin/camaras_recorrido_orbita.json \
  -benchOut bench -benchProfile pcvr_low -benchRecorrido 24
```

`-benchLegacy` reproduce gsplat original para comparar. Luego `splatkit comparar bench --referencia ref --variantes a,b`.

### Licencia

MIT. Incluye código de gsplat-unity (MIT), GPUSorting de Thomas Smith (MIT), aras-p/UnityGaussianSplatting (MIT) y
ZstdSharp (MIT): ver `unity/com.wami.splatoptimizer/Third Party Notices.md`.

---

## English

Wami Splat Optimizer (package `splatkit`) prepares 3D Gaussian Splatting scenes to run fast on headsets (Quest 2/3, PC VR) and desktop, following what
makes PlayCanvas/SuperSplat fast: **draw fewer splats and do less work per frame**. Two parts share the same quality
profiles:

| Part | Folder | What it does |
|---|---|---|
| Offline pipeline (Python) | `python/` | Analysis, cleanup (floaters, needles, haze, huge splats), SH trimming, levelling and scale from COLMAP, **chunked LOD pyramid**, collisions, comparison bench (PSNR/SSIM). No GPU; reads multi-GB PLY files in blocks. |
| Unity runtime (UPM) | `unity/com.wami.splatoptimizer/` | Renderer (based on [gsplat-unity](https://github.com/wuyize25/gsplat-unity), MIT) + `SplatLod` (budgeted dynamic LOD) + `SplatQuality` (profiles and adaptive quality) + batch-mode bench. |

The output (standard 3DGS PLY + pyramid JSON manifest) does not depend on Unity: any engine can read it.

### Why it is faster than stock gsplat/aras-p

| Technique | stock gsplat | SplatKit |
|---|---|---|
| When it sorts | **every frame**, all splats | only when the camera moved (`OnCameraMove`) or the selection changed |
| Sort depths | every splat every frame | only when sorting |
| Sort keys | 32 bits (4 radix passes) | quantized 16 bits (2 passes) |
| VR sort | view axis: turning the head re-sorts | **radial**: turning the head does not re-sort |
| LOD | — | chunked pyramid with a splat budget, stable XR mode (no frustum cull; chunks behind get coarser, like `lodBehindPenalty`) |
| Culling | splats < 2 px | configurable min size and **min contribution** (PlayCanvas `minContribution`) |
| Antialiasing (Mip-Splatting) | no | 2D-filter opacity compensation |
| Resolution | fixed | per profile + adaptive from the GPU time |

### Profiles

`python -m splatkit --lang en perfiles`. The same file (`python/splatkit/data/profiles.json`) is copied into the Unity
package (`Runtime/Resources/SplatKitProfiles.json`; a test checks they are identical). See the table above.

### Quick start

```bash
cd python
python -m pip install -e .
splatkit --lang en preparar scene.ply --nombre garden --colmap <colmap/sparse/0> --unity "<Unity project>"
```

In Unity: add the package, then `GsplatRenderer` + `SplatLod` (with `piramide.json`) + `SplatQuality` (profile). URP
needs the `Gsplat URP Feature` renderer feature and D3D12/Vulkan/Metal.

### License

MIT. Includes gsplat-unity (MIT), Thomas Smith's GPUSorting (MIT), aras-p/UnityGaussianSplatting (MIT) and ZstdSharp
(MIT): see `unity/com.wami.splatoptimizer/Third Party Notices.md`.
