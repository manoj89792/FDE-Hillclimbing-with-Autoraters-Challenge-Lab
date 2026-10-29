---
name: analyze-opportunity
description: Investigates a land parcel and its adjacent neighboring parcels for real estate investment opportunity analysis, including core valuation metrics like acquisition cost-per-acre and neighboring hydrological risks.
metadata:
  adk_additional_tools:
    - get_lot_data
---
# Analyze Opportunity Skill

When asked to investigate a property or analyze an investment opportunity for a lot, follow every step below thoroughly:

1. **Retrieve Subject Lot Data**:
   - Use the `get_lot_data` tool to lookup the subject property attributes by its lot ID (`plot_id`).
   - Review all core details returned: `plot_id`, `parcel_apn`, `acreage`, `zoning`, `last_sale_price`, `estimated_annual_taxes`, `flood_zone`, `environmental_status`, `utility_access`, `topography`, `soil_classification`, `easement_type`, and `adjacent_parcels`.

2. **Retrieve All Neighboring / Adjacent Parcel Data (Mandatory)**:
   - Inspect the `adjacent_parcels` list returned for the subject lot.
   - You MUST call `get_lot_data` for **every single lot ID** listed in `adjacent_parcels` so you have the exact `flood_zone`, `topography`, `easement_type`, `soil_classification`, `last_sale_price`, and `acreage` for all neighboring properties.

3. **Calculate and Highlight Acquisition Cost-Per-Acre as a Core Valuation Metric (Mandatory)**:
   - Derive and state the exact mathematical **Acquisition Cost-Per-Acre** using the explicit formula:
     `Acquisition Cost-Per-Acre = Last Sale Price / Acreage`
   - Show the exact mathematical equation and resulting figure to two decimal places (e.g., `$450,000 / 1.2 acres = $375,000.00 per acre`).
   - Highlight **Acquisition Cost-Per-Acre** in bold and in a Markdown summary table as a **core valuation metric** in both the `Executive Summary` and the `Acquisition Basis and Holding Costs` sections.
   - Benchmark the subject lot's Acquisition Cost-Per-Acre against:
     - The Acquisition Cost-Per-Acre (`last_sale_price / acreage`) of each neighboring parcel in `adjacent_parcels`.
     - The County Assessed Land Value Per Acre (`assessed_land_value / acreage` from the tax assessment tool).
   - Discuss what this Cost-Per-Acre benchmark implies for acquisition pricing efficiency, basis risk, and residual land valuation.

4. **Consult Specialized Flood Risk and Tax Skills**:
   - Consult the `analyze-flood-risk` skill to perform an exhaustive analysis of both the subject parcel's FEMA flood zone AND the neighboring/adjacent properties' FEMA flood zones, drainage, runoff, and shared hydrological impacts.
   - Consult the `analyze-tax-implications` skill to retrieve official county tax records via `get_tax_assessment_data(parcel_apn)` and compute exact monthly tax holding costs (`annual_tax_amount / 12`).

5. **Report Formatting & Completeness**:
   - Include clean Markdown tables in `Acquisition Basis and Holding Costs` and `Risk Assessment` to clearly present the core specifications, cost-per-acre derivation and benchmarking, monthly tax carrying costs, and comparative neighboring parcel flood/hydrological data.
