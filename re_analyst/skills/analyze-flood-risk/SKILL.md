---
name: analyze-flood-risk
description: Investigates FEMA flood zone designations, neighboring/adjacent property flood zones, drainage, runoff, and environmental risks for real estate investment opportunity analysis.
metadata:
  adk_additional_tools:
    - get_lot_data
---
# Analyze Flood Risk Skill

When evaluating environmental, flood, or geographic risks for a lot, you MUST evaluate both the subject parcel AND all neighboring/adjacent properties:

1. **Subject Parcel Flood Zone & Environmental Analysis**:
   - Identify the exact FEMA `flood_zone` classification of the subject lot (e.g., `Zone AE (High Risk)`, `Zone VE (Coastal High Hazard)`, `Zone X (Minimal Risk)`).
   - Detail specific vertical construction and engineering requirements for that zone (e.g., Base Flood Elevation (BFE) compliance, structural pilings/deep foundations for Zone VE, structural fill dirt/elevation certificates for Zone AE, stormwater management for Zone X) and NFIP/private flood insurance cost implications.
   - Evaluate `environmental_status` (e.g., Phase I ESA Clean vs. REC flagged) and specify concrete due diligence next steps (e.g., Phase II subsurface testing, soil vapor testing, remediation budgeting, or standard environmental representations).

2. **Mandatory Investigation of Neighboring / Adjacent Property Flood Risks**:
   - Check the `adjacent_parcels` field from the subject property's `get_lot_data` record.
   - If not already retrieved, call `get_lot_data` for **every parcel ID** in `adjacent_parcels`.
   - Explicitly state the exact Lot ID (`plot_id`), exact FEMA `flood_zone` classification (e.g., `Zone AE (High Risk)`, `Zone VE (Coastal High Hazard)`, `Zone X (Minimal Risk)`), `topography` (slope grade), `soil_classification`, and `easement_type` for **each neighboring/adjacent property**.

3. **Critical Comparative Hydrological, Drainage, and Runoff Analysis**:
   - In the `Risk Assessment` section of the memorandum (and summarized in the `Executive Summary`), include a dedicated subsection and Markdown comparison table analyzing **Neighboring Property Flood Risk & Shared Hydrological Impacts**:
     - **Cross-Parcel Floodplain & Storm Surge Connectivity**: Analyze how high-risk flood zones (`Zone AE` or `Zone VE`) on adjacent parcels impact the subject parcel (e.g., floodplain encroachment, elevated groundwater/water table, tailwater/backwater effects blocking stormwater outfall, coastal velocity wave action, or shared floodwall/berm requirements), or how adjacent `Zone X` parcels compare.
     - **Topographic Slope & Surface Water Runoff / Sheet Flow**: Compare the subject parcel's `topography` grade against each neighboring parcel's `topography` grade (e.g., `0-2% grade (Flat)`, `2-5% grade (Gentle slope)`, `5-12% grade (Moderate slope)`, `15%+ grade (Steep slope)`). Critically analyze whether steeper adjacent parcels will discharge uphill stormwater sheet flow and sediment runoff onto the subject parcel, or whether the subject parcel's runoff will drain toward lower/flatter adjacent parcels (creating off-site discharge liability).
     - **Easements, Soil Permeability & Development Feasibility**: Evaluate how drainage/utility/access easements and soil classifications across the subject and adjacent lots affect stormwater detention/retention basin sizing, perimeter swales, interceptor drains, civil site-work CapEx, and overall development feasibility.
