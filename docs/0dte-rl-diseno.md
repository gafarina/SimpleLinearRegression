# RL para entrada y reparación de estrategias 0DTE: diseño y viabilidad

Nota de diseño sobre aplicar reinforcement learning a decisiones de entrada y ajuste
("reparación") de estructuras 0DTE, usando cadenas de opciones SPY al minuto.

---

## 0. La restricción que define todo

No es el algoritmo. Es la relación **edge / costo** y el **número de episodios independientes**.

### 0.1 Edge vs costo

De la literatura (ver `0dte-estrategias-literatura.md`):

- VRP realizado **mediano** de 10:00 ET a cierre ≈ 0.0011% del subyacente ≈ **0.11 bp**.
- Medio spread medio en SPXW 0DTE ≈ **2.2 bp**. Turnover de entrada de un strangle de
  dos patas ≈ **24 bp**.

El costo de **una** cruzada de spread es ~20x el edge mediano. Un agente con libertad
para reparar va a ejecutar más, no menos.

**Consecuencia de diseño:** el RL no puede fabricar edge, solo asignarlo. La única
política ganadora posible es **operar poco, en estados de alta convicción**. Eso convierte
el problema en *optimal stopping* con acción esparsa, no en trading continuo. El espacio
de acciones debe estar sesgado hacia "no hacer nada" por construcción.

### 0.2 Episodios independientes

| Fuente de decisión | N disponible |
|---|---|
| Decisiones de **entrada** (1/día, 0DTE diarios desde 05/2022) | **~1.000** |
| Decisiones de **gestión** (390 min × ~1.000 días) | **~390.000** |
| Regímenes de mercado genuinamente distintos en 4 años | **~10** |

PPO/DQN típicamente necesitan 10⁵-10⁷ episodios. Con ~1.000 episodios no iid, el RL
model-free sobre la decisión de entrada **va a sobreajustar con certeza**.

---

## 1. La decisión arquitectónica central

**Separar el problema en dos, porque tienen regímenes de datos opuestos.**

| Sub-problema | Datos | Método recomendado |
|---|---|---|
| **A. Timing de mercado / entrada** | ~1.000 muestras | Clasificador binario o contextual bandit, fuertemente regularizado. **No RL.** |
| **B. Gestión / reparación condicional a la posición** | ~390.000 transiciones, ampliable a millones | **Aquí sí RL.** |

La justificación: la política de gestión es *condicional a la posición*. Con la cadena
completa al minuto puedes **generar trayectorias contrafactuales**: inicializar en cada
día histórico muchas combinaciones de (hora de entrada × estructura × strikes × tamaño) y
rodarlas hacia adelante con precios reales. Mil días se convierten en millones de
trayectorias de gestión.

**Límite honesto de esa augmentación:** todas comparten las mismas ~1.000 trayectorias de
precio. El tamaño muestral efectivo para generalizar sobre *regímenes de mercado* sigue
siendo ~1.000. El agente puede aprender bien "dado este estado de posición y de mercado,
qué acción conviene"; **no** puede aprender "qué regímenes son favorables". Por eso A se
mantiene paramétrico y simple.

---

## 2. Gate previo: el estudio de costos (hacer esto ANTES de escribir RL)

Un día de trabajo que decide si el resto del proyecto tiene sentido. Con la cadena al
minuto, calcular sobre **tus propios datos**:

1. Distribución del medio spread `(ask − bid)/2`, expresado en **bp del spot**,
   estratificada por: moneyness, hora del día, y nivel de VIX1D.
2. Lo mismo para el **combo** de cada estructura (suma sobre patas), que es el costo real
   de entrada.
3. Costo por **reparación** típica (cerrar 1 pata + abrir 2 = 3 cruzadas).
4. Fees por contrato. En SPY esto pesa relativamente más que en SPX: el notional por
   contrato es ~1/10, así que el fee fijo por contrato es ~10x en términos relativos.

**Criterio de decisión:** si el costo de ida y vuelta de tu estructura objetivo supera el
percentil ~70 de la distribución de PNL bruto diario, el proyecto es inviable tal cual y
hay que cambiar de estructura o de frecuencia antes de modelar nada.

### Regla de higiene de unidades (no negociable)

El error más caro documentado en esta literatura fue de escala: mezclar una variable en
**fracción** del spot con otra en **porcentaje** del spot hizo que el medio spread se
cobrara a 1/100 de su tamaño, convirtiendo cinco estrategias perdedoras en ganadoras.

