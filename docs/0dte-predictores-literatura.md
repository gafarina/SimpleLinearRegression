# Predictores para 0DTE: qué dice la literatura

Revisión de la evidencia académica sobre qué variables tienen poder predictivo en el
contexto de opciones con vencimiento el mismo día (0DTE) sobre el S&P 500.

> **Nota metodológica.** Varias fuentes primarias (SSRN, cdn.cboe.com, fma.org) no son
> accesibles desde el entorno donde se compiló este documento; las cifras marcadas con (*)
> provienen de resúmenes/abstracts y deberían verificarse contra el PDF original antes de
> citarse en un paper o en un comité de riesgo.

---

## 0. Primero: ¿qué se quiere predecir?

"Predecir un 0DTE" no es un único problema. La literatura trata al menos cuatro targets
distintos, y **los mejores predictores cambian según el target**:

| Target | Variable dependiente típica | Dificultad |
|---|---|---|
| **A. Dirección** del subyacente intradía | signo del retorno SPX de `t` a cierre | Muy alta (R² ~1-3%) |
| **B. Magnitud / volatilidad realizada** del día | RV intradía, rango alto-bajo | Moderada (R² 40-70%) |
| **C. Retorno de una estructura 0DTE** | PNL de straddle, put ratio spread, etc. | Alta, pero mejor que A |
| **D. Terminal / pinning** | P(ITM), strike de cierre | Moderada, efecto pequeño |

La conclusión transversal de la literatura: **B es predecible, A casi no lo es, y C es
predecible principalmente a través de la asimetría (skew) realizada, no de la varianza.**

---

## A. Predictores de dirección intradía

### A.1 Momentum intradía (el predictor clásico)

- **Gao, Han, Li y Zhou (2018), "Market intraday momentum", *Journal of Financial
  Economics*.** El retorno de la primera media hora (medido desde el cierre previo)
  predice positivamente el retorno de la última media hora en SPY. Pendiente escalada
  6.94, significativa al 1%, **R² ≈ 1.6%** — alto para un predictor de retornos.
  La predictibilidad es **más fuerte en días de alta volatilidad, alto volumen,
  recesión y días de publicación macro relevante**. Se replica en otros 10 ETFs.
  https://www.sciencedirect.com/science/article/abs/pii/S0304405X18301351

- **Baltussen, Da, Lammers y Martens (2021), JFE.** Extiende el momentum intradía a
  evidencia global en futuros, con vínculos al rebalanceo de ETFs apalancados.
  https://centaur.reading.ac.uk/95566/1/Accepted-Version.pdf

### A.2 Advertencia: el efecto puede haberse degradado en la era 0DTE

Ambos papers usan muestras que **terminan antes** de que los 0DTE fueran ~50-60% del
volumen SPX. Un estudio no académico sobre 1.085 sesiones SPX (2022-04 a 2026-08)
reporta momentum intradía plano en el agregado y plano año a año. No está peer-reviewed,
pero la dirección es consistente con el mecanismo del punto A.3: el hedging de gamma de
los dealers **amortigua** las tendencias intradía.
https://dev.to/firmtape/intraday-momentum-is-dead-in-the-0dte-era-we-measured-it-on-1085-spx-sessions-43g0

### A.3 Gamma de los dealers (GEX) — el predictor específico de la era 0DTE

- **Dim, Eraker y Vilkov (2024), "0DTEs: Trading, Gamma Risk and Volatility
  Propagation".** El inventario de los market makers, medido como **gamma neta de sus
  posiciones**, es en promedio **positivo** y está **negativamente relacionado con la
  volatilidad intradía futura**. Es decir: gamma neta positiva ⇒ hedging contrarian ⇒
  menor volatilidad y mayor reversión del order flow.
  https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4692190

