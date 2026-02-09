```mermaid
graph TD
    START["Traction Report URL<br/>{dealer_web_url}/dealers/{dealer_id}/leads/{lead_id}"]
    
    START --> SCENARIO{Usage Scenario}
    
    %% Lead Reengagement Branch
    SCENARIO -->|Lead Reengagement| LRE[Shopper Alert Flow<br/>Competing leads detected]
    LRE --> LRE_IMPL{Implementation}
    
    LRE_IMPL -->|Leads Platform API| LP["AlertPayloadBuilder<br/>build_traction_report_url/2"]
    LP --> LP_PARAMS["✓ utm_content=NOCL<br/>✓ utm_medium=email"]
    
    LRE_IMPL -->|CRM Delivery| CRM["Formatter<br/>build_lead_reengagement_<br/>traction_report_section/3"]
    CRM --> CRM_MEDIUM{Medium Type}
    CRM_MEDIUM -->|Email| CRM_EMAIL["✓ utm_content=NOCL<br/>✓ utm_medium=email"]
    CRM_MEDIUM -->|API| CRM_API["✓ utm_content=NOCL<br/>✓ utm_medium=api"]
    
    CRM_EMAIL --> CRM_TARGETS["Delivered to:<br/>• AutoNation ADF<br/>• Avis ADF<br/>• CarMax JSON"]
    CRM_API --> CRM_TARGETS
    
    %% New Lead Branch
    SCENARIO -->|New Lead Delivery| NLD[New Lead with Shopper Details<br/>Lead has trip_id]
    NLD --> NLD_IMPL{Implementation}
    
    NLD_IMPL -->|Email Source| NLD_EMAIL["Formatter<br/>get_traction_report_url/3<br/>source: :email"]
    NLD_EMAIL --> NLD_EMAIL_PARAMS["✓ utm_content=SD0<br/>✓ utm_medium=email or api"]
    
    NLD_IMPL -->|API Source| NLD_API["Formatter<br/>get_traction_report_url/3<br/>source: :api"]
    NLD_API --> NLD_API_PARAMS["✓ utm_content=SD0<br/>✓ utm_medium=api<br/>✓ utm_source=api"]
    
    NLD_EMAIL_PARAMS --> NLD_TARGETS["Delivered to:<br/>• AutoNation ADF<br/>• Avis ADF<br/>• Carvana JSON<br/>• CarMax JSON"]
    NLD_API_PARAMS --> NLD_TARGETS
    
    %% Email Template Branch
    SCENARIO -->|Email Template| EMAIL[Lead Delivery Email<br/>HTML Template]
    EMAIL --> EMAIL_IMPL["LeadDeliveryView<br/>build_traction_profile_url/2"]
    EMAIL_IMPL --> EMAIL_PARAMS["✓ utm_content=SD0<br/>✓ utm_medium=email"]
    EMAIL_PARAMS --> EMAIL_TARGET["Used in:<br/>• Email HTML/Text templates"]
    
    %% Dealer Web UI Branch
    SCENARIO -->|Dealer Web UI| DWU[Internal Navigation<br/>Prospects Table]
    DWU --> DWU_IMPL["ProspectsView<br/>get_traction_report_url_by_lead/2"]
    DWU_IMPL --> DWU_PARAMS["✗ No UTM parameters"]
    DWU_PARAMS --> DWU_TARGET["Phoenix route:<br/>~p'/dealers/{id}/leads/{id}'"]
    
    classDef scenario fill:#fff4e6,stroke:#ff9800,stroke-width:3px
    classDef impl fill:#f3e5f5,stroke:#9c27b0,stroke-width:2px
    classDef params fill:#e1f5ff,stroke:#0066cc,stroke-width:2px
    classDef target fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
    classDef decision fill:#ffebee,stroke:#f44336,stroke-width:2px
    
    class START scenario
    class LRE,NLD,EMAIL,DWU scenario
    class LP,CRM,NLD_EMAIL,NLD_API,EMAIL_IMPL,DWU_IMPL impl
    class LP_PARAMS,CRM_EMAIL,CRM_API,NLD_EMAIL_PARAMS,NLD_API_PARAMS,EMAIL_PARAMS,DWU_PARAMS params
    class CRM_TARGETS,NLD_TARGETS,EMAIL_TARGET,DWU_TARGET target
    class SCENARIO,LRE_IMPL,NLD_IMPL,CRM_MEDIUM decision
  ```

## Traction Report URL Query Parameters Reference

### URL Structure

{dealer_web_url}/dealers/{dealer_id}/leads/{lead_id}?{query_parameters}

### Parameter Combinations by Scenario

| Scenario | utm_content | utm_medium | utm_source | Implementation Location |
|----------|-------------|------------|------------|------------------------|
| **Lead Reengagement** | NOCL | email/api | - | AlertPayloadBuilder, Formatter |
| **New Lead (Email)** | SD0 | email/api | - | Formatter.get_traction_report_url (source: :email) |
| **New Lead (API)** | SD0 | api | api | Formatter.get_traction_report_url (source: :api) |
| **Email Template** | SD0 | email | - | LeadDeliveryView.build_traction_profile_url |
| **Dealer Web UI** | - | - | - | ProspectsView (no UTM params) |

### Parameter Meanings

- **utm_content**
  - `NOCL` = New/Other Competing Leads (lead reengagement alerts)
  - `SD0` = Shopper Details (new lead delivery with tracking)
  
- **utm_medium**
  - `email` = Email delivery channel
  - `api` = API delivery channel
  
- **utm_source**
  - `api` = Only added when source parameter is `:api` in Formatter functions

### Implementation Files

- `apps/engine/lib/engine/utils/external/leads_platform/alert_payload_builder.ex`
- `apps/engine/lib/engine/leads/delivery/formatter.ex`
- `apps/email/lib/email/views/lead_delivery_view.ex`
- `apps/dealer_web/lib/dealer_web/views/prospects_view.ex`

### Delivery Targets

- **CarMax**: JSON format (both NOCL and SD0)
- **Carvana**: JSON format (SD0 only)
- **AutoNation**: ADF format (both NOCL and SD0)
- **Avis**: ADF format (both NOCL and SD0)
- **Leads Platform API**: JSON payload (NOCL only)
- **Email Templates**: HTML/Text (SD0 only)