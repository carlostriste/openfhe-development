# Traspaso de trabajo — OpenFHE sparse/fideslib (rama dev-pr-fideslib-remaining)

Fecha: 2026-09-23. Repo: `openfhe-patch-1.5.1` (OpenFHE v1.5.1).
Estado: **RESUELTO** — el régimen de fase `m==slots` completo (encoding + decoding) funciona
en sparse, y el fix está commiteado en `dev-pr` y `dev-pr-fideslib-remaining`.

---

## 1. Contexto / objetivo original del usuario

1. **Verificar que SPARSE_ENCAPSULATED funciona en el checkout actual** → ✅ hecho (todo PASS salvo 4 fallos pre-existentes).
2. **Crear rama + `.patch`** con los cambios restantes de `fideslib-add-sparse` sobre `dev-pr`, excluyendo artefactos `*.patch`.
3. **Aislar la causa del fallo** en `BOOTSTRAP_SPARSE` y lograr que el cambio de fase `m==2→m==slots` (optimización de fideslib) funcione **también en decoding** (beneficio de slots en la fase ancha).

**3. ✅ RESUELTO** — ver sección 4. El reordenado espejo de la fase de decodificación NO era la
solución; la causa era un `rotScale` del rem sparse desalineado con el stride gigante del evaluador.

---

## 2. Ramas locales y qué contiene cada una

### `dev-pr` (base del usuario)
- `68d64abc` = HEAD original (Refactor EvalChebyPS limpio).
- `e7f65c74` = fix `rotScale` del rem sparse (p.g → p.gRem) + factor de fase `m==slots` en
  `CoeffEncodingOneLevel` y `CoeffDecodingOneLevel` (`ambos m==slots`).
- Bootstrap sparse: 59/59 PASS; en la suite completa solo fallan 4 tests pre-existentes:
  `BOOTSTRAP_NUM_TOWERS_05/06/13/14`.

### `dev-pr-fideslib-remaining` ✅ (ENTREGABLE — verificado)
- `7178e03e` (squash): aplica el resto de fideslib sobre dev-pr (13 archivos, +222/−45).
- `7250c9f4` = port del fix + régimen `ambos m==slots` (mismo contenido que `e7f65c74`).
- **Verificación**: `UnitTests/UTCKKSRNS_BOOT.*` + `UnitTests/UTCKKSRNS_SPARSE_KS.*` =
  **165/169 PASS** (solo los 4 fallos pre-existentes de dev-pr); `BOOTSTRAP_ITERATIVE` 28/28;
  `UTCKKSRNSCS_BOOT` 36/36; `BOOTSTRAP_SPARSE_11/31/52` 3/3.
- **Parche regenerado y verificado** (aplica limpio sobre `dev-pr` y produce una tree idéntica
  a la rama): `/tmp/opencode/dev-pr-fideslib-remaining.patch`.

### Ramas de experimento (ELIMINADAS tras la resolución)
`t2-sparse-m-slots`, `t3-decode-reorder`, `fideslib-pure`, `mtest-merge`, `t-sparse-precompute`,
`t1a-utils`, `t1b-ltpre`, `t1c-actualslots`, `t1d-setup`, `variant-fhe-devpr` y
`fideslib-add-sparse-rebase`. (La resolución vive ahora en `dev-pr`/`dev-pr-fideslib-remaining`.)

---

## 3. Historial de diagnóstico (para referencia)

Cruces empíricos del factor de fase antes del fix:
| config | encoding | decoding | SPARSE | FULL |
|---|---|---|---|---|
| dev-pr | m==2 | m==2 | ✅ | ✅ |
| ambos | m==slots | m==slots | ❌ | — |
| E2 | m==2 | m==slots | ❌ | ✅ |
| E3 | m==slots | m==2 | ✅ | ✅ |

