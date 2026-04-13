# Modelo de Riesgo Crediticio — LendingClub (Tarjetas de Crédito)

**Asignatura:** Modelos de Crédito  
**Institución:** ITESO — Instituto Tecnológico y de Estudios Superiores de Occidente  
**Profesor:** Rodolfo Slay Ramos  
**Equipo:**
- Mónica Valeria Jauregui Rodríguez
- Ricardo Ibarra Talamantes
- Luis Fernando Ramírez Torres

**Fecha:** Abril 2026

---

## Descripción del Proyecto

Este proyecto desarrolla un modelo integral de riesgo crediticio aplicado al portafolio de préstamos para refinanciamiento de tarjetas de crédito de **LendingClub** (2007–2015), compuesto por aproximadamente 249,000 registros.

El objetivo es estimar los tres componentes de la **Pérdida Esperada** bajo el marco regulatorio de Basilea II:

$$EL = PD \times LGD \times EAD$$

| Componente | Descripción | Método |
|---|---|---|
| **PD** (Probability of Default) | Probabilidad de que el cliente incumpla | Regresión Logística + Random Forest |
| **LGD** (Loss Given Default) | Fracción del saldo que se pierde tras el impago | Ridge Regression + Random Forest Regressor |
| **EAD** (Exposure at Default) | Saldo expuesto al momento del incumplimiento | `funded_amnt` como proxy |

---

## Estructura del Repositorio

```
📁 Credit_model_project_2/
│
├── 📄 README.md                              ← Este archivo
├── 📄 Modelo_de_crédito_proyecto_2.pdf        ← Reporte final (PDF)
├── 📓 modelo_riesgo_crediticio.ipynb          ← Notebook principal con todo el pipeline
├── 📊 AAlendingclub_credit_card-limpio.csv    ← Dataset preprocesado (249,343 filas × 40 columnas)
│
├── 📈 Gráficos generados/
│   ├── chart1_rate_vs_default.png             ← Tasa de interés vs tasa de default por grade
│   ├── chart2_stacked_components.png          ← Descomposición de la tasa por componente
│   └── chart3_product_comparison.png          ← Comparación de tasas entre productos crediticios
│
└── 📄 Diagrama_logico.png                     ← Diagrama lógico del modelo (flowchart)
```

---

## Pipeline del Modelo

El notebook `modelo_riesgo_crediticio.ipynb` sigue el siguiente flujo:

### 1. Carga y Exploración de Datos (EDA)
- Carga del dataset preprocesado de LendingClub (préstamos con `purpose = credit_card`).
- Análisis exploratorio: distribución de `loan_status`, tasas de interés por grade, DTI, montos.
- Visualizaciones de la variable objetivo y variables clave.

### 2. Preparación de Datos

**Variable objetivo PD:**
- Se eliminaron préstamos con estatus `Current` (resultado desconocido).
- Se definió `target_pd = 1` para: `Charged Off`, `Late (31-120 days)`, `Late (16-30 days)`, `In Grace Period`, `Default`.
- Se definió `target_pd = 0` para: `Fully Paid`.
- **Resultado:** 95,269 préstamos (77.8% no default, 22.2% default).

**Variable objetivo LGD:**
- Calculada solo sobre préstamos en default: `LGD = 1 - (recoveries / funded_amnt)`.
- Acotada entre 0 y 1.
- **Resultado:** 21,159 préstamos en default, LGD media = 0.94, LGD mediana = 1.0.

**Transformaciones aplicadas:**
- `term`: texto → numérico (36 o 60 meses).
- `emp_length`: texto ordinal → entero (0–10).
- `earliest_cr_line`: fecha → antigüedad crediticia en años.
- Variables categóricas: One-Hot Encoding (`grade`, `sub_grade`, `home_ownership`, `verification_status`, `purpose`).
- Imputación: mediana para numéricas, `-1` para variables de tipo "nunca ocurrió".

**Split train/test:**
- PD: 80/20 estratificado → 76,215 train / 19,054 test.
- LGD: 80/20 sin estratificar → 16,927 train / 4,232 test.
- Escalado con `StandardScaler` para modelos lineales.

### 3. Modelo PD — Clasificación Binaria

Se entrenaron y compararon dos modelos:

| Modelo | AUC-ROC | Gini | KS |
|---|---|---|---|
| Regresión Logística | 0.7199 | 0.4397 | 0.3231 |
| **Random Forest** | **0.7221** | **0.4442** | **0.3257** |

- Validación cruzada estratificada de 5 pliegues: AUC media = 0.7175 ± 0.004.
- Gini de 0.44 dentro del benchmark de la industria (0.40–0.60, Basel Committee, 2006).
- `class_weight='balanced'` utilizado para compensar el desbalance de clases.

**Top 5 variables más importantes (Random Forest):**
1. `int_rate` (tasa de interés)
2. `grade` / `sub_grade` (calificación crediticia)
3. `dti` (relación deuda-ingreso)
4. `cr_hist_years` (antigüedad crediticia)
5. `revol_util` (utilización de crédito revolving)

### 4. Modelo LGD — Regresión

| Modelo | RMSE | MAE | R² |
|---|---|---|---|
| Ridge Regression | 0.0838 | 0.0578 | 0.0144 |
| Random Forest Reg. | 0.0841 | 0.0589 | 0.0076 |

