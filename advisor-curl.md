---

# API Testing Guide with cURL

This guide provides complete cURL commands to test all API endpoints. The server runs on `http://localhost:8000` by default.

**Note:** Save the `access_token` from the login response - you'll need it for all authenticated requests.

## Table of Contents
1. [Health Check](#health-check)
2. [Authentication](#authentication)
3. [Projects](#projects)
4. [Chat Sessions](#chat-sessions)
5. [Knowledge Hub Status](#knowledge-hub-status)

---

## Health Check

### Check Service Status
```bash
curl -X GET http://localhost:8000/
```

**Response:**
```json
{
  "service": "Carbon Markets Advisor",
  "status": "running",
  "docs": "/docs"
}
```

### Check Health with Knowledge Indices
```bash
curl -X GET http://localhost:8000/health
```

**Response:**
```json
{
  "status": "ok",
  "registries_loaded": ["verra", "gold_standard", "art_trees", "plan_vivo", "climate_action_reserve"],
  "total_chunks": 1234
}
```

---

## Authentication

### Register New User
Creates a new user account and returns a JWT token.

```bash
curl -X POST http://localhost:8000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "developer@example.com",
    "name": "Carbon Developer",
    "password": "securepassword123"
  }'
```

**Response (201 Created):**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer",
  "user": {
    "id": 1,
    "email": "developer@example.com",
    "name": "Carbon Developer",
    "is_active": true,
    "created_at": "2026-06-29T11:30:00.000000"
  }
}
```

### Login
Authenticates with email/password and returns a JWT token.

```bash
curl -X POST http://localhost:8000/api/auth/login \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=developer@example.com&password=securepassword123"
```

**Response:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer",
  "user": {
    "id": 1,
    "email": "developer@example.com",
    "name": "Carbon Developer",
    "is_active": true,
    "created_at": "2026-06-29T11:30:00.000000"
  }
}
```

**Note:** Use the `access_token` in the `Authorization: Bearer <token>` header for all subsequent requests.

---

## Projects

### Create Project
Create a new carbon project with detailed intake information.

```bash
curl -X POST http://localhost:8000/api/projects \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -d '{
    "name": "Congo Basin Forest Conservation",
    "description": "REDD+ project in Democratic Republic of Congo protecting 50,000 hectares of primary rainforest",
    "country": "Democratic Republic of Congo",
    "province": "Maniema",
    "region": "Namizimu",
    "area_hectares": 50000.0,
    "forest_type": "natural_forest",
    "land_tenure": "government",
    "deforestation_drivers": ["agriculture", "charcoal", "logging"],
    "annual_deforestation_rate": 2.5,
    "proposed_activities": ["conservation", "community_livelihood", "biodiversity_protection"],
    "investment_estimate_usd": 2500000.0,
    "target_credit_volume_annual": 50000.0,
    "preferred_registries": ["verra", "gold_standard"],
    "data_availability": {
      "forest_inventory": true,
      "satellite_imagery": true,
      "historical_deforestation_data": true,
      "gis_boundary_files": true,
      "land_tenure_documents": true,
      "carbon_rights_documents": false,
      "community_agreements": true,
      "biodiversity_data": false,
      "socioeconomic_baseline": false,
      "fpic_documentation": false
    }
  }'
```

**Response (201 Created):**
```json
{
  "id": 1,
  "name": "Congo Basin Forest Conservation",
  "description": "REDD+ project in Democratic Republic of Congo protecting 50,000 hectares of primary rainforest",
  "country": "Democratic Republic of Congo",
  "province": "Maniema",
  "region": "Namizimu",
  "area_hectares": 50000.0,
  "forest_type": "natural_forest",
  "land_tenure": "government",
  "deforestation_drivers": ["agriculture", "charcoal", "logging"],
  "annual_deforestation_rate": 2.5,
  "proposed_activities": ["conservation", "community_livelihood", "biodiversity_protection"],
  "investment_estimate_usd": 2500000.0,
  "target_credit_volume_annual": 50000.0,
  "preferred_registries": ["verra", "gold_standard"],
  "data_availability": {
    "forest_inventory": true,
    "satellite_imagery": true,
    "historical_deforestation_data": true,
    "gis_boundary_files": true,
    "land_tenure_documents": true,
    "carbon_rights_documents": false,
    "community_agreements": true,
    "biodiversity_data": false,
    "socioeconomic_baseline": false,
    "fpic_documentation": false
  },
  "project_type": null,
  "readiness_level": null,
  "advisory_report": null,
  "created_at": "2026-06-29T11:35:00.000000",
  "updated_at": "2026-06-29T11:35:00.000000"
}
```

### List All Projects
Get all projects owned by the authenticated user.

```bash
curl -X GET http://localhost:8000/api/projects \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

**Response:**
```json
[
  {
    "id": 1,
    "name": "Congo Basin Forest Conservation",
    "description": "REDD+ project in Democratic Republic of Congo protecting 50,000 hectares of primary rainforest",
    "country": "Democratic Republic of Congo",
    "province": "Maniema",
    "region": "Namizimu",
    "area_hectares": 50000.0,
    "forest_type": "natural_forest",
    "land_tenure": "government",
    "deforestation_drivers": ["agriculture", "charcoal", "logging"],
    "annual_deforestation_rate": 2.5,
    "proposed_activities": ["conservation", "community_livelihood", "biodiversity_protection"],
    "investment_estimate_usd": 2500000.0,
    "target_credit_volume_annual": 50000.0,
    "preferred_registries": ["verra", "gold_standard"],
    "data_availability": {
      "forest_inventory": true,
      "satellite_imagery": true,
      "historical_deforestation_data": true,
      "gis_boundary_files": true,
      "land_tenure_documents": true,
      "carbon_rights_documents": false,
      "community_agreements": true,
      "biodiversity_data": false,
      "socioeconomic_baseline": false,
      "fpic_documentation": false
    },
    "project_type": null,
    "readiness_level": null,
    "advisory_report": null,
    "created_at": "2026-06-29T11:35:00.000000",
    "updated_at": "2026-06-29T11:35:00.000000"
  }
]
```

### Get Single Project
Retrieve details of a specific project by ID.

```bash
curl -X GET http://localhost:8000/api/projects/1 \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

**Response:**
```json
{
  "id": 1,
  "name": "Congo Basin Forest Conservation",
  "description": "REDD+ project in Democratic Republic of Congo protecting 50,000 hectares of primary rainforest",
  "country": "Democratic Republic of Congo",
  "province": "Maniema",
  "region": "Namizimu",
  "area_hectares": 50000.0,
  "forest_type": "natural_forest",
  "land_tenure": "government",
  "deforestation_drivers": ["agriculture", "charcoal", "logging"],
  "annual_deforestation_rate": 2.5,
  "proposed_activities": ["conservation", "community_livelihood", "biodiversity_protection"],
  "investment_estimate_usd": 2500000.0,
  "target_credit_volume_annual": 50000.0,
  "preferred_registries": ["verra", "gold_standard"],
  "data_availability": {
    "forest_inventory": true,
    "satellite_imagery": true,
    "historical_deforestation_data": true,
    "gis_boundary_files": true,
    "land_tenure_documents": true,
    "carbon_rights_documents": false,
    "community_agreements": true,
    "biodiversity_data": false,
    "socioeconomic_baseline": false,
    "fpic_documentation": false
  },
  "project_type": null,
  "readiness_level": null,
  "advisory_report": null,
  "created_at": "2026-06-29T11:35:00.000000",
  "updated_at": "2026-06-29T11:35:00.000000"
}
```

### Update Project
Partially update project fields. Only include fields you want to change.

```bash
curl -X PATCH http://localhost:8000/api/projects/1 \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -d '{
    "investment_estimate_usd": 3000000.0,
    "target_credit_volume_annual": 60000.0,
    "data_availability": {
      "forest_inventory": true,
      "satellite_imagery": true,
      "historical_deforestation_data": true,
      "gis_boundary_files": true,
      "land_tenure_documents": true,
      "carbon_rights_documents": true,
      "community_agreements": true,
      "biodiversity_data": true,
      "socioeconomic_baseline": true,
      "fpic_documentation": true
    }
  }'
```

**Response:**
```json
{
  "id": 1,
  "name": "Congo Basin Forest Conservation",
  "description": "REDD+ project in Democratic Republic of Congo protecting 50,000 hectares of primary rainforest",
  "country": "Democratic Republic of Congo",
  "province": "Maniema",
  "region": "Namizimu",
  "area_hectares": 50000.0,
  "forest_type": "natural_forest",
  "land_tenure": "government",
  "deforestation_drivers": ["agriculture", "charcoal", "logging"],
  "annual_deforestation_rate": 2.5,
  "proposed_activities": ["conservation", "community_livelihood", "biodiversity_protection"],
  "investment_estimate_usd": 3000000.0,
  "target_credit_volume_annual": 60000.0,
  "preferred_registries": ["verra", "gold_standard"],
  "data_availability": {
    "forest_inventory": true,
    "satellite_imagery": true,
    "historical_deforestation_data": true,
    "gis_boundary_files": true,
    "land_tenure_documents": true,
    "carbon_rights_documents": true,
    "community_agreements": true,
    "biodiversity_data": true,
    "socioeconomic_baseline": true,
    "fpic_documentation": true
  },
  "project_type": null,
  "readiness_level": null,
  "advisory_report": null,
  "created_at": "2026-06-29T11:35:00.000000",
  "updated_at": "2026-06-29T11:40:00.000000"
}
```

### Generate Advisory Report
Generate a comprehensive AI-powered advisory report analyzing methodology fit across registries.

```bash
curl -X POST http://localhost:8000/api/projects/1/advisory \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

**Response (may take 15-30 seconds):**
```json
{
  "id": 1,
  "name": "Congo Basin Forest Conservation",
  "description": "REDD+ project in Democratic Republic of Congo protecting 50,000 hectares of primary rainforest",
  "country": "Democratic Republic of Congo",
  "province": "Maniema",
  "region": "Namizimu",
  "area_hectares": 50000.0,
  "forest_type": "natural_forest",
  "land_tenure": "government",
  "deforestation_drivers": ["agriculture", "charcoal", "logging"],
  "annual_deforestation_rate": 2.5,
  "proposed_activities": ["conservation", "community_livelihood", "biodiversity_protection"],
  "investment_estimate_usd": 3000000.0,
  "target_credit_volume_annual": 60000.0,
  "preferred_registries": ["verra", "gold_standard"],
  "data_availability": {
    "forest_inventory": true,
    "satellite_imagery": true,
    "historical_deforestation_data": true,
    "gis_boundary_files": true,
    "land_tenure_documents": true,
    "carbon_rights_documents": true,
    "community_agreements": true,
    "biodiversity_data": true,
    "socioeconomic_baseline": true,
    "fpic_documentation": true
  },
  "project_type": "redd_plus",
  "readiness_level": "validation_ready",
  "advisory_report": {
    "project_id": 1,
    "executive_summary": "This project is well-positioned for REDD+ certification under Verra VCS and Gold Standard. The project has strong data availability and addresses key deforestation drivers.",
    "project_classification": "Avoided Deforestation / REDD+",
    "project_type": "redd_plus",
    "readiness_level": "validation_ready",
    "methodology_fits": [
      {
        "registry": "verra",
        "methodology_name": "VM0048",
        "eligible": true,
        "fit_score": 8,
        "strengths": ["Strong baseline data", "Clear deforestation drivers", "Good community engagement"],
        "weaknesses": ["Carbon rights documentation needs strengthening"],
        "risks": ["Leakage risk from adjacent areas", "Permanence concerns"],
        "data_requirements": ["Historical deforestation rates", "Reference level calculations", "Carbon stock assessments"],
        "notes": "VM0048 is the most appropriate methodology for this project type."
      },
      {
        "registry": "gold_standard",
        "methodology_name": "GS REDD+ Methodology",
        "eligible": true,
        "fit_score": 7,
        "strengths": ["Strong community benefits", "Biodiversity co-benefits"],
        "weaknesses": ["More complex requirements", "Longer validation timeline"],
        "risks": ["Higher transaction costs"],
        "data_requirements": ["Detailed socioeconomic baseline", "Biodiversity assessments"],
        "notes": "Gold Standard offers premium pricing but requires additional documentation."
      }
    ],
    "recommended_pathway": "Verra VCS with VM0048 methodology",
    "second_best_pathway": "Gold Standard REDD+",
    "not_recommended": ["ART TREES (jurisdictional scale required)", "Plan Vivo (smaller scale focus)"],
    "main_reason": "Project scale and data availability align well with Verra's requirements",
    "biggest_risk": "Leakage from surrounding areas without landscape-level approach",
    "immediate_next_step": "Complete reference level calculation and submit to Verra for approval",
    "missing_data": [],
    "action_plan": [
      "Finalize reference level calculations",
      "Strengthen carbon rights documentation",
      "Develop leakage mitigation plan",
      "Engage with local stakeholders for FPIC process",
      "Prepare Project Design Document (PDD)"
    ],
    "generated_at": "2026-06-29T11:45:00.000000"
  },
  "created_at": "2026-06-29T11:35:00.000000",
  "updated_at": "2026-06-29T11:45:00.000000"
}
```

### Delete Project
Delete a project and all its associated chat history.

```bash
curl -X DELETE http://localhost:8000/api/projects/1 \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

**Response (200 OK):**
```json
{
  "message": "Project deleted successfully."
}
```

---

## Chat Sessions

Chat sessions allow you to have contextual conversations about specific projects with the AI advisor.

### Create Chat Session
Create a new named conversation for a project.

```bash
curl -X POST http://localhost:8000/api/projects/1/chat/sessions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -d '{
    "title": "Methodology Selection Discussion"
  }'
```

**Response (201 Created):**
```json
{
  "id": 1,
  "project_id": 1,
  "title": "Methodology Selection Discussion",
  "created_at": "2026-06-29T12:00:00.000000",
  "updated_at": "2026-06-29T12:00:00.000000"
}
```

### List Chat Sessions
Get all chat sessions for a project, ordered by newest first.

```bash
curl -X GET http://localhost:8000/api/projects/1/chat/sessions \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

**Response:**
```json
[
  {
    "id": 1,
    "project_id": 1,
    "title": "Methodology Selection Discussion",
    "created_at": "2026-06-29T12:00:00.000000",
    "updated_at": "2026-06-29T12:00:00.000000"
  },
  {
    "id": 2,
    "project_id": 1,
    "title": "Baseline Calculation Questions",
    "created_at": "2026-06-29T11:55:00.000000",
    "updated_at": "2026-06-29T11:55:00.000000"
  }
]
```

### Send Message
Send a message in a chat session. The AI responds with context from the project and knowledge base.

```bash
curl -X POST http://localhost:8000/api/projects/1/chat/sessions/1/messages \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -d '{
    "message": "What are the key differences between Verra VM0048 and Gold Standard REDD+ methodologies for my project?",
    "registries": ["verra", "gold_standard"]
  }'
```

**Response:**
```json
{
  "message": {
    "id": 1,
    "session_id": 1,
    "role": "assistant",
    "content": "Based on your project characteristics, here are the key differences between Verra VM0048 and Gold Standard REDD+ methodologies:\n\n**Verra VM0048 (Methodology for REDD+):**\n- Focuses on avoided deforestation in tropical forests\n- Requires reference level calculation using historical deforestation rates\n- Emphasizes leakage assessment and mitigation\n- Validation timeline: typically 6-12 months\n- Credit pricing: moderate market rates\n\n**Gold Standard REDD+:**\n- Stronger emphasis on sustainable development goals (SDGs)\n- Requires detailed socioeconomic baseline and community benefits\n- More rigorous biodiversity co-benefit requirements\n- Validation timeline: typically 12-18 months\n- Credit pricing: premium market rates (often 20-30% higher)\n\n**For your project:** Given your strong data availability and community engagement, both are viable. Verra offers a faster path to market, while Gold Standard may command premium prices but requires additional documentation.",
    "registries_used": ["verra", "gold_standard"],
    "created_at": "2026-06-29T12:01:00.000000"
  },
  "session_id": 1
}
```

**Note:** The `registries` field is optional. If omitted, the system searches across all available registries.

### Get Session Messages
Retrieve the full message history of a chat session.

```bash
curl -X GET http://localhost:8000/api/projects/1/chat/sessions/1/messages \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

**Response:**
```json
[
  {
    "id": 1,
    "session_id": 1,
    "role": "user",
    "content": "What are the key differences between Verra VM0048 and Gold Standard REDD+ methodologies for my project?",
    "registries_used": null,
    "created_at": "2026-06-29T12:00:30.000000"
  },
  {
    "id": 2,
    "session_id": 1,
    "role": "assistant",
    "content": "Based on your project characteristics, here are the key differences between Verra VM0048 and Gold Standard REDD+ methodologies:\n\n**Verra VM0048 (Methodology for REDD+):**\n- Focuses on avoided deforestation in tropical forests\n- Requires reference level calculation using historical deforestation rates\n- Emphasizes leakage assessment and mitigation\n- Validation timeline: typically 6-12 months\n- Credit pricing: moderate market rates\n\n**Gold Standard REDD+:**\n- Stronger emphasis on sustainable development goals (SDGs)\n- Requires detailed socioeconomic baseline and community benefits\n- More rigorous biodiversity co-benefit requirements\n- Validation timeline: typically 12-18 months\n- Credit pricing: premium market rates (often 20-30% higher)\n\n**For your project:** Given your strong data availability and community engagement, both are viable. Verra offers a faster path to market, while Gold Standard may command premium prices but requires additional documentation.",
    "registries_used": ["verra", "gold_standard"],
    "created_at": "2026-06-29T12:01:00.000000"
  }
]
```

### Delete Chat Session
Delete a chat session and all its messages.

```bash
curl -X DELETE http://localhost:8000/api/projects/1/chat/sessions/1 \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

**Response (200 OK):**
```json
{
  "message": "Chat session deleted."
}
```

---

## Knowledge Hub Status

Check the current state of loaded knowledge indices for each registry.

```bash
curl -X GET http://localhost:8000/api/knowledge/status \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

**Response:**
```json
{
  "registries_loaded": ["verra", "gold_standard", "art_trees", "plan_vivo", "climate_action_reserve"],
  "total_chunks_all": 1234,
  "detail": {
    "verra": {
      "total_chunks": 350,
      "topics": ["methodologies", "baseline", "monitoring", "validation", "pricing"]
    },
    "gold_standard": {
      "total_chunks": 280,
      "topics": ["methodologies", "sdg_co_benefits", "community", "biodiversity", "validation"]
    },
    "art_trees": {
      "total_chunks": 200,
      "topics": ["jurisdictional", "nesting", "allocation", "buffer_pool"]
    },
    "plan_vivo": {
      "total_chunks": 180,
      "topics": ["smallholder", "community", "agroforestry", "livelihoods"]
    },
    "climate_action_reserve": {
      "total_chunks": 224,
      "topics": ["us_forest", "urban_forest", "buffer_pool", "validation"]
    }
  }
}
```

---

## Enum Reference

### Registry Options
- `verra` - Verra VCS (Verified Carbon Standard)
- `gold_standard` - Gold Standard for the Global Goals
- `art_trees` - Architecture for REDD+ Transactions
- `plan_vivo` - Plan Vivo Foundation
- `climate_action_reserve` - Climate Action Reserve

### Forest Type Options
- `natural_forest` - Primary or intact natural forest
- `degraded_forest` - Forest with reduced canopy cover
- `plantation` - Planted forest (monoculture or mixed)
- `agroforestry` - Agricultural systems with trees
- `secondary_forest` - Regrown forest after disturbance
- `protected_area` - Officially protected forest area
- `community_forest` - Community-managed forest
- `mangrove` - Coastal mangrove forest
- `mixed` - Mixed forest types

### Land Tenure Options
- `government` - Government-owned land
- `community` - Community-owned or managed
- `private` - Private ownership
- `concession` - Concession rights
- `mixed` - Mixed tenure arrangements
- `unclear` - Unclear or disputed tenure

### Deforestation Driver Options
- `agriculture` - Agricultural expansion
- `logging` - Commercial or subsistence logging
- `charcoal` - Charcoal production
- `mining` - Mining activities
- `infrastructure` - Road/building construction
- `fire` - Fire-related clearing
- `livestock` - Cattle grazing
- `mixed` - Multiple drivers

### Project Type Options
- `redd` - Reduced Emissions from Deforestation
- `redd_plus` - REDD+ (includes forest enhancement)
- `arr` - Afforestation/Reforestation/Revegetation
- `ifm` - Improved Forest Management
- `alm_agroforestry` - Agroforestry under ALM
- `blue_carbon` - Coastal blue carbon (mangroves, seagrass)
- `jurisdictional_redd` - Jurisdictional-scale REDD+
- `nested_redd` - Nested REDD+ within jurisdictional program

---

## Interactive API Documentation

For interactive testing with auto-generated requests, visit:
- **Swagger UI:** http://localhost:8000/docs
- **ReDoc:** http://localhost:8000/redoc

These provide a web interface to test all endpoints with live authentication.