Experimentos descartados en `t3-decode-reorder` (working tree, nunca pasó): "reordenado espejo"
de la vía sparse de `EvalSlotsToCoeffs` + `EvalSlotsToCoeffsPrecomp` (fase ancha/rem primero) —
basura en las 3 iteraciones. Datos instrumentados de SPARSE_11 (slots=8, lvlb=2,
`SelectLayers(3,2)` = layers=2, rows=1, rem=1): encoding `s=1`: shiftScale=2, offset=3,
rotScale=8, runningAcc=6; rem: offsetRem=1, gRem=2; decoding rem (espejo): shiftScaleRem=4,
rotScale=16, accRem=4, runningAcc tras rem=4; main s=0: shiftScale=1, offset=3, rotScale=4.

---

## 4. RESOLUCIÓN — causa raíz del fallo SPARSE con decoding `m==slots`

**Síntoma**: con el factor de fase `+i` en `m==slots` (fase ancha), el decoding sparse devolvía
basura; con `m==2` (fase estrecha) funcionaba. FULL nunca fallaba.

**Por qué fallaba (álgebra lineal del BSGS)**:
El evaluador `EvalSlotsToCoeffs` acumula cada fase con baby rotations `R^{j·scale}` y paso
gigante de Horner `R^{tRem}` con `tRem = scaleRem·p.gRem` en el bloque remainder. Para que el
paso gigante cancele el pre-rotado de las diagonales del precompute (`-rotScale·(h/gRem)`,
`h = gRem·i+j`), hace falta `rotScale = scaleRem·p.gRem` (cancelación exacta `+tRem·i−rotScale·i=0`).

El código fideslib sparse usaba `rotScale = shiftScaleRem·p.g`; con `g != gRem` (p.ej. g=4, gRem=2
en SPARSE_11) quedaba un twist residual `scaleRem·(gRem−g)·i` en las diagonales del rem. Ese twist
era inofensivo mientras el `+i` vivía en las filas de las fases principales (`m==2`), pero cuando
el `+i` se mueve a las filas del rem (`m==slots`), el mismatch corrompe la descomposición del rem.
La vía fully-packed ya usaba `rotScale = shiftScaleRem·p.gRem` (correcto), por eso nunca falló.

**Fix (1 línea + explicación)**: en `EvalSlotsToCoeffsPrecompute` (vía sparse),
`rotScale = shiftScaleRem * p.g` → `p.gRem`.
**Además**: `CoeffEncodingOneLevel` y `CoeffDecodingOneLevel` → factor `+i` en `m==slots`
(completa la optimización fideslib en encoding y decoding — decisión del usuario "ambos m==slots").

**Verificación final (en ambas ramas)**:
- `UnitTests/UTCKKSRNS_BOOT.*` + `UnitTests/UTCKKSRNS_SPARSE_KS.*`: **165/169 PASS**
  (4 fallos `BOOTSTRAP_NUM_TOWERS_05/06/13/14` pre-existentes de dev-pr).
- `*BOOTSTRAP_ITERATIVE*`: **28/28**; `*UTCKKSRNSCS_BOOT*`: **36/36**.
- Muestras objetivo: `BOOTSTRAP_SPARSE_11/31/52` PASS (3/3).

---

## 5. Cómo regenerar / verificar el parche

```bash
git checkout dev-pr-fideslib-remaining
# el parche es el diff de la rama sobre dev-pr:
git diff dev-pr dev-pr-fideslib-remaining            # contenido
# verificación de que aplica limpio sobre dev-pr:
git checkout -b test-apply dev-pr
git apply --3way /tmp/opencode/dev-pr-fideslib-remaining.patch
git diff dev-pr dev-pr-fideslib-remaining | diff - /tmp/opencode/dev-pr-fideslib-remaining.patch
```

---

## 6. Pendientes opcionales

1. **Regresión del resto del repo** si se desea seguridad total (correr `pke_tests` completo).
2. El fix/ambos `m==slots` ya está en `dev-pr` y `dev-pr-fideslib-remaining`; cualquier trabajo
   nuevo debe partir de cualquiera de las dos (no de las ramas de experimento, eliminadas).