- El R² prácticamente nulo es consecuencia de la concentración del LGD en 1.0 (pérdida total).
- El RMSE bajo (~0.08) refleja que el modelo predice cerca de la media, no que tenga poder predictivo.
- Se utilizó el LGD promedio del portafolio (≈0.94) como proxy conservador para el cálculo de EL.

### 5. Pérdida Esperada (EL)

| Métrica | Valor |
|---|---|
| EL media por préstamo | $6,627 USD |
| EL total del portafolio (test) | $126.3 millones USD |
| EL media como % del EAD | 43.4% |

**Nota:** La estimación es conservadora por dos razones: (1) el LGD proxy de 0.94 sobreestima la pérdida para préstamos de alta calidad, y (2) el uso de `class_weight='balanced'` eleva la PD media predicha por encima de la tasa observada.

---

## Análisis de Formación de Tasas de Interés

El proyecto incluye un análisis detallado de los componentes que conforman la tasa de interés de los préstamos para refinanciamiento de tarjetas:

| Componente | Rango | Fuente |
|---|---|---|
| Tasa Base (Fed Funds + intermediación) | ~4.5% | Federal Reserve (FRED) |
| Prima de Inflación | ~1.8% | Bureau of Labor Statistics (CPI) |
| Prima de Riesgo Crediticio | 0.5% – 20% | Dataset LendingClub (por grade) |
| Prima de Liquidez | 0.8% – 2.0% | Comparación con tasas hipotecarias |
| Costos Administrativos | 0.7% – 3.5% | LendingClub rate disclosures |
| Margen de Beneficio | Residual | Wikipedia: LendingClub |

Se generaron tres gráficos para el reporte:
1. **Tasa de interés vs tasa de default por grade** — demuestra la relación lineal entre riesgo y precio.
2. **Descomposición apilada de la tasa** — muestra cómo cada componente suma al total por grade.
3. **Comparación entre productos** — hipotecas (3.5–4.5%) vs préstamos personales LC (6.9–29.2%) vs tarjetas revolving (15–30%).

---

## Tecnologías Utilizadas

- **Python 3.10+**
- **pandas** — manipulación y limpieza de datos
- **numpy** — operaciones numéricas
- **scikit-learn** — modelos de ML, métricas, validación cruzada, escalado
- **matplotlib / seaborn** — visualizaciones
- **Jupyter Notebook** — desarrollo interactivo

---

## Cómo Ejecutar

1. Clonar el repositorio:
   ```bash
   git clone https://github.com/RIT5641/Credit_model_project_2.git
   cd Credit_model_project_2
   ```

2. Instalar dependencias:
   ```bash
   pip install pandas numpy scikit-learn matplotlib seaborn
   ```

3. Colocar el archivo `AAlendingclub_credit_card-limpio.csv` en el mismo directorio que el notebook.

4. Abrir y ejecutar el notebook:
   ```bash
   jupyter notebook modelo_riesgo_crediticio.ipynb
   ```

5. El notebook genera todas las métricas, gráficos y el resumen ejecutivo de forma automática.

---

## Supuestos y Limitaciones

1. Los préstamos `Current` fueron excluidos por tener resultado desconocido al momento del corte.
2. El LGD se calculó como `1 - (recoveries / funded_amnt)`, aproximación que ignora el valor temporal del dinero y costos de recuperación.
3. El EAD se aproximó mediante `funded_amnt` (monto fondeado), sin modelar amortización parcial previa al default.
4. El uso de `class_weight='balanced'` infla las probabilidades absolutas de PD (~46% vs ~22% real), aunque las métricas discriminantes (AUC, Gini, KS) no se ven afectadas.
5. Se asume independencia entre PD, LGD y EAD (supuesto estándar de Basilea II).
6. Los datos de LendingClub (plataforma P2P) pueden no ser completamente representativos del mercado bancario tradicional de tarjetas de crédito.

---

## Mejoras Futuras

- Implementar **XGBoost / LightGBM** para mayor poder predictivo en PD.
- Modelar LGD con **Beta Regression** (distribución natural para variables acotadas entre 0 y 1).
- Aplicar **calibración de probabilidades** (Platt scaling o isotonic regression) para corregir la inflación de PD.
- Incorporar **variables macroeconómicas** (tasa de desempleo, CPI, spread de crédito) para un modelo sensible al ciclo económico.
- Implementar **SHAP values** para interpretabilidad avanzada.
- Realizar **backtesting** sobre ventanas temporales deslizantes.
- Desarrollar **stress testing** del portafolio bajo escenarios adversos.

---

## Referencias

Las referencias completas se encuentran en el reporte escrito (PDF). Las principales fuentes utilizadas son:

- Basel Committee on Banking Supervision (2006). *Basel II: International Convergence of Capital Measurement and Capital Standards*.
- Di Maggio, M. & Yao, V. (2018). *Fintech Borrowers: Lax-Screening or Cream-Skimming?* Federal Reserve Bank of Philadelphia, WP 18-15.
- Federal Reserve Board (2022). *Credit Card Profitability*. FEDS Notes.
- Siddiqi, N. (2006). *Credit Risk Scorecards*. John Wiley & Sons.
- Thomas, L. C., Edelman, D. B. & Crook, J. N. (2002). *Credit Scoring and Its Applications*. SIAM.
- U.S. Bureau of Labor Statistics. *Consumer Price Index (CPI), 2015–2018*.

---

## Licencia

Proyecto académico. Dataset original proporcionado por LendingClub a través de Kaggle.
