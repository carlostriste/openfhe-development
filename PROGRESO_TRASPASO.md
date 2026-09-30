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

---

## 7. Port a OpenFHE v1.6.0 (rama `v1.6.0-fideslib`)

Base: tag `v1.6.0` (`6206d24f`). Se portó el **delta completo** `68d64abc..dev-pr-fideslib-remaining`
(5 commits cherry-picked con `-x`), resolviendo 42 conflictos en 11 archivos.

- Criterio: conservar sólo cambios funcionales; en logging/estilo/formato se prefirió upstream
  (p. ej. `OPENFHE_DIAGNOSTIC_ERR` en lugar de `std::cout/cerr`).
- Se descartaron del commit `7178e03e` los cambios no funcionales de `CMakeLists.txt` y el `.rst`.
- Se limpiaron los marcadores de conflicto que la rama traía commiteados
  (`ckksrns-fhe.cpp` en `#ifdef BOOTSTRAPTIMING` y `cmake_in_openfhe.rst`) y restos de depuración.
- `m==slots` en `ckksrns-utils.cpp` queda aplicado (v1.6.0 aún tenía `m == 2`); el
  `rotScale = shiftScaleRem * p.gRem` ya lo trae 1.6.0.
- Se restauraron los `OPENFHE_THROW` de `GetScalingFactorReal` y `GetScalingFactorRealBig` (1.6.0
  los había sustituido por un retorno silencioso de `m_approxSF`), y en `GetScalingFactorRealBig`
  el fallback vuelve a ser `m_approxSF * m_approxSF` (correcto para grado 2).

### Adaptación de los accessors FBT (commit `26882db5`)
1.6.0 retuneó/renombró las tablas de functional bootstrapping. Mapeo por rol:
```
coeff_cos_16_double    -> coeff_cos_16_double_50    (SPARSE_ENCAPSULATED cos)
coeff_cos_25_double    -> coeff_cos_28_double_68    (default K=28 cos)
coeff_cos_512_double   -> coeff_cos_672_double_104  (UNIFORM_TERNARY cos)
coeff_exp_25_double_58 -> coeff_exp_28_double_69    (default K=28 exp)
coeff_exp_25_double_66 -> coeff_exp_28_double_69    (default K=28 exp; 1.6.0 ya no
                                                     distingue por digitBitSize)
```
**A confirmar**: `Exp58`/`Exp66` colapsan a la misma tabla 1.6.0; verificar que FIDESlib no
necesite distinguirlas.

### Verificación
- `pke_tests` compila completo con la rama.
- `UTCKKSRNS_SPARSE_KS.*` + `UnitTests/UTCKKSRNS_BOOT.*` + `UnitTests/UTCKKSRNSCS_BOOT.*`:
  **222/222 PASS** (incluye los 4 `BOOTSTRAP_NUM_TOWERS_05/06/13/14` que fallaban en dev-pr;
  en 1.6.0 ya pasan).
- `UnitTests/UTCKKSRNS_EVAL_POLY.*` + `DegreeTwoConstantsChebyshev119`: **84/84 PASS**.
- **Fallo pre-existente (NO del port)**: v1.6.0 aborta en el destructor global de `libOPENFHEcore`
  con `double free or corruption (!prev)` al terminar *cualquier* test (también un test BFVRNS sin
  relación con estos cambios). Los tests pasan; el abort ocurre en `__cxa_finalize`. Correr bajo
  `gdb` evita el bloqueo del manejador de aborto posterior.

### Refactor EvalChebyPS (`68d64abc`) — commit `d91b8ff8`
Es la base canónica de `dev-pr` (limpieza del prototipo FIDESlib) y **no** estaba en 1.6.0. Aporta:
- `InnerEvalChebyshevPS(..., int32_t level_offset)` y `EvalPartialLinearWSumWithBias` (nuevo orden
  `Rescale(cu*qu + su)`), preámbulo de rango de entrada, y menos `adjustForMult/Add`.
- API `target_level` completa: `EvalMult`/`EvalMultInPlace`/`EvalLinearWSum(Mutable)` en
  `cryptocontext.h`, `base-scheme.h`, `base-leveledshe.h`, `base-advancedshe.h`, `ckksrns-leveledshe.*`,
  `ckksrns-advancedshe.*`, y `GetElementForEvalMult(..., scaling_factor)`.
- (Antes del port, `target_level` solo estaba a medias: la firma existía pero se ignoraba; ahora está
  cableado.)

**Nota de rendimiento (a revisar)**: aplicado sobre 1.6.0, 43 de los 222 tests de bootstrap/CS/SPARSE_KS
se ralentizan >2× respecto a 1.6.0 sin el refactor (p. ej. `BOOTSTRAP_SERIALIZE_01` 397 ms → 21.4 s,
`BOOTSTRAP_NUM_TOWERS_18` 158 ms → 4.9 s); la mediana apenas cambia (~1.02×). Es el comportamiento
canónico de dev-pr; si molesta, habría que reconciliar el refactor con las optimizaciones propias de
1.6.0 (la rama `56499713`/`a957c3c0`).
