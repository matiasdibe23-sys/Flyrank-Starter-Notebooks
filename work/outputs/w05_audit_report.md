# Auditoría técnica — `w05_model.ipynb`

**Fecha:** 2026-10-12
**Método:** revisión estática del archivo completo (código, outputs guardados, metadata, widget state). El notebook no se re-ejecutó: no hay acceso a Colab ni al dataset de HF. Las cifras citadas son las guardadas en el notebook; sí verifiqué que sean coherentes entre sí (ver F2).

---

## 1. Veredicto

| Pregunta | Respuesta |
|---|---|
| ¿Las métricas 1.0000 son válidas? | **No.** El target es una función determinista de las features. Las métricas miden cuánto se parece un árbol a un `CASE`, no capacidad predictiva. |
| ¿Hay secretos expuestos? | **No hay secretos hardcodeados ni impresos.** Dos puntos menores (S1, S2). |
| ¿Se puede publicar en portafolio tal cual? | **No.** Hay 3 afirmaciones que contradicen el código o los outputs (F3, F4, F5). |

---

## 2. Resumen de hallazgos

| ID | Severidad | Hallazgo | Dónde |
|---|---|---|---|
| F1 | **Crítica** | Target definido con las mismas 3 variables que son features | Celda de carga SQL + `feature_cols` |
| F2 | **Alta** | ROC-AUC del baseline = 0.5 es un placeholder, no una medición. La comparación vs baseline es inválida | `eval_metrics` |
| F3 | **Alta** | El texto de interpretación contradice el output de permutation importance | Markdown sec. 4 vs output |
| F4 | **Alta** | El diseño de split declarado no coincide con el código | Markdown sec. 2 |
| F5 | Media | `leakage_note` del JSON exportado dice "feature matrix validated" | Celda de export |
| F6 | Media | Sin análisis de errores, sin CV, sin PR-AUC ni matriz de confusión | Sec. 3 y 4 |
| F7 | Baja | `fillna(0)` en `avg_position`, posición promedio sin ponderar, timestamp hardcodeado | Varias |
| S1 | Baja | Token inyectado en SQL por f-string y `except: pass` | Celda de carga |
| S2 | Baja | El self-check dice "no URLs" pero hay path del dataset interno | `PANEL_URL` |

---

## 3. Leakage: detalle

### F1 — Target reconstruible por regla (Crítica)

El target se define así:

```
impressions > 1000  AND  ctr < 0.015  AND  avg_position <= 20
```

Las features son `total_impressions`, `avg_ctr`, `avg_position` y `total_pageviews`. Las tres primeras son exactamente las variables de la regla, calculadas con las mismas agregaciones. El modelo no predice nada: reconstruye la regla.

Evidencia en el propio notebook:

- Tres condiciones AND se representan con un árbol de **profundidad 3**. Usaste profundidad 4 y 5, así que 1.0000 era el resultado esperado desde el diseño.
- `total_pageviews` (la única feature fuera de la regla) tiene importancia 0.0000.
- El Random Forest llega a 1.0000 con `max_depth=5`. Cualquier modelo con capacidad suficiente lo haría.

Consecuencia de negocio: un clasificador que predice una etiqueta definida por un SQL es estrictamente peor que el SQL (más lento, más mantenimiento, misma respuesta). Hoy el modelo no aporta valor incremental sobre la regla.

### F2 — Comparación vs baseline inválida (Alta)

```python
"ROC-AUC": round(roc_auc_score(y_true, y_proba) if y_proba is not None else 0.5, 4)
```

Al baseline se le pasa `y_proba=None`, así que su AUC sale como 0.5 por construcción. Pero `w04_baseline_score` es un score continuo: su AUC real se puede calcular y casi seguro es muy superior a 0.5. La tabla muestra un baseline "azar" que no lo es.

Chequeo de coherencia que hice: con positivos ≈ 14.06 % de 41 728 filas de test, precision 0.8713 y recall 0.9238 implican ≈ 6 220 predichos positivos, ≈ 800 FP y ≈ 450 FN. Eso da accuracy ≈ 0.970, que coincide con lo reportado. Las cifras son internamente consistentes; lo roto es el AUC.

Además, el umbral del baseline es `percentile(85)` del train: fija la tasa de predicción en ≈ 15 % usando el conocimiento de la prevalencia real (14 %). Es una ventaja menor, pero declárala.

### F4 — Claims de split que el código no sostiene (Alta)