Implementar desde el primer commit:

```python
def assert_bp_of_spot(x, name):
    """Falla ruidosamente si una serie de costos sale de la banda plausible en bp."""
    med = np.nanmedian(np.abs(x))
    assert 0.05 < med < 100, f"{name}: mediana {med:.4f} bp fuera de banda — revisar escala"
```

Llamarla en **todos** los caminos que tocan costos. Una sola convención de unidades en
todo el repo (recomendado: bp del spot), documentada en el README.

---

## 3. Formulación del MDP (sub-problema B)

### Estado

**Posición** (lo que hace el problema Markov):
`delta_neta, gamma_neta, vega_neta, theta_neta, pnl_mtm_actual, n_patas_abiertas,
n_reparaciones_hechas, distancia_a_strikes_cortos (en σ intradía), tamaño`

**Mercado** (reusar los predictores de `0dte-predictores-literatura.md`):
`minutos_hasta_cierre` (crítico — el perfil gamma/theta es determinista en t),
`IV_ATM_0DTE` o VIX1D corregido por sesgo overnight, `RV_intradía_acumulada`,
`retorno_primeros_30min`, `pendiente y skew de la superficie 0DTE`,
`proxy GEX` (gamma ponderada por open interest de la cadena), `ancho de spread actual`,
`dummy de anuncio macro` + minutos desde el anuncio.

**Normalización:** todo en unidades invariantes al nivel (bp de spot, σ intradía), no en
dólares. Si no, el agente aprende el nivel del SPY, no la dinámica.

### Acciones

Discretas y sesgadas hacia la inacción:

```
0: mantener                      <- debe ser la acción modal por amplio margen
1: cerrar todo
2: cerrar la pata tensionada
3: rodar la pata no tensionada más cerca
4: convertir condor -> mariposa
5: delta-hedge con subyacente (shares)
6: añadir pata de cobertura de cola
```

Cada acción ≠ 0 dispara el costo real de la cadena en ese minuto. Sin excepción.

### Reward

**No maximizar E[PNL].** Un payoff dominado por colas bajo E[PNL] produce una política
short-gamma que se ve espectacular hasta que no.

Opciones, en orden de preferencia:

1. **Utilidad exponencial** (formulación de deep hedging de Buehler et al. 2019): optimiza
   una medida de riesgo convexa, es la referencia estándar del área.
2. **Mean-CVaR** con α = 0.95, siguiendo la línea de *risk-averse RL* para hedging.
3. E[PNL] − λ·downside — solo como baseline.

**Estructura temporal:** reward denso de mark-to-market por minuto (la recompensa
terminal pura es demasiado esparsa y de varianza alta para ~390 pasos), con el **costo
cargado íntegro en el minuto de la transacción**, no amortizado.

---

## 4. Entorno: replay vs simulador

Tener la cadena completa al minuto permite **replay exacto**: bid/ask reales de cada pata
en cada minuto, sin necesidad de modelo de pricing. Es la gran ventaja de tu dataset.

### Supuestos de fill que hay que declarar explícitamente

- **Banda de ejecución.** Cruzar el spread completo es conservador; asumir mid es
  optimista. Reportar **ambos extremos** como bracket en todo resultado. Si la conclusión
  cambia de signo entre mid y bid/ask, no hay conclusión.
- **Sin impacto de mercado.** Válido en tamaño chico; el backtest no lo valida.
- **Quotes rancias o cruzadas** en OTM profundo cerca del cierre: filtrar, no interpolar.
- **Feedback de gamma no capturado.** Tus reparaciones están correlacionadas con las de
  todos los demás. El replay es ciego a esto por construcción.

### Riesgo específico de SPY (vs SPX)

- **Americana + liquidación física ⇒ riesgo de asignación temprana** en cortos ITM,
  agudo en fechas ex-dividendo. Un agente entrenado en replay no lo verá nunca y va a
  aprender políticas que lo ignoran. Penalizar explícitamente los cortos ITM cerca del
  cierre en días ex-div.
- **Fees relativos ~10x** por el notional menor por contrato.
- SPX es europea, cash-settled, sin asignación. Si el objetivo final es producción,
  evaluar si el dataset correcto no es SPXW.

### Complemento recomendado

