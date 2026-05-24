# How Was Built: `alquileres_espana_valencia_estudiantes_2014_2025.csv`

> Dataset unificado de alquileres en España con foco en Comunitat Valenciana y segmento estudiantil (2014–2025).
> **34,003 filas × 83 columnas** (48 originales + 35 calculadas)

---

## Índice

1. [Arquitectura general del pipeline](#1-arquitectura-general-del-pipeline)
2. [Paso 0 — Exploración y análisis del dataset base](#2-paso-0--exploración-y-análisis-del-dataset-base)
3. [Paso 1 — Carga del CSV existente](#3-paso-1--carga-del-csv-existente)
4. [Paso 2 — Generación de filas sintéticas 2025](#4-paso-2--generación-de-filas-sintéticas-2025)
5. [Paso 3 — Bloque 1: Evolución y tendencia de precio](#5-paso-3--bloque-1-evolución-y-tendencia-de-precio)
6. [Paso 4 — Bloque 2: Accesibilidad y poder adquisitivo](#6-paso-4--bloque-2-accesibilidad-y-poder-adquisitivo)
7. [Paso 5 — Bloque 3: Contexto económico y mercado](#7-paso-5--bloque-3-contexto-económico-y-mercado)
8. [Paso 6 — Bloque 4: Perfil del inquilino y demanda](#8-paso-6--bloque-4-perfil-del-inquilino-y-demanda)
9. [Paso 7 — Bloque 5: Específico estudiantes Valencia](#9-paso-7--bloque-5-específico-estudiantes-valencia)
10. [Paso 8 — Bloque 6: KPIs precalculados para Looker](#10-paso-8--bloque-6-kpis-precalculados-para-looker)
11. [Paso 9 — Formateo y exportación final](#11-paso-9--formateo-y-exportación-final)
12. [Verificación de calidad](#12-verificación-de-calidad)

---

## 1. Arquitectura general del pipeline

```
┌─────────────────────────────────────────────────────────────────────┐
│  alquileres_espana_valencia_estudiantes_2014_2024.csv               │
│  (31,001 rows × 48 cols)                                           │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  1. DATA EXPLORATION & ANALYSIS                                     │
│  - Distribución por año, CCAA, provincia                            │
│  - Estadísticas de precio por segmento                              │
│  - Patrones de estacionalidad                                       │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  2. 2025 SYNTHETIC DATA GENERATION                                  │
│  - Muestreo proporcional por CCAA desde 2024                        │
│  - Incremento de precios: +3-5% general, +8% estudiante CV         │
│  - Trimestres Q1-Q3 (año en curso)                                  │
│  - calidad_dato = SIMULADO                                          │
│  → +3,002 filas                                                     │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  3. FEATURE ENGINEERING (35 new columns)                            │
├─────────────────┬─────────────────┬─────────────────┬───────────────┤
│  BLOQUE 1       │  BLOQUE 2       │  BLOQUE 3       │  BLOQUE 4     │
│  Tendencia      │  Accesibilidad  │  Contexto       │  Perfil       │
│  precio (8)     │  SMI/IPC (7)    │  económico (6)  │  demanda (5)  │
├─────────────────┼─────────────────┼─────────────────┼───────────────┤
│  BLOQUE 5       │  BLOQUE 6       │                 │               │
│  Estudiante     │  KPIs Looker    │                 │               │
│  Valencia (4)   │  pre-calc (5)   │                 │               │
└─────────────────┴─────────────────┴─────────────────┴───────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  4. FORMAT & EXPORT                                                 │
│  - UTF-8 BOM encoding                                              │
│  - Floats 2 decimales, booleans lowercase                          │
│  - NULLs como cadena vacía                                          │
│  - Orden columnas: originales + nuevas                              │
│  ─────────────────────────────────────────────────────────────────  │
│  → alquileres_espana_valencia_estudiantes_2014_2025.csv             │
│    (34,003 rows × 83 cols)                                          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Paso 0 — Exploración y análisis del dataset base

Antes de generar ningún cambio, se analizó la estructura y distribución del dataset original.

```python
import pandas as pd

df = pd.read_csv(
    r'alquileres_espana_valencia_estudiantes_2014_2024.csv',
    low_memory=False
)

# Estructura general
print('Shape:', df.shape)
print('Columns:', len(df.columns))
print('Years:', sorted(df['año'].unique()))

# Distribución por año
print(df['año'].value_counts().sort_index())
```

**Output:**
```
Shape: (31001, 48)
Years: [2014, 2015, 2016, 2017, 2018, 2019, 2020, 2021, 2022, 2023, 2024]
Rows per year:
2014    2025
2015    2117
2016    2136
2017    2163
2018    2227
2019    2540
2020    2562
2021    2779
2022    3539
2023    4123
2024    4790
```

```python
# Distribución por CCAA
print(df['ccaa'].value_counts())

# Distribución por tipo de alquiler
print(df['tipo_alquiler'].value_counts())

# Distribución por tipo de vivienda
print(df['tipo_vivienda'].value_counts())
```

**Output clave:**
- **CCAA dominante**: Comunitat Valenciana (13,369 filas, ~43%)
- **tipo_alquiler**: 25,001 residencial, 3,000 turistico_presion, 3,000 estudiantil
- **tipo_vivienda**: 18,529 piso, 4,150 casa, 3,380 estudio, 3,000 habitacion, 843 duplex

```python
# Estadísticas de precio
print('precio_mes_eur stats:')
print(df['precio_mes_eur'].describe())

# Datos 2024 específicamente
df24 = df[df['año'] == 2024]
print('2024 precio_mes_eur stats:')
print(df24['precio_mes_eur'].describe())

# Datos estudiantiles CV
cv_stud = df[(df['ccaa'] == 'Comunitat Valenciana') & (df['tipo_alquiler'] == 'estudiantil')]
print('CV student rows:', len(cv_stud))
print('CV student precio stats:')
print(cv_stud['precio_mes_eur'].describe())
```

**Output clave:**
- Precio medio global: 862€
- Precio medio 2024: 1,133€ (tendencia alcista)
- Precio medio estudiante CV: 474€ (más asequible)

```python
# Tipos de dato y nulos
print('Dtypes:')
print(df.dtypes)
print('Null counts:')
print(df.isnull().sum())

# Ver datos específicos de CV
cv = df[df['ccaa'] == 'Comunitat Valenciana']
print('CV rows per year:')
print(cv['año'].value_counts().sort_index())
print('CV provincia distribution:')
print(cv['provincia'].value_counts())
print('CV distrito distribution:')
print(cv['distrito'].value_counts().head(20))
```

---

## 3. Paso 1 — Carga del CSV existente

```python
import pandas as pd
import numpy as np
import warnings
warnings.filterwarnings('ignore')

df = pd.read_csv(
    r'C:\Users\Pablo\Downloads\alquileres_espana_valencia_estudiantes_2014_2024.csv',
    low_memory=False,
    encoding='utf-8-sig'
)
print(f"Loaded {len(df)} rows x {len(df.columns)} cols")
```

**Resultado:** 31,001 filas × 48 columnas cargadas correctamente.

---

## 4. Paso 2 — Generación de filas sintéticas 2025

Se generaron **3,002 filas** para 2025 mediante muestreo proporcional desde 2024 con ajustes de precio realistas.

### 4.1 Configuración inicial

```python
np.random.seed(42)  # Reproducibilidad

df24 = df[df['año'] == 2024].copy()
n_2025 = 3000

# Distribución proporcional por CCAA (mantener la misma que 2024)
ccaa_dist = df24['ccaa'].value_counts(normalize=True)

id_counter = df['id_registro'].max() + 1
```

### 4.2 Parámetros económicos por CCAA

```python
# Tasas de paro estimadas para 2025 por CCAA (fuente: EPA/INE, tendencia descendente)
tasa_paro_ccaa_2025 = {
    'Andalucia': 22.5, 'Canarias': 21.0, 'Extremadura': 22.0,
    'Castilla-La Mancha': 19.0, 'Murcia': 17.5, 'Comunitat Valenciana': 17.0,
    'Asturias': 16.5, 'Cantabria': 15.0, 'Baleares': 14.5,
    'Galicia': 14.0, 'Aragon': 13.0, 'Cataluña': 12.5,
    'La Rioja': 12.0, 'Navarra': 11.0, 'Madrid': 12.5,
    'Castilla y Leon': 13.5, 'Pais Vasco': 10.0
}

# Factores regionales de precio de compra (proxies de mercado)
regional_factor = {
    'Madrid': 1.8, 'Cataluña': 1.7, 'Comunitat Valenciana': 1.2,
    'Baleares': 1.9, 'Andalucia': 0.9, 'Aragon': 0.9,
    'Asturias': 0.9, 'Canarias': 0.9, 'Cantabria': 0.9,
    'Castilla-La Mancha': 0.9, 'Castilla y Leon': 0.9,
    'Extremadura': 0.9, 'Galicia': 0.9, 'La Rioja': 0.9,
    'Murcia': 0.9, 'Navarra': 0.9, 'Pais Vasco': 0.9
}
```

### 4.3 Parámetros específicos de Valencia (estudiantil)

```python
# Calidad de zona universitaria para distritos de Valencia
zona_calidad = {
    'Benimaclet': 'PRIME', 'Algiros': 'PRIME', 'Camins al Grau': 'PRIME',
    'Rascanya': 'BUENA', 'Quatre Carreres': 'BUENA',
    'Patraix': 'ECONOMICA', "L'Olivereta": 'ECONOMICA', 'Jesus': 'ECONOMICA',
}

# Competencia de pisos turísticos estimada por distrito (% de oferta desplazada)
turismo_comp = {
    'Ciutat Vella': 55, 'Poblats Maritims': 40, 'Eixample': 25,
    'Extramurs': 15, 'El Pla del Real': 12, 'Campanar': 8,
    'Benimaclet': 10, 'Algiros': 10, 'Camins al Grau': 8,
    'Rascanya': 5, 'Quatre Carreres': 5, 'Patraix': 3,
    "L'Olivereta": 3, 'Jesus': 3, 'La Saidia': 12,
    'Benicalap': 3, 'Pobles del Nord': 2, "Pobles de l'Oest": 2,
    'Pobles del Sud': 2
}
```

### 4.4 Bucle de generación

```python
for ccaa, pct in ccaa_dist.items():
    n_ccaa = max(1, int(round(n_2025 * pct)))
    ccaa_rows_24 = df24[df24['ccaa'] == ccaa]

    if len(ccaa_rows_24) == 0:
        continue

    # Muestreo con reemplazo desde 2024
    sampled = ccaa_rows_24.sample(n=n_ccaa, replace=True, random_state=42 + hash(ccaa) % 10000)

    is_cv = (ccaa == 'Comunitat Valenciana')

    for _, row_24 in sampled.iterrows():
        r = id_counter
        id_counter += 1

        # Asignar trimestre (solo Q1-Q3 para 2025, año en curso)
        trimestre = np.random.choice(['Q1', 'Q2', 'Q3'])
        if trimestre == 'Q1':
            mes = np.random.randint(1, 4)
        elif trimestre == 'Q2':
            mes = np.random.randint(4, 7)
        else:
            mes = np.random.randint(7, 11)
        fecha = f'2025-{mes:02d}-01'

        # Factor de incremento de precio según segmento
        tipo_alq = str(row_24['tipo_alquiler']).strip() if pd.notna(row_24['tipo_alquiler']) else 'residencial'
        if is_cv and tipo_alq == 'estudiantil':
            price_factor = np.random.uniform(1.06, 1.10)    # +6-10% estudiantes CV
        elif is_cv:
            price_factor = np.random.uniform(1.04, 1.07)    # +4-7% CV general
        else:
            price_factor = np.random.uniform(1.03, 1.05)    # +3-5% resto nacional

        precio = round(row_24['precio_mes_eur'] * price_factor, 2)

        # Recalcular precio/m2
        sup = row_24['superficie_m2'] if pd.notna(row_24['superficie_m2']) and row_24['superficie_m2'] > 0 else 60.0
        precio_m2 = round(precio / sup, 2) if sup > 0 else 0.0

        new_row = row_24.to_dict()
        new_row.update({
            'id_registro': r,
            'año': 2025,
            'trimestre': trimestre,
            'mes': mes,
            'fecha_referencia': fecha,
            'precio_mes_eur': precio,
            'precio_m2_mes': precio_m2,
            'calidad_dato': 'SIMULADO',
            'notas': 'Proyección 2025 basada en tendencia 2023-2024',
        })
        rows_2025.append(new_row)

df_2025 = pd.DataFrame(rows_2025)
print(f"Generated {len(df_2025)} rows for 2025")

# Combinar datasets
df_all = pd.concat([df, df_2025], ignore_index=True)
df_all['año'] = df_all['año'].astype(int)
print(f"Combined: {len(df_all)} rows x {len(df_all.columns)} cols")
```

**Lógica de incremento de precios 2025:**

| Segmento | Incremento | Justificación |
|----------|-----------|---------------|
| Nacional general | +3-5% | Tendencia de mercado estable |
| CV general | +4-7% | Mayor presión turística y demográfica |
| CV estudiantil | +6-10% | Demanda post-pandemia, auge de estudiantes internacionales |

---

## 5. Paso 3 — Bloque 1: Evolución y tendencia de precio

Columnas críticas para gráficos de series temporales en Looker Studio.

### 5.1 `precio_mes_eur_media_movil_3y` y `precio_mes_eur_media_movil_5y`

Medias móviles para suavizar la volatilidad y mostrar tendencias.

```python
df_all = df_all.sort_values(['ccaa', 'tipo_vivienda', 'año']).reset_index(drop=True)

df_all['precio_mes_eur_media_movil_3y'] = (
    df_all.groupby(['ccaa', 'tipo_vivienda'])['precio_mes_eur']
    .transform(lambda x: x.rolling(window=3, min_periods=1).mean().round(2))
)

df_all['precio_mes_eur_media_movil_5y'] = (
    df_all.groupby(['ccaa', 'tipo_vivienda'])['precio_mes_eur']
    .transform(lambda x: x.rolling(window=5, min_periods=1).mean().round(2))
)
```

### 5.2 `precio_index_2014_100`

Índice de precio reescalado a 2014=100, esencial para comparar evolución entre regiones con diferentes escalas de precio.

```python
precio_2014 = df_all[df_all['año'] == 2014].groupby(['ccaa', 'tipo_vivienda'])['precio_mes_eur'].mean().reset_index()
precio_2014.columns = ['ccaa', 'tipo_vivienda', 'precio_mes_eur_2014']

df_all = df_all.merge(precio_2014, on=['ccaa', 'tipo_vivienda'], how='left')
df_all['precio_index_2014_100'] = np.where(
    df_all['precio_mes_eur_2014'] > 0,
    (df_all['precio_mes_eur'] / df_all['precio_mes_eur_2014']) * 100,
    np.nan
).round(2)
df_all.drop(columns=['precio_mes_eur_2014'], inplace=True)
```

### 5.3 `variacion_precio_mensual_pct`

Cambio porcentual mes a mes. NULL si el campo `mes` es NULL (datos anuales sin desglose mensual).

```python
df_all = df_all.sort_values(['ccaa', 'tipo_vivienda', 'año']).reset_index(drop=True)

df_all['variacion_precio_mensual_pct'] = (
    df_all.groupby(['ccaa', 'tipo_vivienda'])['precio_mes_eur']
    .transform(lambda x: x.pct_change() * 100)
).round(2)

# Solo válido cuando hay dato de mes
df_all['variacion_precio_mensual_pct'] = df_all['variacion_precio_mensual_pct'].where(
    df_all['mes'].notna() & (df_all['mes'] > 0), np.nan
)
```

### 5.4 `variacion_precio_acumulada_desde_2014_pct`

Variación total acumulada desde el año base 2014. Mide el incremento/decremento total en el período.

```python
precio_2014_agg = df_all[df_all['año'] == 2014].groupby(['ccaa', 'tipo_vivienda'])['precio_mes_eur'].mean().reset_index()
precio_2014_agg.columns = ['ccaa', 'tipo_vivienda', 'base_2014']

df_all = df_all.merge(precio_2014_agg, on=['ccaa', 'tipo_vivienda'], how='left')
df_all['variacion_precio_acumulada_desde_2014_pct'] = np.where(
    df_all['base_2014'] > 0,
    ((df_all['precio_mes_eur'] - df_all['base_2014']) / df_all['base_2014']) * 100,
    np.nan
).round(2)
df_all.drop(columns=['base_2014'], inplace=True)
```

### 5.5 `precio_percentil_nacional` y `precio_percentil_ccaa`

Percentiles (1-100) para responder: "Este alquiler es más caro que el X% de España".

```python
df_all['precio_percentil_nacional'] = (
    df_all.groupby(['año', 'tipo_vivienda'])['precio_mes_eur']
    .rank(pct=True) * 100
).round(0).fillna(50).astype(int)
df_all['precio_percentil_nacional'] = df_all['precio_percentil_nacional'].clip(1, 100)

df_all['precio_percentil_ccaa'] = (
    df_all.groupby(['año', 'ccaa', 'tipo_vivienda'])['precio_mes_eur']
    .rank(pct=True) * 100
).round(0).fillna(50).astype(int)
df_all['precio_percentil_ccaa'] = df_all['precio_percentil_ccaa'].clip(1, 100)
```

### 5.6 `tendencia_precio`

Clasificación de la tendencia a 3 años en categorías discretas.

```python
def classify_trend(group):
    last3 = group.tail(3)
    if len(last3) < 2:
        return pd.Series([None]*len(group), index=group.index)
    first_val = last3.iloc[0]['precio_mes_eur']
    last_val = last3.iloc[-1]['precio_mes_eur']
    if first_val == 0:
        return pd.Series([None]*len(group), index=group.index)

    # Tasa de crecimiento anual compuesta (CAGR) a 3 años
    annual_change = ((last_val / first_val) ** (1/3) - 1) * 100

    if annual_change > 10:
        label = 'SUBIDA_FUERTE'
    elif annual_change > 3:
        label = 'SUBIDA_MODERADA'
    elif annual_change > -3:
        label = 'ESTABLE'
    elif annual_change > -10:
        label = 'BAJADA'
    else:
        label = 'BAJADA_FUERTE'
    return pd.Series([label]*len(group), index=group.index)

df_all['tendencia_precio'] = None
df_all = df_all.sort_values(['ccaa', 'tipo_vivienda', 'año']).reset_index(drop=True)
for (ccaa, tv), group in df_all.groupby(['ccaa', 'tipo_vivienda']):
    labels = classify_trend(group.reset_index(drop=True))
    df_all.loc[group.index, 'tendencia_precio'] = labels.values
```

**Umbrales de clasificación:**

| Categoría | Criterio (CAGR 3 años) |
|-----------|----------------------|
| `SUBIDA_FUERTE` | > 10% anual |
| `SUBIDA_MODERADA` | 3% – 10% anual |
| `ESTABLE` | −3% – 3% anual |
| `BAJADA` | −3% – −10% anual |
| `BAJADA_FUERTE` | < −10% anual |

---

## 6. Paso 4 — Bloque 2: Accesibilidad y poder adquisitivo

Columnas críticas para indicadores ODS 11 y análisis de asequibilidad.

### 6.1 SMI (Salario Mínimo Interprofesional)

```python
smi_map = {
    2014: 645, 2015: 648.6, 2016: 655.2, 2017: 707.6,
    2018: 735.9, 2019: 900, 2020: 950, 2021: 965,
    2022: 1000, 2023: 1080, 2024: 1134, 2025: 1184
}
df_all['salario_minimo_interprofesional'] = df_all['año'].map(smi_map)
```

### 6.2 Métricas derivadas del SMI

```python
df_all['meses_smi_para_alquiler'] = (
    df_all['precio_mes_eur'] / df_all['salario_minimo_interprofesional']
).round(2)

df_all['esfuerzo_smi_pct'] = (
    df_all['precio_mes_eur'] / df_all['salario_minimo_interprofesional'] * 100
).round(2)
```

### 6.3 IPC e inflación

```python
ipc_map = {
    2014: -0.2, 2015: -0.5, 2016: -0.2, 2017: 2.0,
    2018: 1.7, 2019: 0.8, 2020: -0.3, 2021: 3.1,
    2022: 8.4, 2023: 3.5, 2024: 2.8, 2025: 2.3
}
df_all['ipc_anual_espana'] = df_all['año'].map(ipc_map)
```

### 6.4 Precio real deflactado

Ajuste a euros constantes de 2014 para eliminar el efecto inflación.

```python
# Factor de inflación acumulada desde 2014
years_sorted = sorted(ipc_map.keys())
cumulative_factor = {}
cum = 1.0
for y in years_sorted:
    cum *= (1 + ipc_map[y] / 100)
    cumulative_factor[y] = cum

df_all['precio_real_deflactado'] = df_all.apply(
    lambda r: round(r['precio_mes_eur'] * (1.0 / cumulative_factor.get(r['año'], 1.0)), 2),
    axis=1
)
```

### 6.5 Brecha compra-alquiler

Ratio que indica si es más barato comprar o alquilar.

```python
precio_compra_base = {
    2014: 1400, 2015: 1430, 2016: 1460, 2017: 1520,
    2018: 1610, 2019: 1680, 2020: 1650, 2021: 1720,
    2022: 1850, 2023: 1950, 2024: 2100, 2025: 2280
}

# Precio de compra estimado por m² (base nacional × factor regional)
df_all['precio_compra_m2_estimado'] = df_all.apply(
    lambda r: round(precio_compra_base.get(r['año'], 2100) * regional_factor.get(r['ccaa'], 0.9), 2),
    axis=1
)

# Ratio price-to-rent
df_all['brecha_precio_compra_alquiler'] = df_all.apply(
    lambda r: round(
        (r['precio_compra_m2_estimado'] * r['superficie_m2']) / (r['precio_mes_eur'] * 12)
        if r['precio_mes_eur'] > 0 and r['superficie_m2'] > 0 else np.nan,
        2
    ),
    axis=1
)
```

**Interpretación del ratio:**
- **> 20**: Alquilar es más barato que comprar
- **15 – 20**: Zona gris (depende de contexto)
- **< 15**: Comprar es más barato que alquilar

---

## 7. Paso 5 — Bloque 3: Contexto económico y mercado

Macro-indicadores para superposición en gráficos de Looker Studio.

### 7.1 Tasa de paro por CCAA

Valores realistas aproximados de la EPA/INE por CCAA y año, con tendencia descendente 2014→2025.

```python
tasa_paro_base = {
    'Andalucia': (32, 35, 34, 32, 30, 28, 27, 26, 25, 24, 23, 22.5),
    'Aragon': (18, 17, 16, 15, 14, 13, 13, 12, 12, 11, 10.5, 10),
    'Asturias': (22, 21, 20, 19, 18, 17, 17, 16, 16, 15, 14.5, 14),
    'Baleares': (24, 23, 22, 21, 20, 19, 18, 17, 16, 15, 14.5, 14),
    'Canarias': (31, 30, 29, 28, 27, 26, 25, 24, 23, 22, 21, 20),
    'Cantabria': (18, 17, 16, 15, 15, 14, 14, 13, 13, 12, 11.5, 11),
    'Castilla-La Mancha': (28, 27, 26, 25, 24, 23, 22, 21, 20, 19, 18.5, 18),
    'Castilla y Leon': (20, 19, 18, 17, 16, 15, 15, 14, 14, 13, 12.5, 12),
    'Cataluña': (18, 17, 16, 15, 14, 13, 13, 12, 12, 11, 10.5, 10),
    'Comunitat Valenciana': (24, 23, 22, 21, 20, 19, 18, 17, 17, 16, 15.5, 15),
    'Extremadura': (30, 29, 28, 27, 26, 25, 24, 23, 22, 21, 20.5, 20),
    'Galicia': (20, 19, 18, 17, 16, 15, 15, 14, 14, 13, 12.5, 12),
    'La Rioja': (16, 15, 14, 13, 13, 12, 12, 11, 11, 10, 9.5, 9),
    'Madrid': (16, 15, 14, 13, 12, 11, 11, 10, 10, 9, 8.5, 8),
    'Murcia': (26, 25, 24, 23, 22, 21, 20, 19, 18, 17, 16.5, 16),
    'Navarra': (14, 13, 12, 11, 11, 10, 10, 9, 9, 8, 7.5, 7),
    'Pais Vasco': (12, 11, 10, 10, 9, 8, 8, 7, 7, 6, 5.5, 5),
}
year_idx_map = {2014: 0, 2015: 1, 2016: 2, 2017: 3, 2018: 4, 2019: 5,
                2020: 6, 2021: 7, 2022: 8, 2023: 9, 2024: 10, 2025: 11}

def get_tasa_paro(row):
    ccaa = row['ccaa']
    year = row['año']
    if ccaa in tasa_paro_base:
        idx = year_idx_map.get(year, 0)
        vals = tasa_paro_base[ccaa]
        if idx < len(vals):
            return vals[idx]
    return 20.0

df_all['tasa_paro_ccaa'] = df_all.apply(get_tasa_paro, axis=1)
```

### 7.2 Euribor 12 meses

```python
euribor_map = {
    2014: 0.48, 2015: 0.17, 2016: -0.08, 2017: -0.18,
    2018: -0.16, 2019: -0.25, 2020: -0.27, 2021: -0.50,
    2022: 1.86, 2023: 4.02, 2024: 3.56, 2025: 2.45
}
df_all['euribor_12m'] = df_all['año'].map(euribor_map)
```

### 7.3 Tipo de contrato predominante

Según la evolución legislativa española.

```python
def get_contrato(row):
    year = row['año']
    zt = row['zona_tensionada_oficial']
    if year >= 2023 and zt:
        return 'CONTENCION'       # Ley de Vivienda 2023 en zonas tensionadas
    elif year >= 2019:
        return 'MEDIA_DURACION'   # Contratos 3-5 años
    else:
        return 'LARGA_DURACION'   # Contratos 5 años

df_all['tipo_contrato_predominante'] = df_all.apply(get_contrato, axis=1)
```

### 7.4 Ley de Vivienda 2023 — Cap机制 de precios

```python
# ¿Aplica la ley?
df_all['ley_vivienda_aplicable'] = (df_all['año'] >= 2023) & (df_all['zona_tensionada_oficial'] == True)

# Tope legal de precio: precio_m2_ref_ministerio × 1.1 × superficie
df_all['cap_precio_ley_vivienda'] = np.where(
    df_all['ley_vivienda_aplicable'],
    round(df_all['precio_m2_ref_ministerio'] * 1.1 * df_all['superficie_m2'].fillna(60), 2),
    np.nan
)

# Exceso sobre tope legal (positivo = no conforme)
df_all['precio_sobre_cap_legal'] = np.where(
    df_all['ley_vivienda_aplicable'],
    round(df_all['precio_mes_eur'] - df_all['cap_precio_ley_vivienda'], 2),
    np.nan
)
```

---

## 8. Paso 6 — Bloque 4: Perfil del inquilino y demanda

Para gráficos de segmentación y análisis demográfico en Looker Studio.

### 8.1 `perfil_inquilino_estimado`

```python
def get_perfil(row):
    tipo_alq = str(row['tipo_alquiler']).strip() if pd.notna(row['tipo_alquiler']) else ''
    if tipo_alq == 'estudiantil':
        return 'ESTUDIANTE'

    precio = row['precio_mes_eur']
    habs = row['num_habitaciones'] if pd.notna(row['num_habitaciones']) else 1
    smi = row['salario_minimo_interprofesional']

    if precio < smi * 0.3 and habs <= 1:
        return 'JOVEN_PROFESIONAL'
    elif habs >= 3:
        return 'FAMILIA'
    elif precio > 1500:
        return 'JOVEN_PROFESIONAL'
    else:
        return 'FAMILIA'

df_all['perfil_inquilino_estimado'] = df_all.apply(get_perfil, axis=1)
```

**Reglas de asignación:**

| Condición | Perfil |
|-----------|--------|
| tipo_alquiler = estudiantil | ESTUDIANTE |
| precio < SMI×0.3 AND num_hab ≤ 1 | JOVEN_PROFESIONAL |
| num_hab ≥ 3 | FAMILIA |
| precio > 1,500€ | JOVEN_PROFESIONAL |
| resto | FAMILIA |

### 8.2 `rango_precio_categoria`

```python
def get_rango_precio(precio):
    if pd.isna(precio):
        return np.nan
    if precio < 400:
        return 'MUY_ECONOMICO'
    elif precio < 700:
        return 'ECONOMICO'
    elif precio < 1000:
        return 'MEDIO'
    elif precio < 1500:
        return 'ALTO'
    else:
        return 'MUY_ALTO'

df_all['rango_precio_categoria'] = df_all['precio_mes_eur'].apply(get_rango_precio)
```

### 8.3 `demanda_estimada_categoria`

```python
def get_demanda(row):
    tension = row['indice_tension_mercado']
    if pd.isna(tension):
        return np.nan
    if tension > 3:
        return 'MUY_ALTA'
    elif tension >= 2:
        return 'ALTA'
    elif tension >= 1:
        return 'MEDIA'
    else:
        return 'BAJA'

df_all['demanda_estimada_categoria'] = df_all.apply(get_demanda, axis=1)
```

### 8.4 `tiempo_hasta_alquiler_categoria`

```python
def get_tiempo(row):
    dias = row['dias_mercado_media']
    if pd.isna(dias):
        return np.nan
    if dias < 7:
        return 'MUY_RAPIDO'
    elif dias <= 21:
        return 'RAPIDO'
    elif dias <= 45:
        return 'NORMAL'
    else:
        return 'LENTO'

df_all['tiempo_hasta_alquiler_categoria'] = df_all.apply(get_tiempo, axis=1)
```

### 8.5 `oferta_relativa_zona`

Compara la oferta de anuncios activos de cada zona con la media de su CCAA.

```python
df_all['oferta_relativa_zona'] = np.nan
for ccaa, group in df_all.groupby('ccaa'):
    mean_anuncios = group['num_anuncios_activos'].mean()
    if mean_anuncios == 0:
        continue
    deviation = (group['num_anuncios_activos'] - mean_anuncios) / mean_anuncios * 100
    labels = pd.cut(
        deviation,
        bins=[-float('inf'), -30, -10, 10, float('inf')],
        labels=['ESCASEZ_CRITICA', 'ESCASEZ', 'EQUILIBRIO', 'EXCESO']
    )
    df_all.loc[group.index, 'oferta_relativa_zona'] = labels
```

---

## 9. Paso 7 — Bloque 5: Específico estudiantes Valencia

Enriquecimiento solo para filas donde `tipo_alquiler = 'estudiantil'`.

### 9.1 `curso_academico`

```python
def get_curso(year):
    return f"{year-1}-{str(year)[-2:]}"

df_all['curso_academico'] = df_all['año'].apply(get_curso)
```

**Mapeo:** año 2024 → "2023-24", año 2025 → "2024-25", etc.

### 9.2 `precio_vs_media_estudiantil_pct`

Desviación porcentual respecto a la media de alquiler estudiantil en Valencia ese año.

```python
df_all['precio_vs_media_estudiantil_pct'] = np.nan
estudiantil_mask = df_all['tipo_alquiler'] == 'estudiantil'

for year in df_all['año'].unique():
    year_mask = estudiantil_mask & (df_all['año'] == year)
    avg_price = df_all.loc[year_mask, 'precio_mes_eur'].mean()
    if avg_price > 0:
        df_all.loc[year_mask, 'precio_vs_media_estudiantil_pct'] = (
            (df_all.loc[year_mask, 'precio_mes_eur'] - avg_price) / avg_price * 100
        ).round(2)
```

### 9.3 `zona_universitaria_calidad`

Califica los distritos de Valencia según su atractivo para estudiantes.

```python
def get_zona_calidad(row):
    distrito = str(row['distrito']).strip() if pd.notna(row['distrito']) else ''
    tipo_alq = str(row['tipo_alquiler']).strip() if pd.notna(row['tipo_alquiler']) else ''
    if tipo_alq != 'estudiantil':
        return np.nan
    if distrito in zona_calidad:
        return zona_calidad[distrito]
    return 'PERIFERICA'

df_all['zona_universitaria_calidad'] = df_all.apply(get_zona_calidad, axis=1)
```

**Clasificación de zonas:**

| Categoría | Distritos |
|-----------|-----------|
| PRIME | Benimaclet, Algirós, Camins al Grau (cerca UPV/UV) |
| BUENA | Rascanya, Quatre Carreres |
| ECONOMICA | Patraix, L'Olivereta, Jesús |
| PERIFERICA | Resto de distritos |

### 9.4 `competencia_pisos_turisticos`

Estima el % de oferta residencial desplazada por pisos turísticos, con escalado temporal.

```python
def get_competencia(row):
    distrito = str(row['distrito']).strip() if pd.notna(row['distrito']) else ''
    if distrito in turismo_comp:
        base = turismo_comp[distrito]
        year_factor = 1.0 + (row['año'] - 2014) * 0.015  # +1.5% anual
        return round(min(base * year_factor, 60), 1)
    return 0.0

df_all['competencia_pisos_turisticos'] = df_all.apply(get_competencia, axis=1)
```

**Valores base por distrito (máx 60%):**

| Distrito | % Base | Factor 2025 |
|----------|--------|-------------|
| Ciutat Vella | 55% | 60% (cap) |
| Poblats Maritims | 40% | 47% |
| Eixample | 25% | 29% |
| Extramurs | 15% | 18% |
| Benimaclet | 10% | 12% |

---

## 10. Paso 8 — Bloque 6: KPIs precalculados para Looker

Columnas pre-agregadas para que los scorecards de Looker Studio funcionen con simples AVG/SUM.

### 10.1 `es_precio_anomalo`

Detecta outliers fuera de 3 desviaciones estándar de la media del grupo.

```python
df_all['es_precio_anomalo'] = False

for (year, ccaa, tv), group in df_all.groupby(['año', 'ccaa', 'tipo_vivienda']):
    mean_v = group['precio_mes_eur'].mean()
    std_v = group['precio_mes_eur'].std()
    if std_v > 0:
        lower = mean_v - 3 * std_v
        upper = mean_v + 3 * std_v
        df_all.loc[group.index, 'es_precio_anomalo'] = (
            (group['precio_mes_eur'] < lower) | (group['precio_mes_eur'] > upper)
        )
```

### 10.2 `ranking_precio_provincia`

Ranking de provincia por precio medio ese año (1 = más cara).

```python
df_all['ranking_precio_provincia'] = 0

for year in df_all['año'].unique():
    year_mask = df_all['año'] == year
    prov_avg = df_all.loc[year_mask].groupby('provincia')['precio_mes_eur'].mean().rank(ascending=False)
    df_all.loc[year_mask, 'ranking_precio_provincia'] = (
        df_all.loc[year_mask, 'provincia'].map(prov_avg).fillna(0).astype(int)
    )
```

### 10.3 `alerta_accesibilidad`

Semáforo para conditional formatting en dashboard.

```python
def get_alerta(row):
    esfuerzo = row['esfuerzo_economico_pct']
    if pd.isna(esfuerzo):
        return np.nan
    if esfuerzo < 30:
        return 'VERDE'
    elif esfuerzo < 40:
        return 'AMARILLO'
    elif esfuerzo < 50:
        return 'NARANJA'
    else:
        return 'ROJO'

df_all['alerta_accesibilidad'] = df_all.apply(get_alerta, axis=1)
```

**Criterios del semáforo:**

| Alerta | Esfuerzo económico | Acción recomendada |
|--------|-------------------|-------------------|
| VERDE | < 30% | Saludable |
| AMARILLO | 30–40% | Monitorear |
| NARANJA | 40–50% | Intervención necesaria |
| ROJO | > 50% | Crítico |

### 10.4 `segmento_mercado`

Dimensión combinada para filtrado rápido en Looker.

```python
df_all['segmento_mercado'] = (
    df_all['ccaa'].astype(str) + '_' +
    df_all['tipo_alquiler'].astype(str) + '_' +
    df_all['tipo_vivienda'].astype(str)
)
```

**Ejemplo:** `"Comunitat_Valenciana_estudiantil_habitacion"`

### 10.5 `decada_precio`

Buckets de precio para histogramas.

```python
def get_decada(precio):
    if pd.isna(precio):
        return np.nan
    brackets = [
        (0, 100, '0-100'), (100, 200, '100-200'), (200, 300, '200-300'),
        (300, 400, '300-400'), (400, 500, '400-500'), (500, 600, '500-600'),
        (600, 700, '600-700'), (700, 800, '700-800'), (800, 900, '800-900'),
        (900, 1000, '900-1000'), (1000, 1250, '1000-1250'),
        (1250, 1500, '1250-1500'), (1500, 2000, '1500-2000'),
    ]
    for lo, hi, label in brackets:
        if lo <= precio < hi:
            return label
    return '2000+'

df_all['decada_precio'] = df_all['precio_mes_eur'].apply(get_decada)
```

---

## 11. Paso 9 — Formateo y exportación final

### 11.1 Limpieza y formato

```python
# Reemplazar infinitos por NaN
df_all = df_all.replace([np.inf, -np.inf], np.nan)

# Redondear floats a 2 decimales
float_cols = df_all.select_dtypes(include=['float64']).columns
for col in float_cols:
    df_all[col] = df_all[col].round(2)

# Booleanos a lowercase string (requisito Looker)
bool_cols = df_all.select_dtypes(include=['bool']).columns
for col in bool_cols:
    df_all[col] = df_all[col].map(lambda x: 'true' if x else 'false')

# NULLs a cadena vacía
df_all = df_all.where(pd.notna(df_all), None)

# Formato fecha
df_all['fecha_referencia'] = df_all['fecha_referencia'].astype(str)

# Ordenar por id_registro
df_all = df_all.sort_values('id_registro').reset_index(drop=True)
```

### 11.2 Orden de columnas

```python
original_cols = list(df.columns)
new_cols = [
    'precio_mes_eur_media_movil_3y', 'precio_mes_eur_media_movil_5y',
    'precio_index_2014_100', 'variacion_precio_mensual_pct',
    'variacion_precio_acumulada_desde_2014_pct', 'precio_percentil_nacional',
    'precio_percentil_ccaa', 'tendencia_precio',
    'salario_minimo_interprofesional', 'meses_smi_para_alquiler',
    'esfuerzo_smi_pct', 'ipc_anual_espana', 'precio_real_deflactado',
    'brecha_precio_compra_alquiler', 'precio_compra_m2_estimado',
    'tasa_paro_ccaa', 'euribor_12m', 'tipo_contrato_predominante',
    'ley_vivienda_aplicable', 'cap_precio_ley_vivienda',
    'precio_sobre_cap_legal', 'perfil_inquilino_estimado',
    'rango_precio_categoria', 'demanda_estimada_categoria',
    'tiempo_hasta_alquiler_categoria', 'oferta_relativa_zona',
    'curso_academico', 'precio_vs_media_estudiantil_pct',
    'zona_universitaria_calidad', 'competencia_pisos_turisticos',
    'es_precio_anomalo', 'ranking_precio_provincia',
    'alerta_accesibilidad', 'segmento_mercado', 'decada_precio',
]
column_order = original_cols + new_cols
column_order = [c for c in column_order if c in df_all.columns]
df_all = df_all[column_order]
```

### 11.3 Exportación

```python
output_path = r'alquileres_espana_valencia_estudiantes_2014_2025.csv'

df_all.to_csv(
    output_path,
    index=False,
    encoding='utf-8-sig',  # UTF-8 with BOM
    na_rep=''               # NULL → cadena vacía
)

print(f"Done! {len(df_all)} rows x {len(df_all.columns)} cols")
print(f"Saved to: {output_path}")
print(f"New columns: {len(new_cols)}")
```

**Resultado final:**
```
Done! 34003 rows x 83 cols
Saved to: alquileres_espana_valencia_estudiantes_2014_2025.csv
New columns: 35
```

---

## 12. Verificación de calidad

```python
import pandas as pd

df = pd.read_csv(
    r'alquileres_espana_valencia_estudiantes_2014_2025.csv',
    low_memory=False
)

print('Shape:', df.shape)
print('Columns:', len(df.columns))
print()

# Verificar años
print('Years:')
print(df['año'].value_counts().sort_index())
print()

# Verificar 2025
d25 = df[df['año'] == 2025]
print('2025 rows:', len(d25))
print('2025 trimestres:', d25['trimestre'].value_counts().to_dict())
print('2025 precio medio:', round(d25['precio_mes_eur'].mean(), 2))
print('2025 calidad_dato:', d25['calidad_dato'].unique())
print()

# Verificar columnas nuevas
new_cols = [
    'precio_mes_eur_media_movil_3y', 'precio_mes_eur_media_movil_5y',
    'precio_index_2014_100', 'variacion_precio_mensual_pct',
    'variacion_precio_acumulada_desde_2014_pct', 'precio_percentil_nacional',
    'precio_percentil_ccaa', 'tendencia_precio',
    'salario_minimo_interprofesional', 'meses_smi_para_alquiler',
    'esfuerzo_smi_pct', 'ipc_anual_espana', 'precio_real_deflactado',
    'brecha_precio_compra_alquiler', 'precio_compra_m2_estimado',
    'tasa_paro_ccaa', 'euribor_12m', 'tipo_contrato_predominante',
    'ley_vivienda_aplicable', 'cap_precio_ley_vivienda',
    'precio_sobre_cap_legal', 'perfil_inquilino_estimado',
    'rango_precio_categoria', 'demanda_estimada_categoria',
    'tiempo_hasta_alquiler_categoria', 'oferta_relativa_zona',
    'curso_academico', 'precio_vs_media_estudiantil_pct',
    'zona_universitaria_calidad', 'competencia_pisos_turisticos',
    'es_precio_anomalo', 'ranking_precio_provincia',
    'alerta_accesibilidad', 'segmento_mercado', 'decada_precio',
]
missing = [c for c in new_cols if c not in df.columns]
print('Missing new columns:', missing if missing else 'None')
print()

# Verificar SMI 2025
print('SMI 2025:', d25['salario_minimo_interprofesional'].unique())
print('Euribor 2025:', d25['euribor_12m'].unique())
print('IPC 2025:', d25['ipc_anual_espana'].unique())
```

**Output de verificación:**
```
Shape: (34003, 83)
Years:
2014    2025
2015    2117
...
2024    4790
2025    3002
2025 rows: 3002
2025 trimestres: {'Q1': 1030, 'Q2': 996, 'Q3': 976}
2025 precio medio: 1165.19
2025 calidad_dato: ['SIMULADO']
Missing new columns: None
SMI 2025: [1184.]
Euribor 2025: [2.45]
IPC 2025: [2.3]
```

---

## Apéndice A: Diccionario de columnas nuevas

| # | Columna | Tipo | Bloque |
|---|---------|------|--------|
| 49 | `precio_mes_eur_media_movil_3y` | FLOAT | B1 - Tendencia |
| 50 | `precio_mes_eur_media_movil_5y` | FLOAT | B1 - Tendencia |
| 51 | `precio_index_2014_100` | FLOAT | B1 - Tendencia |
| 52 | `variacion_precio_mensual_pct` | FLOAT | B1 - Tendencia |
| 53 | `variacion_precio_acumulada_desde_2014_pct` | FLOAT | B1 - Tendencia |
| 54 | `precio_percentil_nacional` | INTEGER (1-100) | B1 - Tendencia |
| 55 | `precio_percentil_ccaa` | INTEGER (1-100) | B1 - Tendencia |
| 56 | `tendencia_precio` | STRING | B1 - Tendencia |
| 57 | `salario_minimo_interprofesional` | FLOAT | B2 - Accesibilidad |
| 58 | `meses_smi_para_alquiler` | FLOAT | B2 - Accesibilidad |
| 59 | `esfuerzo_smi_pct` | FLOAT | B2 - Accesibilidad |
| 60 | `ipc_anual_espana` | FLOAT | B2 - Accesibilidad |
| 61 | `precio_real_deflactado` | FLOAT | B2 - Accesibilidad |
| 62 | `brecha_precio_compra_alquiler` | FLOAT | B2 - Accesibilidad |
| 63 | `precio_compra_m2_estimado` | FLOAT | B2 - Accesibilidad |
| 64 | `tasa_paro_ccaa` | FLOAT | B3 - Contexto |
| 65 | `euribor_12m` | FLOAT | B3 - Contexto |
| 66 | `tipo_contrato_predominante` | STRING | B3 - Contexto |
| 67 | `ley_vivienda_aplicable` | BOOLEAN | B3 - Contexto |
| 68 | `cap_precio_ley_vivienda` | FLOAT | B3 - Contexto |
| 69 | `precio_sobre_cap_legal` | FLOAT | B3 - Contexto |
| 70 | `perfil_inquilino_estimado` | STRING | B4 - Perfil |
| 71 | `rango_precio_categoria` | STRING | B4 - Perfil |
| 72 | `demanda_estimada_categoria` | STRING | B4 - Perfil |
| 73 | `tiempo_hasta_alquiler_categoria` | STRING | B4 - Perfil |
| 74 | `oferta_relativa_zona` | STRING | B4 - Perfil |
| 75 | `curso_academico` | STRING | B5 - Estudiante |
| 76 | `precio_vs_media_estudiantil_pct` | FLOAT | B5 - Estudiante |
| 77 | `zona_universitaria_calidad` | STRING | B5 - Estudiante |
| 78 | `competencia_pisos_turisticos` | FLOAT | B5 - Estudiante |
| 79 | `es_precio_anomalo` | BOOLEAN | B6 - KPIs |
| 80 | `ranking_precio_provincia` | INTEGER | B6 - KPIs |
| 81 | `alerta_accesibilidad` | STRING | B6 - KPIs |
| 82 | `segmento_mercado` | STRING | B6 - KPIs |
| 83 | `decada_precio` | STRING | B6 - KPIs |

---

## Apéndice B: Script completo

El script completo de Python está disponible en:
- `build_2025_dataset.py` (637 líneas)

Para ejecutarlo de principio a fin:

```bash
python build_2025_dataset.py
```

**Requisitos:**
- Python 3.10+
- pandas 2.0+
- numpy 1.24+

**Tiempo de ejecución estimado:** ~2-5 minutos en hardware moderno.

---

## 13. Fase 3 — Enriquecimiento con datos estudiantiles y nuevas fuentes

Se ha realizado una tercera fase de enriquecimiento del dataset, centrada en:
1. Datos desagregados por **distrito y barrio de Valencia** con precios reales de Idealista (2024-2025)
2. Datos de **alquiler de habitaciones para estudiantes** por zona universitaria
3. **10 nuevas columnas** que miden accesibilidad, esfuerzo económico y presión sobre estudiantes
4. **588 nuevas filas** (448 por distrito/año + 140 estudiantiles)

### 13.1 Nuevas fuentes incorporadas

| Código | Fuente | Descripción | Tipo |
|--------|--------|-------------|------|
| C1 | Idealista — Precios por barrios Valencia 2024-2025 | Precio del alquiler por m² en 19 distritos de Valencia. Datos de marzo 2025 y cierre 2024 | REAL |
| C2 | INE — IPVA (Índice de Precios de Vivienda en Alquiler) | Variación anual del precio del alquiler por CCAA y capitales. Base 2015 | REAL |
| C3 | Ministerio de Vivienda — IRAV | Índice de Referencia de Arrendamientos de Vivienda (Ley 12/2023) | REAL |
| C4 | UV — DataEnhance: Precios compra/alquiler barrios Valencia | Dataset de la Universitat de València con precios por barrio (2022) | REAL |
| C5 | Gesrooms / Uniplaces / Pisos.com / Erasmusu | Precios de alquiler de habitaciones para estudiantes en Valencia | REAL |
| C6 | ODS 11 — Naciones Unidas | Indicadores de vivienda adecuada, esfuerzo económico y sobrecarga | REAL |

### 13.2 Pasos de descarga, cruce y limpieza

```python
import pandas as pd
import numpy as np

# Carga del dataset existente (v2.0)
df = pd.read_csv('dataset_original.csv', low_memory=False, encoding='utf-8-sig')

# Definición de 19 distritos de Valencia con datos de Idealista
# Fuente: valenciaextra.com, lasprovincias.es, idealista.com
distritos_valencia = {
    'Ciutat Vella': {'barrios': [...], 'precio_m2_2024': 18.6, ...},
    'Eixample': {'barrios': [...], 'precio_m2_2024': 17.1, ...},
    ...
}

# Cruce: se usa distrito + año como clave primaria para Valencia ciudad
# Para cada distrito y año (2018-2024) se genera 1 fila por barrio
for nombre_distrito, datos in distritos_valencia.items():
    for año in años_a_generar:
        precio_m2 = datos['precio_m2_2024'] * (ipva[año] / ipva[2024])
        ...
```

**Criterios de cruce:**
- **Clave primaria**: `distrito` + `año` para datos de Valencia ciudad
- Las columnas nuevas se cruzan usando el nombre del distrito (19 distritos oficiales)
- Para datos nacionales no-Valencia, las columnas nuevas se calculan con estimaciones generales

**Limpieza aplicada:**
- Estandarización de nombres de distrito (formato capitalizado)
- Rangos validados: precio_m2 entre 5-25 €/m², esfuerzo económico entre 5-80%
- Outliers fuera de 3σ eliminados para precio por m²
- Consistencia temporal: precios históricos ≥ precio actual × factor IPVA

### 13.3 Nuevas columnas creadas (10 columnas)

| # | Columna | Tipo | Cálculo | Fuente |
|---|---------|------|---------|--------|
| 84 | `precio_alquiler_historico` | FLOAT | Precio mensual del alquiler poblado con `precio_mes_eur` para filas existentes y valores específicos para filas nuevas de Valencia | C1, C2 |
| 85 | `precio_por_m2` | FLOAT | Precio medio por metro cuadrado calculado como `precio_mes_eur / superficie_m2` | C1 |
| 86 | `variacion_interanual_pct` | FLOAT | Variación % del precio respecto al año anterior en la misma zona. Usa `variacion_precio_anual_pct` donde existe | C2 |
| 87 | `zona_tensionada` | STRING | 'si' si el barrio/distrito está declarado zona tensionada según Ley 12/2023 | C3 |
| 88 | `distancia_universidad_km` | FLOAT | Distancia en km al campus universitario más cercano (UV, UPV). Poblado para filas Valencia | C4, B5 |
| 89 | `precio_habitacion_estudiante` | FLOAT | Precio medio estimado de habitación en piso compartido para estudiantes | C5 |
| 90 | `ratio_salario_alquiler` | FLOAT | `(precio_mes_eur / SMI_año) × 100`. Mide esfuerzo sobre salario mínimo | C2, B6 |
| 91 | `oferta_pisos_estudiantes` | INTEGER | Nº estimado de anuncios de alquiler para estudiantes en el barrio | C5 |
| 92 | `poblacion_estudiantil_zona` | INTEGER | Estimación de estudiantes universitarios residentes en el barrio | B5, C4 |
| 93 | `indice_accesibilidad` | FLOAT | Índice compuesto (0-100): inverso esfuerzo económico (50%) + cercanía uni (30%) + oferta estudiante (20%) | C1-C6 |

### 13.4 Nuevas filas añadidas (588 filas)

**Bloque A — Filas por distrito y año (448 filas):**
- 16 distritos urbanos × 7 años (2018-2024) × 4 barrios promedio = 448 filas
- Distritos: todos con mercado de alquiler significativo

**Bloque B — Filas de alquiler estudiantil (140 filas):**
- 20 zonas universitarias × 7 años (2018-2024) = 140 filas
- Zonas clave: Benimaclet, Blasco Ibañez, Russafa, Algirós, Camins al Grau, Jesús, Campanar, L'Eixample

**Zonas universitarias cubiertas:**
- **UV Blasco Ibañez**: El Pla del Real (Blasco Ibañez, Ciutat Universitària)
- **UPV Vera**: Benimaclet (Vera, Benimaclet), Camins al Grau
- **UV Tarongers**: Algirós (Algirós, Aiora)
- **Campus periféricos**: Burjassot (desde Benicalap), Moncada (CEU)

### 13.5 Datos reales vs. simulados/estimados

| Dato | Tipo | Justificación |
|------|------|---------------|
| Precio m² por distrito 2024-2025 | **REAL** | Datos directos informes Idealista para Valencia |
| Variación interanual por distrito | **REAL** | Informe Idealista + IPVA INE 2024 |
| Precios históricos (2018-2023) | **ESTIMADO** | Retroproyección usando IPVA INE (base 2015) |
| Precios habitación estudiante 2024 | **REAL** | Precios medios Gesrooms, Uniplaces, Pisos.com, Erasmusu |
| Precios habitación históricos | **ESTIMADO** | Retroproyección desde 2024 con factor IPVA |
| Población estudiantil por barrio | **ESTIMADO** | Distribución proporcional del censo universitario |
| Oferta anuncios estudiantiles | **ESTIMADO** | Basado en recuento de anuncios con factor de crecimiento |
| Distancia a universidad | **REAL** | Medidas desde centroide del barrio a campus (Google Maps) |
| Zona tensionada | **REAL** | Declaraciones oficiales Generalitat + Ley 12/2023 |
| Ratio salario/alquiler | **REAL** (SMI) / **ESTIMADO** (precio) | SMI oficial BOE |
| Índice de accesibilidad | **ESTIMADO** | Compuesto ponderado basado en literatura académica |

### 13.6 Estructura final del dataset

```
dataset_alquiler_valencia_enriquecido.csv
  Filas: 34,591
  Columnas: 91 (83 originales + 8 nuevas Fase 3 + 2 preexistentes recalculadas)
  Cobertura: 2014-2025 | Nacional + Valencia distrito/barrio + Estudiantil
  Versión: 3.0
```

**Distribución de filas:**
- Nacional original: 34,003
- Valencia barrio/distrito: +448 (C1)
- Alquiler estudiantil zona: +140 (C5)
- **Total: 34,591 filas**

### 13.7 Código Python completo

El script completo está disponible en:
- `enrich_dataset.py`

Para ejecutarlo:
```bash
python enrich_dataset.py
```

**Requisitos:** Python 3.10+, pandas 2.0+, numpy 1.24+

**Salida esperada:**
```
[1] Cargando CSV existente...
  Cargadas 34003 filas x 83 columnas
[4] Generando nuevas filas por distrito y año...
  Generadas 448 nuevas filas
[5] Generando filas de alquiler estudiantil...
  Generadas 140 filas
[6] Total filas: 34591
[9] Exportando dataset enriquecido...
  Filas: 34591 | Columnas: 91
```

---

## 14. Fase 4 — Enriquecimiento ODS, Sostenibilidad y Agenda 2030

Se ha realizado una cuarta fase de enriquecimiento del dataset, centrada en:
1. **Indicadores ODS** — 6 columnas vinculadas a los Objetivos de Desarrollo Sostenible (ODS 1, 3, 10, 11, 17)
2. **Sostenibilidad urbana** — 4 métricas de eficiencia habitacional, presión turística e índice compuesto
3. **Agenda 2030 / cumplimiento** — 3 indicadores de cumplimiento de metas (esfuerzo ≤30%, Ley 12/2023, año objetivo)
4. Backup del campo `ods_11_indicador` original como `ods_11_indicador_v2`
5. Columna `version_dataset = '4.0'`

### 14.1 Nuevas fuentes incorporadas

| Código | Fuente | Descripción | Tipo |
|--------|--------|-------------|------|
| D1 | ONU — SDG Indicators Database | Base de datos oficial de indicadores de seguimiento de los 17 ODS. Meta 11.1.1, meta 1.4.1, indicador 10.2.1 | Metodología |
| D2 | Eurostat — Housing Cost Overburden Rate | Umbral oficial europeo (>30% renta) para clasificar hogares en riesgo de exclusión habitacional | Metodología |
| D3 | UN-Habitat — City Prosperity Index | Marco metodológico del Índice de Prosperidad Urbana. Subíndice de equidad e inclusión social | Metodología |
| D4 | Ley 12/2023 de Vivienda — BOE | Ley estatal por el derecho a la vivienda. Zonas tensionadas, IRAV, topes de actualización de renta | Legal |
| D5 | Agenda 2030 — Resolución ONU A/RES/70/1 | Documento marco con las 169 metas de los 17 ODS | Metodología |

### 14.2 Nuevas columnas creadas (13 + 1 backup + 1 version)

| # | Columna | Tipo | Cálculo | Fuente |
|---|---------|------|---------|--------|
| 94 | `ods_1_riesgo_pobreza_vivienda` | FLOAT (0-100) | `esfuerzo_economico_pct × 0.6 + (sobrecarga_vivienda × 40)` | B6, C6 |
| 95 | `ods_3_hacinamiento_estimado` | STRING | `superficie_m2 / num_habitaciones` → CRITICO/ALTO/MEDIO/ADECUADO | Inmueble |
| 96 | `ods_10_brecha_acceso_regional` | FLOAT | Desviación % del precio respecto a la mediana nacional del año | Propio |
| 97 | `ods_11_vivienda_adecuada` | STRING | Combina esfuerzo económico + zona tensionada + hacinamiento | C3, C6, B6 |
| 98 | `ods_11_indicador_v2` | STRING | Backup del `ods_11_indicador` original v3.0 | Backup |
| 99 | `ods_11_meta_11_1_cumplimiento` | FLOAT (0-100) | Score ponderado: esfuerzo≤30%(40pts) + no tensión(20pts) + hacinamiento(25pts) + accesibilidad(15pts) | D1, D5 |
| 100 | `ods_17_fuente_dato_calidad` | STRING | Mapea calidad_dato + fuente_origen a etiqueta de calidad estadística | D1 |
| 101 | `sostenibilidad_densidad_habitacional` | FLOAT | `num_habitaciones / superficie_m2` | Inmueble |
| 102 | `presion_turistica_sostenibilidad` | STRING | Categorización de `competencia_pisos_turisticos` en 5 niveles | D3 |
| 103 | `indice_sostenibilidad_urbana` | FLOAT (0-100) | `(100-riesgo)×0.35 + cumplimiento×0.35 + (100-turismo)×0.15 + accesibilidad×0.15` | D3 |
| 104 | `categoria_sostenibilidad` | STRING | ≥75=SOSTENIBLE, 50-74=EN_TRANSICION, 25-49=VULNERABLE, <25=INSOSTENIBLE | D3 |
| 105 | `agenda2030_año_objetivo` | INTEGER (2014-2050) | Año estimado en que esfuerzo ≤30% con tendencia actual | D5 |
| 106 | `cumplimiento_ley_vivienda_2023` | STRING | CUMPLE/MARGINAL/INCUMPLE/NO_APLICA según topes Ley 12/2023 | D4 |
| 107 | `meta_esfuerzo_30pct_cumplida` | BOOLEAN | TRUE si esfuerzo ≤ 30% (umbral Eurostat/ONU) | D2 |
| 108 | `version_dataset` | STRING | '4.0' | Pipeline |

### 14.3 Código Python ilustrativo

```python
import pandas as pd
import numpy as np

df = pd.read_csv('dataset_alquiler_valencia_enriquecido.csv', low_memory=False, encoding='utf-8-sig')

# --- A1: ods_1_riesgo_pobreza_vivienda ---
sobrecarga_int = df['sobrecarga_vivienda'].astype(int)
df['ods_1_riesgo_pobreza_vivienda'] = (df['esfuerzo_economico_pct'] * 0.6 + sobrecarga_int * 40).clip(0, 100).round(2)

# --- A5: ods_11_meta_11_1_cumplimiento ---
mask_esfuerzo = df['esfuerzo_economico_pct'] <= 30
mask_no_tension = df['zona_tensionada'] == 'no'
df['ods_11_meta_11_1_cumplimiento'] = (
    mask_esfuerzo.astype(int) * 40 +
    mask_no_tension.astype(int) * 20 +
    (df['ods_3_hacinamiento_estimado'].isin(['ADECUADO', 'MEDIO']).astype(int) * 25) +
    ((df['indice_accesibilidad'] >= 60).astype(int) * 15)
).clip(0, 100).round(1)

# --- B3: indice_sostenibilidad_urbana ---
df['indice_sostenibilidad_urbana'] = (
    (100 - df['ods_1_riesgo_pobreza_vivienda']) * 0.35 +
    df['ods_11_meta_11_1_cumplimiento'] * 0.35 +
    (100 - df['competencia_pisos_turisticos'].fillna(0)) * 0.15 +
    df['indice_accesibilidad'] * 0.15
).round(1)

# --- B4: categoria_sostenibilidad ---
def cat_sost(x):
    if x >= 75: return 'SOSTENIBLE'
    if x >= 50: return 'EN_TRANSICION'
    if x >= 25: return 'VULNERABLE'
    return 'INSOSTENIBLE'
df['categoria_sostenibilidad'] = df['indice_sostenibilidad_urbana'].apply(cat_sost)
```

### 14.4 Estructura final del dataset

```
dataset_alquiler_valencia_enriquecido_v4.csv
  Filas: 34,591
  Columnas: 106 (91 fase 3 + 13 nuevas fase 4 + 1 backup + 1 version)
  Cobertura: 2014-2025 | Nacional + Valencia distrito/barrio + Estudiantil + ODS
  Versión: 4.0
```

### 14.5 Validación de resultados

| Métrica | Valor |
|---------|-------|
| Columnas nuevas añadidas | 13 |
| Shape final | (34,591, 106) |
| SOSTENIBLE | 18,344 filas (53.0%) |
| EN_TRANSICION | 9,767 filas (28.2%) |
| VULNERABLE | 3,813 filas (11.0%) |
| INSOSTENIBLE | 2,667 filas (7.7%) |
| ADECUADA | 17,576 filas |
| EN_RIESGO | 9,069 filas |
| INADECUADA | 3,420 filas |
| CRITICA | 4,526 filas |
| % meta_esfuerzo_30pct_cumplida = true | 55.1% |
| % cumplimiento_ley_vivienda_2023 = CUMPLE | 7.7% |
| Mediana ods_11_meta_11_1_cumplimiento | 75.0 |
| Mediana indice_sostenibilidad_urbana | 80.6 |

### 14.6 Código Python completo

El script completo está disponible en:
- `enrich_fase4_ods.py`

Para ejecutarlo:
```bash
python enrich_fase4_ods.py
```

**Requisitos:** Python 3.10+, pandas 2.0+, numpy 1.24+

**Salida esperada:**
```
[1] Cargando CSV existente (v3.0)...
  Shape: (34591, 91)
[2] Bloque A - Indicadores ODS...
[3] Bloque B - Sostenibilidad Urbana...
[4] Bloque C - Agenda 2030 / Cumplimiento...
[7] Exportando dataset v4.0...
  Shape final: (34591, 106)
Fase 4 completada con exito!
```

---
