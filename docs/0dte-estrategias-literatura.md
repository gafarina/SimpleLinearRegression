# ¿Existe una estrategia "mejor" para 0DTE? Lo que dice la literatura

Respuesta corta: **no**. La literatura académica no identifica ninguna estrategia 0DTE
con retorno ajustado por riesgo positivo y robusto una vez que se contabilizan costos de
ejecución realistas. Lo que sí hay es un ordenamiento relativo claro, y una lección
metodológica contundente.

> **Nota metodológica.** SSRN, cdn.cboe.com, fma.org, arxiv.org y verdadcap.com están
> bloqueados por el proxy del entorno donde se compiló este documento. Las cifras marcadas
> con (*) provienen de abstracts o resúmenes de búsqueda y deben verificarse contra el PDF
> original. La excepción es la tabla de la sección 1, que sí se leyó directamente de la
> fuente primaria.

---

## 1. El resultado central: la corrección de costos de Vilkov (2026-08)

Este es el hallazgo más importante y más reciente sobre performance de estrategias 0DTE,
y **cambia el signo de las conclusiones publicadas**.

El replication package *"0DTE Trading Rules — Tail Risk, Implementation, and Tactical
Timing"* (Vilkov, SPXW 0DTE, 09/2016-01/2026, 7 tipos de estructura × configuraciones de
moneyness) contenía un **error de escala de unidades en los costos de transacción**:

- `bas`, `mid`, `tv` estaban en **fracción** del spot; `reth_und`, `reth` en **porcentaje**
  del spot. El código los combinaba directamente.
- Resultado: el **medio spread bid-ask se cobraba a 1/100 de su tamaño real** —
  ~0.022 bp en vez de ~2.2 bp. El fee fijo de 0.5 bp sí estaba bien escalado.
- El error fue reportado por el equipo de gex.live, que replicó la tabla publicada a
  cuatro decimales en las siete estructuras antes de diagnosticar la inconsistencia.

### Sharpe ratios incondicionales, antes y después de la corrección

| Estructura | SR mid (bruto) | SR neto (antes) | SR neto (**después**) |
|---|---|---|---|
| Put Ratio Spread | +1.06 | +0.84 | **−0.61** |
| Risk Reversal | +0.65 | +0.44 | **+0.10** |
| Bear Put Spread | +0.48 | +0.30 | **−0.73** |
| Strangle / Straddle | −0.27 | −0.51 | **−0.97** |
| Iron Butterfly / Condor | −0.56 | −0.96 | **−2.67** |

En el protocolo condicional (con timing táctico), sobre la muestra completa:
put ratio spread pasa de **+0.93 a −0.75**; la canasta top-3 de **+0.82 a −0.82**.

El medio spread medio pasa de 0.022 bp a 2.2 bp, y el turnover de entrada de 0.24 bp
aparentes a **24 bp reales** para un strangle de dos patas.

**Sutileza importante:** las columnas *brutas* también se mueven. El target binario
`y = 1{pnl_net > 0}` depende del costo, así que corregirlo **reetiqueta los datos de
entrenamiento** y el clasificador aprende una señal distinta y más débil. Quien reescale
el costo solo en la etapa de evaluación, sin reconstruir el target, va a obtener un
número más favorable que el correcto.

**Advertencia de estado:** el PDF del paper y los documentos bajo `docs/` del repo
**todavía no estaban actualizados** al momento de escribir esto. Las cifras optimistas
siguen circulando. Consultar siempre `KNOWN-ISSUES.md` antes de usar cualquier número.

https://github.com/vilkovgr/0dte-strategies/blob/main/KNOWN-ISSUES.md

---

## 2. Ordenamiento relativo: qué es "menos malo"

Aun con todos los Sharpe netos en terreno negativo o nulo, el ranking relativo es
informativo y consistente con la teoría:

1. **Risk reversal (+0.10)** — la única que sobrevive cerca de cero. Es una apuesta
   **direccional**, de dos patas, y captura la asimetría del VRP 0DTE.
2. **Put ratio spread (−0.61)** — mejor bruto de todas (+1.06), destruida por costos.
3. **Bear put spread (−0.73)**.
4. **Strangle / straddle (−0.97)** — venta pura de volatilidad.
5. **Iron butterfly / condor (−2.67)** — **la peor, por lejos**.

### El hallazgo incómodo

**La estructura más popular entre retail — el iron condor 0DTE — es la que peor
desempeño tiene en la literatura.** La razón es mecánica: cuatro patas × spread relativo
alto sobre primas muy bajas. El condor cobra la prima más pequeña y paga el costo de
transacción más grande. El "win rate" alto que reportan los brokers (63-72% (*)) es
exactamente lo que uno esperaría de una estrategia de payoff asimétrico — y es
irrelevante para el Sharpe.

### La lección estructural

Las estructuras **direccionales** (risk reversal, ratio spread) dominan a las de
**venta pura de volatilidad** (straddle, condor). Esto conecta con el resultado de
predictibilidad: el PNL 0DTE está impulsado por la **skewness realizada**, no por la
varianza realizada (que explica como máximo ~4.5% de R² cross-day).

---

## 3. Por qué la venta de volatilidad 0DTE no funciona neta

Tres papers independientes convergen al mismo diagnóstico:

**a) La prima existe pero es diminuta.** Bandi, Fusari y Renò documentan que vender
varianza 0DTE es rentable en promedio, pero la magnitud económica es pequeña porque el
valor temporal OTM a horas del vencimiento es pequeño — y difícil de monetizar tras
fricciones realistas (*). El VRP realizado mediano de 10:00 ET a cierre es ≈ **0.0011%
del subyacente**.
https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4503344

