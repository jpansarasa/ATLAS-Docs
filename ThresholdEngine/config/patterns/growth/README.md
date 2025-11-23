# Growth Patterns

Patterns detecting economic expansion and growth acceleration signals.

## Patterns

| Pattern ID | Name | Trigger | Signal Range |
|------------|------|---------|--------------|
| gdp-acceleration | GDP Growth Acceleration | GDP >3% YoY | +1 to +2 |
| ism-expansion | ISM Manufacturing Expansion | ISM >52 and rising | +1 to +2 |
| industrial-production | Industrial Production Growth | INDPRO >3% YoY | +1 to +2 |
| retail-sales-surge | Retail Sales Surge | RSAFS >5% YoY | +1 to +2 |
| housing-starts | Housing Starts Strength | HOUST >1.5M | +1 to +2 |

## Pattern Details

### GDP Acceleration
- **Source**: Bureau of Economic Analysis
- **Moderate Growth**: >3% YoY
- **Strong Growth**: >4% YoY

### ISM Expansion
- **Source**: Institute for Supply Management
- **Expansion**: ISM >50
- **Strong Expansion**: ISM >55, Boom: >58

### Industrial Production
- **Source**: Federal Reserve
- **Trigger**: Industrial Production Index >3% YoY
- **Strong Growth**: >5% YoY

### Retail Sales
- **Source**: Census Bureau
- **Trigger**: Retail and Food Services >5% YoY
- **Strong Growth**: >8% YoY

### Housing Starts
- **Source**: Census Bureau
- **Healthy Level**: >1.5M annualized
- **Strong Level**: >1.7M annualized

## FRED Series Used

- `GDP` - Real Gross Domestic Product
- `NAPM` - ISM Manufacturing PMI
- `INDPRO` - Industrial Production Index
- `RSAFS` - Advance Retail Sales
- `HOUST` - Housing Starts

## Applicable Regimes

All growth patterns apply to: Recovery, Growth
