# Dynamic Competitor Pricing para Odoo

[![Odoo](https://img.shields.io/badge/Odoo-17%20%7C%2018%20%7C%2019-714B67?logo=odoo&logoColor=white)](https://www.odoo.com/)
[![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![MIT](https://img.shields.io/badge/Licencia-MIT-green)](LICENSE)

[Español](#español) · [English](#english)

---

## Español

### Índice

- [Descripción](#descripción)
- [Características](#características)
- [Requisitos](#requisitos)
- [Instalación](#instalación)
- [Configuración por producto](#configuración-por-producto)
- [Automatización (CRON)](#automatización-cron)
- [Panel de resultados y ROI](#panel-de-resultados-y-roi)
- [Importación masiva de URLs](#importación-masiva-de-urls)
- [Alertas de competencia](#alertas-de-competencia)
- [Configuración avanzada](#configuración-avanzada)
- [Estructura del módulo](#estructura-del-módulo)
- [Límites y buenas prácticas](#límites-y-buenas-prácticas)
- [Licencia](#licencia)

### Descripción

Módulo de **Odoo 17, 18 y 19** que ajusta el precio de venta de tus productos a partir del **precio de la competencia**, obtenido por scraping. Define una regla de ajuste, un margen mínimo de seguridad y un umbral de alerta; un CRON diario recorre el catálogo activo, actualiza `list_price` y deja registro de cada cambio.

### Características

- **Scraping automatizado** de fichas de producto (Amazon, Google Shopping y otras).
- **Cuatro reglas de ajuste**: competidor más barato menos un porcentaje, menos un importe fijo, media de competidores y coincidencia de tendencia predictiva.
- **Protección de margen**: nunca baja de `standard_price` + margen mínimo configurado.
- **Alertas de fluctuación**: si un competidor se mueve más allá del umbral, deja un aviso en el chatter y crea una actividad *To-Do*.
- **ROI de la estrategia**: compara unidades y facturación del periodo activo con el periodo anterior equivalente.
- **Histórico auditable**: botón *Price Fluctuation* con gráfico y lista de cambios.
- **Importación masiva por CSV** y listado centralizado de URLs exportable a Excel.
- **Anti-bloqueo**: rotación de *User-Agents*, retardos aleatorios y soporte de proxies o APIs de scraping.
- **Multi-hilo**: varias URLs del mismo producto se consultan en paralelo con `ThreadPoolExecutor`.

### Requisitos

| Componente | Versión |
|---|---|
| Odoo | 17, 18 o 19 (Community o Enterprise) |
| Python | El que exija tu versión de Odoo (3.10+) |
| Librerías | `requests` y `beautifulsoup4` en el entorno de Odoo |
| Módulos Odoo | `product` y `sale` (dependencias del manifiesto) |

El módulo no incluye un `requirements.txt`: instala las dos librerías en el mismo Python que ejecuta Odoo.

### Instalación

1. Copia la carpeta `odoo-dynamic-pricing` dentro de tu directorio de *addons* y renómbrala a `dynamic_pricing` (el nombre debe ser un identificador válido de módulo).
2. Reinicia el servicio de Odoo.
3. Activa el **modo desarrollador**.
4. Ve a **Aplicaciones → Actualizar lista de aplicaciones**.
5. Busca **Dynamic Competitor Pricing** e instálalo.

```bash
pip install requests beautifulsoup4
sudo systemctl restart odoo        # según tu instalación
```

> [!IMPORTANT]
> La carpeta debe llamarse igual que el módulo para que Odoo lo detecte. Si copias el repositorio tal cual (`odoo-dynamic-pricing`), Odoo no lo verá.

### Configuración por producto

Abre **Ventas → Productos** y entra en cualquier producto. Verás la pestaña **Dynamic Competitor Pricing** con estos campos:

| Campo | Qué controla |
|---|---|
| `Enable Dynamic Pricing` | Activa la regla para ese producto |
| `Minimum Margin (%)` | Margen mínimo sobre `standard_price` |
| `Adjustment Rule` | Regla de cálculo del nuevo precio |
| `Rule Value (X)` | Porcentaje o importe fijo que aplica la regla |
| `Alert Threshold (%)` | Variación de la competencia que dispara la alerta (15 % por defecto) |
| `Competitor URLs` | Lista de URLs rivales, con su plataforma |

Reglas disponibles:

| Regla | Comportamiento |
|---|---|
| `Lowest competitor - X%` | Precio del rival más barato menos X % |
| `Lowest competitor - Fixed Amount` | Precio del rival más barato menos un importe |
| `Average competitor` | Media de los precios obtenidos |
| `Predictive Trend Match` | Ajuste según la tendencia detectada |

En la misma pestaña añade las URLs de la competencia: elige la plataforma y pega la URL de la ficha del producto.

### Automatización (CRON)

El módulo crea la acción planificada **Dynamic Pricing: Update Dynamic Prices**, que se ejecuta **una vez al día** y procesa los productos con precios dinámicos activos. Puedes cambiarla en **Ajustes → Técnico → Acciones planificadas**.

El CRON:

1. Lee las URLs de cada producto activo.
2. Scrapea los precios en paralelo.
3. Aplica la regla y comprueba el margen mínimo.
4. Actualiza `list_price` y guarda el registro del cambio.

### Panel de resultados y ROI

- **Price Fluctuation**: botón inteligente en el producto que abre el histórico de cambios en vista gráfica y de lista.
- **Impacto en ventas**: el panel de la pestaña compara unidades vendidas e ingresos del periodo con la regla activa frente al periodo anterior equivalente, usando datos de `sale.order.line`.

### Importación masiva de URLs

1. Ve a **Ventas → Productos → Import Competitor URLs**.
2. Sube un CSV con las columnas `sku`, `platform` y `url`.
3. Revisa el catálogo consolidado en **Ventas → Productos → Competitor URLs**.

La exportación nativa de Odoo sirve para generar plantillas o informes de esas URLs.

### Alertas de competencia

Cuando un competidor sube o baja el precio más allá de `Alert Threshold (%)`:

- Se escribe un aviso destacado en el **chatter del producto**.
- Se crea una **actividad To-Do** asignada al responsable del producto.

### Configuración avanzada

Con el modo desarrollador activo, en **Ajustes → Técnico → Parámetros del sistema**:

| Clave | Valor |
|---|---|
| `dynamic_pricing.proxy_url` | Proxy, p. ej. `http://usuario:clave@ip:puerto` |
| `dynamic_pricing.api_provider` | `scrapingbee` o `zenrows` |
| `dynamic_pricing.api_key` | Clave privada del proveedor |
| `dynamic_pricing.render_js` | `True` para renderizar JavaScript (SPA, React, Vue) |

El proxy y los proveedores de scraping ayudan con errores 403 y captchas, pero no los eliminan: si un sitio bloquea el scraping, la extracción fallará para ese producto.

### Estructura del módulo

```text
odoo-dynamic-pricing/
├── __init__.py
├── __manifest__.py                     # Metadatos, dependencias y datos
├── data/cron_data.xml                  # CRON diario
├── models/
│   ├── product_template.py             # Reglas, campos y actualización de precio
│   ├── scraper_utils.py                # requests + BeautifulSoup, proxies y APIs
│   ├── dynamic_competitor_url.py       # URLs de competidores
│   ├── dynamic_competitor_price_log.py # Histórico de precios rivales
│   ├── dynamic_price_log.py            # Histórico de cambios propios
│   ├── dynamic_pricing_dashboard.py    # Panel de ROI
│   └── import_competitor_url_wizard.py # Asistente CSV
├── security/ir.model.access.csv
├── views/                              # Vistas, pestañas y dashboard
├── static/description/icon.png
├── LICENSE
└── README.md
```

### Límites y buenas prácticas

- **El scraping es frágil por naturaleza**: si la tienda cambia su HTML o añade protección anti-bot, el precio no se extraerá. Revisa los logs del CRON y el histórico.
- **Respeta los términos de uso** de los sitios que consultes y la legislación aplicable. Automatizar precios no autoriza a ignorar sus condiciones ni el `robots.txt`.
- **Prueba con un producto y una URL** antes de activar el módulo en todo el catálogo.
- **El margen mínimo es tu red de seguridad**, pero depende de que `standard_price` esté bien mantenido; si el coste es incorrecto, la protección no sirve.
- **La comparación de ROI** usa la fecha del primer registro de precio como inicio del periodo activo: si borras esos registros, el cálculo se reinicia.
- **Licencia del manifiesto**: `__manifest__.py` declara `LGPL-3` mientras que el archivo `LICENSE` es MIT y el repositorio figura como MIT. Si vas a redistuir el módulo, unifica ese campo con la licencia real.

### Licencia

**MIT** — consulta [LICENSE](LICENSE).

---

## English

### Table of contents

- [Overview](#overview)
- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Per-product configuration](#per-product-configuration)
- [Automation (CRON)](#automation-cron)
- [Results panel and ROI](#results-panel-and-roi)
- [Bulk URL import](#bulk-url-import)
- [Competitor alerts](#competitor-alerts)
- [Advanced configuration](#advanced-configuration)
- [Module structure](#module-structure)
- [Limits and best practices](#limits-and-best-practices)
- [License](#license)

### Overview

An **Odoo 17, 18 and 19** module that adjusts your products' sales price from **competitor prices** obtained by scraping. You define an adjustment rule, a minimum safety margin and an alert threshold; a daily CRON walks the active catalogue, updates `list_price` and records every change.

### Features

- **Automated scraping** of product pages (Amazon, Google Shopping and others).
- **Four adjustment rules**: lowest competitor minus a percentage, minus a fixed amount, competitor average, and predictive trend matching.
- **Margin protection**: the price never drops below `standard_price` + the configured minimum margin.
- **Fluctuation alerts**: when a competitor moves beyond the threshold, a warning lands in the chatter and a *To-Do* activity is created.
- **Strategy ROI**: compares units and revenue for the active period against the equivalent previous period.
- **Auditable history**: a *Price Fluctuation* smart button with a chart and a list of changes.
- **Bulk CSV import** and a centralised, Excel-exportable URL list.
- **Anti-blocking**: *User-Agent* rotation, random delays and support for proxies or scraping APIs.
- **Multi-threading**: several URLs for the same product are fetched in parallel with `ThreadPoolExecutor`.

### Requirements

| Component | Version |
|---|---|
| Odoo | 17, 18 or 19 (Community or Enterprise) |
| Python | Whatever your Odoo version requires (3.10+) |
| Libraries | `requests` and `beautifulsoup4` in the Odoo environment |
| Odoo modules | `product` and `sale` (manifest dependencies) |

The module ships no `requirements.txt`: install both libraries into the same Python that runs Odoo.

### Installation

1. Copy the `odoo-dynamic-pricing` folder into your *addons* directory and rename it to `dynamic_pricing` (the folder must be a valid module identifier).
2. Restart the Odoo service.
3. Enable **developer mode**.
4. Go to **Apps → Update Apps List**.
5. Search for **Dynamic Competitor Pricing** and install it.

```bash
pip install requests beautifulsoup4
sudo systemctl restart odoo        # depending on your setup
```

> [!IMPORTANT]
> The folder must match the module name for Odoo to detect it. Copying the repository as-is (`odoo-dynamic-pricing`) will not work.

### Per-product configuration

Open **Sales → Products** and enter any product. You will find the **Dynamic Competitor Pricing** tab with these fields:

| Field | What it controls |
|---|---|
| `Enable Dynamic Pricing` | Turns the rule on for that product |
| `Minimum Margin (%)` | Minimum margin over `standard_price` |
| `Adjustment Rule` | Price calculation rule |
| `Rule Value (X)` | Percentage or fixed amount applied by the rule |
| `Alert Threshold (%)` | Competitor change that triggers the alert (15 % by default) |
| `Competitor URLs` | List of rival URLs, with their platform |

Available rules:

| Rule | Behaviour |
|---|---|
| `Lowest competitor - X%` | Cheapest rival price minus X % |
| `Lowest competitor - Fixed Amount` | Cheapest rival price minus a fixed amount |
| `Average competitor` | Average of the fetched prices |
| `Predictive Trend Match` | Adjustment based on the detected trend |

Add competitor URLs in the same tab: pick the platform and paste the product page URL.

### Automation (CRON)

The module creates the scheduled action **Dynamic Pricing: Update Dynamic Prices**, which runs **once a day** over products with dynamic pricing enabled. You can change it in **Settings → Technical → Scheduled Actions**.

The CRON:

1. Reads each active product's URLs.
2. Scrapes the prices in parallel.
3. Applies the rule and checks the minimum margin.
4. Updates `list_price` and stores the change log.

### Results panel and ROI

- **Price Fluctuation**: a smart button on the product that opens the change history as a chart and a list.
- **Sales impact**: the tab panel compares units sold and revenue during the active period against the equivalent previous period, using `sale.order.line` data.

### Bulk URL import

1. Go to **Sales → Products → Import Competitor URLs**.
2. Upload a CSV with the columns `sku`, `platform` and `url`.
3. Review the consolidated catalogue in **Sales → Products → Competitor URLs**.

Odoo's native export produces templates or reports for those URLs.

### Competitor alerts

When a competitor raises or lowers its price beyond `Alert Threshold (%)`:

- A highlighted warning is written to the **product chatter**.
- An **Odoo To-Do activity** is created and assigned to the product owner.

### Advanced configuration

With developer mode on, go to **Settings → Technical → System Parameters**:

| Key | Value |
|---|---|
| `dynamic_pricing.proxy_url` | Proxy, e.g. `http://user:pass@ip:port` |
| `dynamic_pricing.api_provider` | `scrapingbee` or `zenrows` |
| `dynamic_pricing.api_key` | Private provider key |
| `dynamic_pricing.render_js` | `True` to render JavaScript (SPAs, React, Vue) |

Proxies and scraping providers help with 403 errors and captchas but do not remove them: if a site blocks scraping, extraction fails for that product.

### Module structure

```text
odoo-dynamic-pricing/
├── __init__.py
├── __manifest__.py                     # Metadata, dependencies and data
├── data/cron_data.xml                  # Daily CRON
├── models/
│   ├── product_template.py             # Rules, fields and price updates
│   ├── scraper_utils.py                # requests + BeautifulSoup, proxies and APIs
│   ├── dynamic_competitor_url.py       # Competitor URLs
│   ├── dynamic_competitor_price_log.py # Rival price history
│   ├── dynamic_price_log.py            # Own price change history
│   ├── dynamic_pricing_dashboard.py    # ROI panel
│   └── import_competitor_url_wizard.py # CSV wizard
├── security/ir.model.access.csv
├── views/                              # Views, tabs and dashboard
├── static/description/icon.png
├── LICENSE
└── README.md
```

### Limits and best practices

- **Scraping is fragile by nature**: if a shop changes its HTML or adds anti-bot protection, the price will not be extracted. Check the CRON logs and the history.
- **Respect the terms of service** of the sites you query and applicable law. Price automation does not entitle you to ignore their conditions or `robots.txt`.
- **Test with one product and one URL** before enabling the module across the catalogue.
- **The minimum margin is your safety net**, but it relies on an accurate `standard_price`; a wrong cost defeats the protection.
- **ROI comparison** uses the first price log entry as the start of the active period: deleting those records resets the calculation.
- **Manifest licence**: `__manifest__.py` declares `LGPL-3` while the `LICENSE` file is MIT and the repository is listed as MIT. Align that field with the actual licence before redistributing.

### License

**MIT** — see [LICENSE](LICENSE).