**b) Los spreads relativos son enormes.** Bogousslavsky y Muravyev: los trades 0DTE
pierden **4.7%** relativo a otros trades de opciones (t = −10), frente a +0.19% de los
no-0DTE (*). Causa mecánica: primas bajas ⇒ spread relativo alto.
https://www.lsu.edu/business/files/event-files/2025-finance-mardi-gras/retail_option_trading_v2.pdf

**c) La distribución de payoffs está dominada por la cola.** Las distribuciones de PNL a
nivel de estrategia son anchas, de cola pesada e **inestables entre regímenes**; para
muchas estructuras el riesgo a la baja es grande relativo al carry medio. No hay "carry
estable" que cosechar.

---

## 4. Qué sí funcionaba, y por qué dejó de funcionar

**Almeida, Freire y Hizmeri, "0DTE Asset Pricing".** La mayoría de los 0DTE viola las
cotas de precio asociadas a inversionistas aversos al riesgo. Una estrategia que explota
esas violaciones era **altamente rentable hasta 2022** — y **se disipa** con la
disponibilidad diaria de 0DTE, consistente con una integración creciente con el mercado
subyacente.
https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4701401

**Implicación directa:** cualquier backtest que abarque 2016-2022 sin cortar la muestra
va a mostrar un alfa que ya no existe. El régimen cambió cuando los vencimientos
pasaron a ser diarios (mayo 2022).

---

## 5. Evidencia sobre quién gana y quién pierde

- **Beckmeyer, Branger y Gayda (2023), "Retail Traders Love 0DTE Options... But Should
  They?"** Los retail son muy activos en 0DTE y en agregado pierden. Investigadores de
  Münster estiman pérdidas retail de **~$358.000 por día (~$90M anuales)** desde mayo
  2022 (*).
  https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4404704

- **Composición del mercado:** ~75% del trading retail en opciones SPX es 0DTE, pero
  ~94% del volumen 0DTE total es **institucional** (*). No estás compitiendo contra otros
  retail.

---

## 6. El único uso con beneficio económico documentado: cobertura, no alfa

**O'Donovan, "0DTE Options and the Price of Tail Protection".** Tras la extensión de los
vencimientos SPX a todos los días hábiles (16 de mayo de 2022), el skew de puts SPX se
**comprimió** en todos los plazos, con un pico de 71 bp a 30 días. La reducción anual
implícita en el costo del seguro de cola de renta variable es de **~USD 1.400 millones**
(*). Dos canales: sustitución del flujo de clientes hacia contratos puramente intradía en
el extremo corto, y caída de los costos de *warehousing* de los dealers, que absorben más
put-gamma en plazos intermedios.
https://papers.ssrn.com/sol3/papers.cfm?abstract_id=6836498

**Lectura:** la literatura documenta un beneficio económico real de los 0DTE **como
instrumento de cobertura barata**, no como fuente de alfa. Es el único resultado
claramente positivo y robusto del área.

---

## 7. ¿Y el timing táctico / condicional?

- Formular el problema como **clasificación direccional binaria** (signo del PNL) da
  resultados implementables mejores que predecir la magnitud del retorno.
- **Pero** el timing condicional tampoco salva la economía: post-corrección, el protocolo
  condicional lleva el put ratio spread a −0.75 y la canasta top-3 a −0.82.
- **Advertencia adicional:** el "model zoo" del mismo paquete usa un costo plano de
  0.5 bp que **excluye el medio spread por completo**. Como el medio spread promedia
  2.2 bp, esas cifras "netas" subestiman el costo realista en **~5x** y deben leerse
  como cota superior.

---

## Conclusión

| Pregunta | Respuesta de la literatura |
|---|---|
| ¿Hay una estrategia 0DTE mejor? | No hay ninguna con Sharpe neto positivo robusto. |
| ¿Cuál es la menos mala? | Risk reversal (SR neto +0.10) — direccional, 2 patas. |
| ¿Cuál es la peor? | Iron condor/butterfly (SR neto −2.67) — la más popular en retail. |
| ¿Direccional o vender vol? | Direccional. La skew manda, no la varianza. |
| ¿Sirve el timing? | Mejora el bruto; no alcanza para cruzar a positivo. |
| ¿Hay algún uso con valor? | Sí: **comprar** cobertura de cola, que se abarató ~$1.4bn/año. |
| ¿Qué mata todo? | El spread relativo. 24 bp de turnover de entrada en un strangle. |

**La lección metodológica** es quizás más valiosa que cualquiera de estos números: un
error de escala de un factor 100 en el costo de transacción convirtió cinco estrategias
perdedoras en ganadoras, pasó peer review parcial, y solo se detectó porque un tercero
replicó la tabla a cuatro decimales. En 0DTE, **el modelado de costos no es un detalle
de implementación — es el resultado**.

---

## Referencias

| Trabajo | Fuente | Aporte |
|---|---|---|
| Vilkov, "0DTE Trading Rules" + KNOWN-ISSUES | SSRN 4641356 / GitHub | Sharpe por estructura, corrección de costos |
| Bandi, Fusari, Renò (2023) | SSRN 4503344 | VRP existe pero no monetizable |
| Almeida, Freire, Hizmeri (2024) | SSRN 4701401 | Violación de cotas; alfa se disipa post-2022 |
| Beckmeyer, Branger, Gayda (2023) | SSRN 4404704 | Pérdidas retail |
| Bogousslavsky, Muravyev | Working paper | −4.7% en trades 0DTE |
| O'Donovan | SSRN 6836498 | Cobertura de cola más barata, ~$1.4bn/año |
| Dim, Eraker, Vilkov (2024) | SSRN 4692190 | Gamma de dealers y volatilidad |