| Afirmación en el markdown | Realidad en el código |
|---|---|
| "80/20 Stratified split **grouped by** `content_hash_id`" | Se usa `train_test_split(..., stratify=y)`. No es un split agrupado. Como el df ya está agregado a una fila por `content_hash_id`, agrupar por ese ID no protege nada |
| "eliminating data leakage into the evaluation phase" | No hay leakage de train a test (el split es limpio), pero sí leakage target↔features (F1). La frase da una falsa tranquilidad |
| "Snapshot Integrity: features from June 2026" | Features y target salen **del mismo mes**. No hay separación temporal, así que el modelo no predice el futuro |
| Plantilla del curso: "Grouped by client? Time-aware?" | Ninguna de las dos. No se carga columna de cliente, así que páginas del mismo cliente pueden caer en train y test |

### F6 — Rigor de evaluación (Media)

- La sección 4 pide "where is the model wrong". Con 0 errores no hay análisis, y no se reconoce esa ausencia como limitación.
- Una sola partición, sin validación cruzada y sin intervalos.
- Con 14 % de positivos falta PR-AUC, matriz de confusión y precision@K (la métrica operativa real: "de las K páginas que mando a refrescar, cuántas lo necesitaban").

---

## 4. Consistencia narrativa vs outputs

### F3 — Importancia de features (Alta)

El markdown afirma: *"`avg_ctr` and `total_impressions` are the dominant split features, validating our domain thesis"*.

El output dice:

| Feature | Importancia |
|---|---|
| total_impressions | 0.1915 |
| avg_position | 0.0658 |
| avg_ctr | **0.0153** |
| total_pageviews | 0.0000 |

`avg_ctr` es la **tercera**, no la dominante, y `avg_position` pesa 4× más. El texto se escribió antes de ver el output o sin leerlo. Además, esta importancia solo refleja qué condición de la regla es más "binding" en los datos, no importancia causal ni de negocio: no valida ninguna tesis de dominio.

### F5 — Receipt JSON engañoso (Media)

`"leakage_note": "Rule reconstruction achieved 1.0 metrics; feature matrix validated."`

"Feature matrix validated" es falso: la matriz contiene el leakage. Este JSON está pensado para el portafolio, así que propaga la afirmación. También lleva `ROC-AUC: 0.5` para el baseline (F2) y `run_timestamp` hardcodeado ("2026-10-08").

### F7 — Detalles técnicos (Baja)

- `.fillna(0)` sobre `avg_position`: posición 0 no existe y significaría "mejor que #1". Hoy no afecta al target (exige impresiones > 1000), pero contamina la feature si hay NULLs. Usa indicador de missing o filtra.
- `AVG(gsc_avg_position)` no está ponderado por impresiones. Un día con 5 impresiones pesa igual que uno con 5 000. Usa `SUM(position * impressions) / SUM(impressions)`.
- `run_timestamp` debe ser `datetime.now()`.

---

## 5. Secretos y datos sensibles

Alcance: celdas de código, outputs, metadata del notebook y widget state. Revisión manual del archivo completo, buscando patrones de `hf_`, `sk-`, `ghp_`, `Bearer`, `api_key`, `password`, `secret`, `token`.

**Resultado: no hay claves ni tokens expuestos.** El token se lee de Colab Secrets (`userdata.get('HF_TOKEN')`), no se imprime y no aparece en ningún output. El widget state solo contiene estilos de una barra de progreso.

### S1 — Manejo del token (Baja)

```python
con.execute(f"CREATE SECRET hf_auth (TYPE HTTP, BEARER_TOKEN '{hf_token}');")
...
except Exception as e:
    pass
```

- El token se interpola en el texto SQL. Si DuckDB lanzara un error, el mensaje podría incluir la sentencia con el token. Aquí queda tapado por `pass`, pero la combinación es frágil: el día que quites el `except`, un traceback guardado en el notebook filtraría el token.
- `except Exception: pass` oculta fallos de autenticación. Si el secret no se crea, la lectura puede fallar más adelante con un error confuso, o leer solo datos públicos sin que lo notes.
- Mejora: capturar solo `userdata.SecretNotFoundError` y registrar el fallo, sin interpolar el token en un string que pueda imprimirse. Si el dataset es público, quita el bloque entero.

### S2 — Path interno y claim del self-check (Baja)

