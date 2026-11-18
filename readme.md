# Proyecto EDA: Análisis de Campañas de Marketing Bancario

Proyecto de análisis exploratorio de datos (EDA) en Python sobre campañas de marketing telefónico de una entidad bancaria portuguesa. El objetivo es analizar qué factores influyen en que un cliente contrate un depósito a plazo fijo (`y`) y extraer conclusiones útiles para futuras campañas.

---

## Estructura del proyecto

El proyecto está organizado en las siguientes carpetas:

```text
python_for_data/
├── data/
│   ├── raw/                                     # Datos originales
│   │   ├── bank-additional.csv                  # Datos de llamadas y variables macroeconómicas
│   │   └── customer-details.xlsx                # Datos demográficos de clientes (3 hojas: 2012, 2013, 2014)
│   └── processed/
│       └── bank_cleaned.csv                     # Dataset final limpio y unificado
├── notebooks/
│   ├── 01_carga_y_exploracion.ipynb             # Carga inicial, unión del excel y merge de tablas
│   ├── 02_limpieza_y_transformacion.ipynb       # Limpieza, tipos de datos, tratamiento de nulos y nuevas variables
│   └── 03_analisis_exploratorio_y_visualizacion.ipynb # Análisis estadístico, gráficos y conclusiones
├── requirements.txt
└── README.md
```

---

## Pasos seguidos en el proyecto

### 1. Carga e integración de los datos
- El archivo `customer-details.xlsx` tenía los clientes repartidos en tres hojas por año (`2012`, `2013` y `2014`). Las leí con Pandas y las uní en un único DataFrame con `pd.concat()`. En total eran 43.170 clientes.
- Crucé esa tabla con `bank-additional.csv` mediante un `merge` usando como clave el identificador del cliente (`id_` en el CSV e `ID` en el Excel).
- Los 43.000 registros de llamadas tenían su correspondencia en la base de clientes. Los 170 restantes del Excel eran clientes a los que nunca se llegó a llamar en esta campaña.
- Eliminé columnas redundantes como índices automáticos (`Unnamed: 0`) y el ID duplicado tras la unión.

### 2. Limpieza y preparación de los datos
Durante la primera revisión detecté varios problemas que corregí en el segundo notebook:
- **Separadores decimales**: Columnas como `euribor3m`, `cons.price.idx`, `cons.conf.idx` y `nr.employed` venían con comas en lugar de puntos (por ejemplo `93,994`). Pandas las leía como texto (`object`). Las convertí a números flotantes reemplazando la coma por punto.
- **Nombres de columnas**: Estandaricé todos los nombres a minúsculas y cambié los puntos por guiones bajos (ejemplo: `emp.var.rate` a `emp_var_rate`).
- **Fechas en español**: La columna `date` contenía textos como `2-agosto-2019`. Creé una función para parsearlas a tipo `datetime` y extraje el mes (`contact_month`) y el año (`contact_year`).
- **Tratamiento de nulos**:
  - `age` (5.120 nulos): En vez de meter una media general para todos, imputé la mediana según la profesión (`job`). Por ejemplo, la edad típica de un estudiante ronda los 25 años y la de un jubilado los 59, por lo que rellenar con la media global de 40 habría falseado los grupos.
  - `euribor3m` (9.256 nulos): Comprobé que el Euribor tiene una correlación de más del 97% con la tasa de variación de empleo (`emp_var_rate`), así que imputé sus nulos usando la mediana de su correspondiente nivel de empleo.
  - Categóricas (`job`, `education`, `marital`): Asigné el valor `'unknown'`, habitual en este tipo de registros bancarios cuando el cliente no aporta el dato.
  - Productos financieros (`default`, `housing`, `loan`): Asigné `-1` a los casos desconocidos para no perder filas ni asumir falsos negativos.
- **Nuevas variables**:
  - `total_children`: suma de niños y adolescentes en casa (`kidhome + teenhome`).
  - `previously_contacted`: indicador binario de si ya se había contactado con el cliente en campañas anteriores (`pdays != 999`).
  - `age_group`: división en 4 tramos de edad (`<30`, `30-44`, `45-59`, `60+`).

El dataset limpio se guardó en `data/processed/bank_cleaned.csv` con 43.000 filas y 33 columnas.

---

## Principales conclusiones del análisis

### 1. Tasa de conversión general
De los 43.000 contactos realizados, contrataron el depósito 4.844 clientes, lo que supone una tasa de conversión global del **11,27%**.

### 2. Perfil del cliente que más contrata
- **Edad**: El grupo más receptivo con mucha diferencia son los **mayores de 60 años**, con una conversión del **39,4%** (casi 4 veces la media). Los jóvenes menores de 30 años también convierten por encima de la media (**16,4%**). En cambio, la franja de 30 a 59 años no llega al 10%, probablemente por tener más cargas familiares e hipotecas.
- **Profesión**: Los perfiles con mayor tasa de éxito son **estudiantes (31,3%)** y **jubilados (25,2%)**. Por el contrario, los obreros (`blue-collar`), pese a ser uno de los grupos a los que más se llama, solo convierten un **6,9%**.
- **Educación**: Los clientes con estudios universitarios tienen una conversión superior (**13,7%**) frente a quienes tienen estudios básicos (**7,8% - 10,3%**).

### 3. Canales y estrategia de contacto
- **Móvil vs Fijo**: Las llamadas a móviles consiguen un **14,7%** de conversión frente a un **5,2%** en teléfonos fijos (casi el triple).
- **Clientes ya conocidos**: Los clientes a los que ya se había contactado en campañas previas tienen una tasa de contratación del **64,0%**, frente al **9,2%** de los clientes nuevos. Es el factor individual que más dispara la probabilidad de venta.
- **Número de llamadas en la campaña**:
  - 1 llamada: 13,0%
  - 2 llamadas: 11,5%
  - 3 llamadas: 10,8%
  - 4 llamadas: 9,3%
  - Más de 6 llamadas: menos del 7%
  A partir de la tercera llamada la efectividad cae notablemente. Insistir más de 3 o 4 veces no suele funcionar y satura al cliente.

### 4. Contexto económico
- El Euribor a 3 meses tiene una correlación negativa con la contratación (-0,31). Cuando los tipos están bajos (<1,3%), la tasa de contratación sube a casi el **24%**, ya que los clientes prefieren asegurar rentabilidad fija ante la incertidumbre.

---

## Recomendaciones para el banco

1. **Priorizar el recontacto**: Enfocar las primeras llamadas en clientes que ya participaron en campañas anteriores, donde la tasa de éxito supera el 60%.
2. **Focalizar por perfiles**: Lanzar campañas específicas para personas mayores de 60 años / jubilados (mayor capacidad de ahorro estable) y productos adaptados para jóvenes y estudiantes.
3. **Dejar de usar el teléfono fijo**: Centralizar la operativa en llamadas a teléfonos móviles.
4. **Límite de contactos por campaña**: Establecer una regla interna de máximo 3 intentos por cliente. Si tras 3 llamadas no hay interés, es mejor parar para no malgastar tiempo del equipo comercial.
5. **Aprovechar los momentos de tipos bajos**: Reforzar las campañas de depósitos garantizados cuando el Euribor baje.

---

## Requisitos para ejecutar el proyecto

Instalar las librerías necesarias ejecutando:

```bash
pip install -r requirements.txt
```

## Autor

Jose Ayala Farell