# GIS Lumiere APIs - README

## Base URL
Production:
https://t01bot.trst01.in

All endpoints use HTTP POST.

## 1. Satellite Image Classification
Endpoint:
POST /api/satellite-classification

Purpose:
Analyzes an AOI using Sentinel-2 imagery and land-cover datasets.

Input:
- plotcode
- start_date (YYYY-MM-DD)
- end_date (YYYY-MM-DD)
- Either:
  - coordinates (Polygon)
  - geojson file (multipart)

Output:
- AOI information
- Vegetation/water/moisture metrics (NDVI, NDWI, NDMI, MNDWI, SAVI, EVI, NBR, NDRE)
- Dynamic World classification
- Generated maps
- Metrics summary image
- AI summary (Markdown)
- Interpretation, recommendations, confidence

## 2. Deforestation Alerts
Endpoint:
POST /api/deforestation-alerts

Purpose:
Detects tree-cover loss using Hansen GFC and integrates GLAD-L, GLAD-S2 and RADD alerts.

Output:
- Hansen loss statistics
- Annual loss analysis
- Combined loss maps
- Integrated alerts map
- Metrics summary
- AI summary

## 3. Biomass Estimation
Endpoint:
POST /api/biomass-estimation

Purpose:
Estimates above-ground biomass and carbon stock.

Output:
- Mean biomass
- Total biomass
- Carbon estimation
- GEDI metrics
- ESA CCI Biomass metrics
- Sentinel vegetation metrics
- Biomass maps
- Carbon maps
- Metrics summary
- AI summary

## 4. Wetland & Peatland Detection
Endpoint:
POST /api/wetland-peatland

Purpose:
Identifies wetlands, peatlands and hydrological conditions.

Output:
- NDWI, MNDWI, NDMI
- JRC water metrics
- Dynamic World water/flooded vegetation
- Copernicus LC100 wetland screening
- GFW peatland screening
- AWD analysis
- Wetland reversal risk
- Generated maps
- Metrics summary
- AI summary

## Common Request

JSON:
{
  "plotcode":"PLOT-001",
  "start_date":"YYYY-MM-DD",
  "end_date":"YYYY-MM-DD",
  "coordinates":[[[lng,lat],...]]
}

OR multipart/form-data:
- plotcode
- start_date
- end_date
- geojson

## Common Response

All APIs return:
- status
- plotcode
- job_id
- analysis_type
- aoi
- date_range
- datasets_used
- metrics
- images (AWS S3 URLs)
- ai_summary (Markdown)
- interpretation
- recommendations
- limitations
- confidence
- llm_provider
- llm_model