- `PANEL_URL = "hf://datasets/FlyRank/internship-warehouse/..."` expone el nombre de la organización y del warehouse.
- El self-check marca "No client names, URLs, or private queries anywhere". Esa casilla es falsa mientras el path esté en el notebook.
- La URL del badge de Colab revela tu usuario y repo de GitHub. Es normal si el repo es público; confirma que lo es.
- Mejora: mover el path a una variable de entorno o a Colab Secrets, o confirmar que el dataset es público y quitar la casilla del self-check.

---

## 6. Plan de remediación (por prioridad)

### P0 — Confirmar el diagnóstico (15 min)

```python
from sklearn.tree import export_text
dt3 = DecisionTreeClassifier(max_depth=3, random_state=42).fit(X_train, y_train)
print(export_text(dt3, feature_names=feature_cols))   # debería mostrar 1000 / 0.015 / 20
print("acc depth=3:", dt3.score(X_test, y_test))

# Ablación: ¿qué queda sin las variables de la regla?
dt_abl = DecisionTreeClassifier(max_depth=4, random_state=42).fit(X_train[['total_pageviews']], y_train)
print("acc solo pageviews:", dt_abl.score(X_test[['total_pageviews']], y_test), "| baseline mayoría:", 1 - y_test.mean())

# Baseline con AUC real
from sklearn.metrics import roc_auc_score, average_precision_score
s = df_raw.loc[idx_test, 'w04_baseline_score']
print("baseline AUC:", roc_auc_score(y_test, s), "| PR-AUC:", average_precision_score(y_test, s))
```

Si el depth-3 da 1.0000 y los umbrales coinciden con el SQL, F1 queda demostrado con evidencia.

### P1 — Rediseñar la pregunta (aquí está el valor real)

Elige una de dos rutas y dilo explícito en el notebook:

**Ruta A — Predicción real (recomendada).** El target no puede derivarse de las features del mismo periodo.
- Features del mes *t* (p. ej. mayo), target del mes *t+1* (p. ej. caída de clics > X % o pérdida de posición).
- Split temporal (entrena en meses ≤ *t*, prueba en *t+1*). Si hay columna de cliente, `GroupShuffleSplit` por cliente.
- Compara contra dos baselines: regla SQL actual y tasa base.

**Ruta B — Destilación de reglas.** Si el objetivo es "el árbol redescubre la regla", dilo así.
- No lo vendas como modelo predictivo ni reportes accuracy como logro.
- El entregable es el árbol extraído (`export_text`) y la verificación de que coincide con el SQL.

### P2 — Corregir evaluación

- Calcular AUC real del baseline (código en P0).
- Añadir PR-AUC, matriz de confusión, precision@K y recall@K con K = capacidad real del equipo de contenido.
- Validación cruzada estratificada (5 folds) con media ± desviación.
- Si hay errores, analizar los 20 más confiados y los falsos negativos; si no hay errores, declararlo como limitación, no como resultado.

### P3 — Corregir texto y artefactos

- Reescribir la sección de importancia con los números reales (F3).
- Reemplazar `leakage_note` por: `"Target is a deterministic function of 3 features; metrics reflect rule reconstruction, not predictive performance."`
- Corregir las afirmaciones de split (F4) y la casilla del self-check (S2).
- `run_timestamp = datetime.now().isoformat()`.

### P4 — Métrica de impacto de negocio

Sustituye accuracy por algo que el cliente pueda valorar:

```
clics_recuperables ≈ Σ_{páginas marcadas} impresiones × (CTR_benchmark(posición) − CTR_actual)
```

Reporta: de las K páginas priorizadas, cuántas clics/mes estimados se recuperan frente a la priorización del baseline, con el supuesto de CTR benchmark explícito. Etiquétalo como estimación direccional, no como resultado medido.

---

## 7. Preguntas que debes poder responder antes de presentar esto

1. ¿Qué decisión concreta toma el equipo con este modelo que no tome ya con el `CASE`?
2. ¿Cuál es la tasa de acierto del modelo en el mes siguiente, sobre páginas que el modelo no vio?
3. ¿Qué pasa si el umbral del negocio cambia de 0.015 a 0.02? El modelo tendría que reentrenarse; la regla, no.
4. ¿Cuánto cuesta (tiempo de equipo) refrescar una página y cuántos clics debe recuperar para pagarse?

## 8. Ángulo para el portafolio

Un 1.0000 sin contexto es una señal de alarma para cualquier revisor técnico. "Detecté leakage determinista en mi propio target, lo demostré con un árbol depth-3 y rediseñé el problema con split temporal" es más sólido que cualquier métrica perfecta. Documenta el antes y el después con los números.