La literatura de deep hedging entrena sobre **paths simulados** y valida en datos reales.
Un híbrido razonable: pre-entrenar en un simulador (rough vol + saltos — recordar que a
vencimiento cero domina el **salto**, no la difusión) y hacer fine-tuning/validación en
replay. Mitiga parcialmente la escasez de regímenes.

---

## 5. Controles de sobreajuste (con ~1.000 días, esto es el proyecto)

1. **Purged walk-forward CV con embargo** (López de Prado). Nunca shuffle aleatorio.
2. **Holdout final intocable**: reservar el último año y evaluarlo **una sola vez**.
3. **Corte de régimen en mayo 2022** (vencimientos diarios). Resultados pre y post por
   separado; el alfa de violación de cotas de precio se disipa después de 2022.
4. **Contar y reportar el número de configuraciones probadas.** Aplicar Deflated Sharpe
   Ratio. Con 1.000 episodios y 50 configuraciones, un SR de 1.0 in-sample no es evidencia.
5. **Jerarquía de baselines.** El agente debe superar, en este orden:
   - no operar (SR = 0),
   - estructura estática a hora fija, sin reparaciones,
   - política de reparación basada en reglas (ej. cerrar a −2x prima),
   - clasificador logístico sobre el signo del PNL.

   Si no supera al #3, es ruido. El resultado publicado relevante es que la clasificación
   binaria direccional supera a la predicción de magnitud — ese es el listón real.

6. **Semillas múltiples.** RL con pocos datos tiene varianza enorme entre semillas.
   Reportar la distribución sobre ≥10 semillas, no la mejor corrida.

---

## 6. Expectativa honesta del resultado

Dado que ninguna estructura conserva Sharpe neto positivo tras costos correctos, el
resultado **más probable** es que el agente aprenda que la política óptima es operar muy
poco o no operar. Eso no es un fracaso: es información, y es la respuesta correcta al
ratio edge/costo.

Los dos resultados genuinamente valiosos que este montaje puede producir:

1. **¿Existe un subconjunto de estados donde el edge supera el costo?** Un clasificador
   con alta precisión y bajísimo recall que opere 20 días al año puede ser económicamente
   viable donde una política diaria no lo es.
2. **¿Las "reparaciones" agregan o destruyen valor?** Buena parte del folklore de ajustes
   consiste en añadir riesgo para no realizar una pérdida. Un agente con objetivo
   riesgo-averso muy probablemente aprenda a **cortar** en vez de reparar. Eso es
   contrastable, y es un resultado publicable con tus datos.

La pregunta 2 es, en mi opinión, la más interesante y la que mejor se ajusta a lo que
permite el dataset.

---

## 7. Orden de trabajo sugerido

1. Estudio de costos + guardas de unidades (**gate**).
2. Motor de replay con contabilidad exacta de patas y costos; tests contra PNL calculado a
   mano en un puñado de días.
3. Baselines: estática, reglas, logístico. Con purged CV.
4. Augmentación contrafactual del sub-problema de gestión.
5. RL riesgo-averso solo sobre gestión, condicionado a posición.
6. Composición: entrada paramétrica + gestión aprendida.
7. Holdout, una vez.

---

## Referencias

| Trabajo | Aporte |
|---|---|
| Buehler, Gonon, Teichmann, Wood (2019), "Deep Hedging" | Formulación de referencia: redes + medida de riesgo convexa + fricciones |
| [Deep Hedging with RL: A Practical Framework (arXiv 2512.12420)](https://arxiv.org/abs/2512.12420) | Entorno leak-free, reward consciente de costos, actor-critic |
| [Deep RL Algorithms for Option Hedging (arXiv 2504.05521)](https://arxiv.org/pdf/2504.05521) | Comparativa de algoritmos; PPO gana en eficiencia muestral |
| [Option Hedging with Risk Averse RL (arXiv 2010.12245)](https://arxiv.org/pdf/2010.12245) | Objetivos CVaR para hedging |
| [Robust Risk-Aware Option Hedging (arXiv 2303.15216)](https://arxiv.org/pdf/2303.15216) | Robustez frente a mala especificación del modelo |
| Vilkov, KNOWN-ISSUES.md | El error de escala de costos y por qué las guardas son obligatorias |
| López de Prado, *Advances in Financial ML* | Purged CV, embargo, Deflated Sharpe Ratio |