- **Adams, Fontaine y Ornthanalai (2024), "0DTE Index Options and Market Volatility:
  How Large is Their Impact?" (Cboe research).** La volatilidad de mercado es **menor**
  en días con 0DTE disponibles. El impacto máximo de la gamma de los OMM sobre la
  volatilidad anualizada en ventanas de 30 minutos es de **+6.4 puntos porcentuales** (*),
  y el efecto depende del **signo y magnitud** de la posición de gamma.
  https://cdn.cboe.com/resources/education/research_publications/gammasqueezes.pdf

  **Implicación práctica:** el GEX no es un predictor direccional; es un predictor del
  *régimen de volatilidad y de reversión*. Úsalo como interacción / variable de estado,
  no como señal de compra-venta.

### A.4 Order flow imbalance en opciones

- **"Information content of order imbalance in the index options market"** (*Int. Review
  of Financial Analysis*, 2021). Mayor desbalance de órdenes en calls se asocia con
  retornos spot contemporáneos más altos, pero **predice negativamente** el retorno del
  índice la semana siguiente. El canal es la demanda de cobertura de las contrapartes.
  https://www.sciencedirect.com/science/article/abs/pii/S1059056021002367

- **"Option Volume Imbalance as a predictor for equity market returns"** (arXiv 2201.09319).
  https://arxiv.org/pdf/2201.09319

- **"Returns and Order Flow Imbalances: Intraday Dynamics and Macroeconomic News Effects"**
  (arXiv 2508.06788). VAR estructural identificado por heterocedasticidad, a frecuencia de
  1 segundo, por cada intervalo de 15 minutos, en futuros E-mini S&P 500. Es la referencia
  metodológica correcta si se quiere modelar la endogeneidad precio↔flujo intradía.
  https://arxiv.org/pdf/2508.06788

### A.5 Anuncios macro

- **Predictibilidad tras anuncios de política monetaria** (SPY 1-minuto, 1997-2025):
  el pico de predictibilidad ocurre ~26 minutos después del anuncio y la tendencia de
  reacción dura ~49 minutos (*). Esto es exactamente la ventana en la que un 0DTE tiene
  gamma máxima.
  https://research.library.fordham.edu/gabelli_thesis/182/

---

## B. Predictores de volatilidad / magnitud del movimiento

Este es el terreno donde la evidencia es más sólida.

### B.1 VIX1D

- **Albers (2025), "A New Star Is Born: Does the VIX1D Render Common Volatility
  Forecasting Models for the US Equity Market Obsolete?", *Journal of Futures Markets*.**
  El VIX1D (construido con SPX weeklies 0DTE y 1DTE) tiene relación positiva con la
  volatilidad realizada futura, y **HAR-RV-VIX1D supera a HAR-RV-VIX** para horizontes
  cortos.
  https://onlinelibrary.wiley.com/doi/full/10.1002/fut.70023

- **"The daily rise and fall of the VIX1D: Causes and solutions of its overnight bias"**
  (*Finance Research Letters*, 2024). **Crítico si se usa VIX1D como feature:** el índice
  tiene un sesgo intradía sistemático por el tratamiento del período overnight. Hay que
  corregirlo o usar efectos fijos por hora del día.
  https://www.sciencedirect.com/science/article/pii/S1544612324002162

- **"Do short-term market swings improve realized volatility forecasts?"** (*FRL*, 2023).
  Los swings de corto plazo mejoran significativamente el pronóstico de RV de corto plazo.
  https://www.sciencedirect.com/science/article/pii/S1544612323010012

### B.2 Riesgo de salto (jump risk) — el componente dominante en 0DTE

- **Bandi, Fusari y Renò (2023-2024), "0DTE Option Pricing".** Fórmula de valoración
  específica para 0DTE; el modelo ajusta mejor que las alternativas sobre datos
  intradiarios (2014-01-02 a 2023-05-11). El punto clave: a vencimiento cero, la
  distribución está dominada por el componente de **salto**, no por la difusión.
  https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4503344

- **Božović, "Intraday Jumps and 0DTE Options: Pricing and Hedging Implications".**
  Con 22.410 observaciones de 5 minutos de SPXW 0DTE, la prima implícita por riesgo de
  salto es significativa y **casi el doble** de las primas combinadas de difusión y
  volatilidad (*).
  https://papers.ssrn.com/abstract=5223127

  **Implicación práctica:** cualquier modelo lineal/gaussiano sobre 0DTE va a estar mal
  especificado en las colas. Es la razón por la que los modelos de clasificación binaria
  funcionan mejor que los de regresión de magnitud (ver C.2).

