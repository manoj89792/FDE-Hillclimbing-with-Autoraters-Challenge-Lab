---
name: analyze-tax-implications
description: Investigates property tax holding costs, monthly carrying costs, and acquisition cost-per-acre valuation benchmarks for a parcel.
metadata:
  adk_additional_tools:
    - get_tax_assessment_data
    - get_lot_data
---
# Analyze Tax Implications Skill

When investigating financial carrying costs, acquisition basis, or tax implications for a lot:

1. **Retrieve Official Tax Assessment Data**:
   - Locate the `parcel_apn` from the lot property records.
   - Use the `get_tax_assessment_data` tool with that `parcel_apn` to retrieve official county tax assessment details (`annual_tax_amount`, `assessed_land_value`, `assessed_improvement_value`, `total_assessed_value`, `tax_year`, `payment_status`).

2. **Calculate Exact Monthly Tax Holding Cost**:
   - Explicitly calculate and state the **Monthly Tax Holding Cost** using the formula:
     `Monthly Tax Holding Cost = Annual Tax Amount / 12`
   - Show the exact formula and math (e.g., `$23,600.00 / 12 months = $1,966.67 per month`). If the lot record's `estimated_annual_taxes` differs from the official `annual_tax_amount`, calculate the monthly holding cost for the official assessed tax (and also show the estimated figure for completeness) and highlight the pre-construction carrying cost impact over a 12-to-24 month entitlement/permitting timeline.

3. **Verify Acquisition Cost-Per-Acre Valuation Metric**:
   - Ensure the `Acquisition Basis and Holding Costs` section prominently calculates and highlights the **Acquisition Cost-Per-Acre** (`Last Sale Price / Acreage`) with its explicit mathematical formula (`$[last_sale_price] / [acreage] acres = $[cost_per_acre]/acre`), emphasizes it as a **core valuation metric**, and benchmarks it against the Assessed Land Value Per Acre (`$[assessed_land_value] / [acreage] acres`) as well as neighboring parcels' cost-per-acre.