### B.3 Baseline obligatorio

HAR-RV (Corsi 2009) sobre RV intradía, más el precio del straddle ATM al abrir como
"movimiento implícito". Cualquier feature nuevo debe superar este baseline.

---

## C. Predictores del PNL de una estructura 0DTE

### C.1 Prima de riesgo de varianza (VRP)

- **Almeida, Freire y Hizmeri (2024-2025), "0DTE Asset Pricing".** Resultado central:
  en el mercado 0DTE los inversionistas exigen **más compensación por retornos positivos
  que por negativos** — el VRP está dominado por la compensación por riesgo *al alza*, y
  **predice negativamente los retornos de mercado**. Además, la mayoría de los 0DTE viola
  las cotas de precio de inversionistas aversos al riesgo; una estrategia que explota esa
  violación era muy rentable hasta 2022 y **se disipa** con la disponibilidad diaria de
  0DTE.
  https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4701401

  **Implicación:** el alfa documentado es en buena medida histórico. Cuidado con
  backtests que incluyen 2016-2022.

### C.2 Qué features funcionan realmente (evidencia aplicada)

Del trabajo asociado al repositorio de estrategias 0DTE de Vilkov, con features tomadas
a las **10:00 ET**:

- **Familias de predictores:** varianza implícita (IV), skewness implícita (IS), proxies
  de pendiente; momentos realizados rezagados (retorno, varianza, skewness del día
  previo); moneyness, PNL rezagado (1 día y media de 5 días), griegas (delta, gamma,
  vega); volumen, notional flow en delta/gamma/vega, bid-ask spreads, profundidad
  mostrada; proxies tipo GEX (gamma ponderada por open interest, ratios de presión de
  flujo, medidas de estrechez de liquidez).

- **Hallazgo 1 — la asimetría domina.** El PNL está impulsado más por la **skewness
  realizada** (movimiento direccional) que por la varianza realizada, salvo en
  estructuras puramente de varianza. La varianza realizada sola explica poco: máximo
  **~4.5% de R²** cross-day (put ratio spreads). Añadir skewness realizada mejora
  sustancialmente el ajuste.

- **Hallazgo 2 — clasificar > regresionar.** Modelos logísticos sobre el **signo** del
  PNL dan resultados implementables mejores que regresiones sobre la magnitud del retorno.

- **Hallazgo 3 — el VRP a un día es económicamente diminuto.** VRP realizado mediano de
  10:00 ET a cierre ≈ **0.0011% del subyacente**, difícil de monetizar después de costos.

- **Performance:** put ratio spreads SR 0.93 neto; strangle/straddle SR 0.39 neto;
  canasta top-3 equiponderada SR 0.82 neto. Sharpe brutos típicos 0.56-1.18 antes de
  costos.

  https://github.com/vilkovgr/0dte-strategies/blob/main/docs/paper/paper-annotated.md

### C.3 Costos de transacción: el predictor negativo más fuerte

- **Beckmeyer, Branger y Gayda (2023), "Retail Traders Love 0DTE Options... But Should
  They?"** Los retail son muy activos en 0DTE y en agregado **pierden**.
  https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4404704

- **Bogousslavsky y Muravyev, "An Anatomy of Retail Option Trading".** Las operaciones
  0DTE pierden **4.7%** relativo a otras operaciones de opciones (t = -10), frente a
  +0.19% de las no-0DTE (*). La causa mecánica: precios mucho más bajos ⇒ **spreads
  relativos mucho más grandes**. Además, ~75% del trading retail en opciones SPX es 0DTE,
  pero ~94% del volumen 0DTE total es institucional.
  https://www.lsu.edu/business/files/event-files/2025-finance-mardi-gras/retail_option_trading_v2.pdf

  **Implicación:** el spread relativo al momento de entrada es una feature de primer
  orden, no un detalle de implementación. Un modelo que predice retornos brutos sobre
  0DTE OTM baratos está prediciendo algo que no es negociable.

---

## D. Pinning / distribución terminal

- **Ni, Pearson y Poteshman (2005), *JFE*.** En vencimientos, los precios cierran cerca
  de los strikes con alto open interest. El mecanismo es el rebalanceo de hedges
  delta-neutrales. Efecto documentado: al menos 2% de las acciones con opciones ven su
  retorno alterado en un día de vencimiento típico, con desviación media ≥16.5 bps (*).
  El efecto es pequeño para índices grandes, pero el open interest por strike y el
  "max pain" son features legítimas para modelar la distribución terminal.

---

## Resumen ejecutivo: ranking de predictores

**Nivel 1 — evidencia fuerte, publicada en revistas top:**
1. **VIX1D / varianza implícita de muy corto plazo** → para volatilidad realizada
   (Albers 2025). Corregir el sesgo overnight.
2. **Gamma neta de los dealers (GEX)** → para el régimen de volatilidad y reversión
   (Dim-Eraker-Vilkov 2024; Adams-Fontaine-Ornthanalai 2024). Signo y magnitud.
3. **Retorno de la primera media hora** → para dirección de la última media hora
   (Gao-Han-Li-Zhou 2018 JFE), con la advertencia de degradación en la era 0DTE.
4. **Skewness implícita y skewness realizada** → para PNL de estructuras direccionales.
5. **Spread relativo / costos** → predictor negativo dominante del PNL neto.

**Nivel 2 — evidencia razonable:**
6. Order flow imbalance en opciones (call vs put).
7. Componente de salto / prima de riesgo de salto.
8. Dummies de anuncios macro + ventana de reacción (~26-49 min post-anuncio).
9. Volumen y profundidad del libro.
10. Open interest por strike (pinning).

**Lo que NO funciona bien:**
- Varianza realizada sola para predecir PNL (R² máximo ~4.5%).
- Predecir la **magnitud** del retorno del 0DTE en vez del **signo**.
- Estrategias calibradas sobre 2016-2022 sin re-testear post-2022 (el alfa se disipa).

---

## Recomendaciones para modelar

1. **Empezar por el target B (volatilidad), no por A (dirección).** La relación
   señal-ruido es un orden de magnitud mejor.
2. **Clasificación binaria antes que regresión** cuando el target es PNL.
3. **Modelar el estado, no solo el nivel.** El GEX funciona como variable de régimen y
   en interacciones (p.ej. `momentum_primera_media_hora × 1{GEX < 0}`), no como
   regresor aditivo.
4. **Fijar la hora de corte de las features** (p.ej. 10:00 ET) y no mezclar información
   de horas distintas — es la fuente de look-ahead bias más común en 0DTE.
5. **Incluir costos desde el primer backtest.** Un Sharpe bruto de 1.0 se convierte
   típicamente en 0.3-0.9 neto según la estructura.
6. **Cortar la muestra en 2022-2023.** El régimen cambió con la disponibilidad diaria
   de vencimientos.

---

## Referencias

| Trabajo | Publicación | Aporte |
|---|---|---|
| Gao, Han, Li, Zhou (2018) | *J. Financial Economics* | Momentum intradía, R² 1.6% |
| Baltussen, Da, Lammers, Martens (2021) | *J. Financial Economics* | Momentum intradía global |
| Ni, Pearson, Poteshman (2005) | *J. Financial Economics* | Pinning en vencimientos |
| Bandi, Fusari, Renò (2023) | SSRN 4503344 | Valoración 0DTE, saltos |
| Almeida, Freire, Hizmeri (2024) | SSRN 4701401 | VRP 0DTE, asimetría al alza |
| Dim, Eraker, Vilkov (2024) | SSRN 4692190 | Gamma MM → volatilidad futura |
| Adams, Fontaine, Ornthanalai (2024) | Cboe research | Impacto de gamma OMM, +6.4 pp |
| Beckmeyer, Branger, Gayda (2023) | SSRN 4404704 | Pérdidas retail en 0DTE |
| Bogousslavsky, Muravyev | Working paper | -4.7% en trades 0DTE |
| Albers (2025) | *J. Futures Markets* | VIX1D en HAR-RV |
| Božović | SSRN 5223127 | Prima de salto intradía |
