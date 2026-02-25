```mermaid
flowchart LR
    subgraph Triggers["1. Trigger Event"]
        NL["New Lead"]
        CL["Competing Lead"]
        WIS["Walk-In Self"]
        WIC["Walk-In Competitor"]
    end

    subgraph Channel["2. Delivery Channel"]
        EMAIL["Email"]
        API["API"]
    end

    subgraph UTM["3. UTM Params<br/>(content / medium / source)"]
        SD0_E["SD0 / email"]
        SD0_A["SD0 / api / api"]
        NOCL_E["NOCL / email"]
        NOCL_A["NOCL / api"]
        WKINS_E["WKINS / email"]
        WKINS_A["WKINS / api"]
        WKINC_E["WKINC / email"]
        WKINC_A["WKINC / api"]
    end

    subgraph Dest["4. Destination"]
        XML["Dealer XML Email"]
        CRM["Major Account CRM<br/>(Carmax, AutoNation, Avis)<br/>direct API"]
        LP["Leads Platform API<br/>(routes to downstream CRM<br/>via method field)"]
    end

    DW(("Dealer Web<br/>Shopper Details"))

    %% New Lead
    NL --> EMAIL -->|new lead| SD0_E --> XML
    NL --> API -->|new lead| SD0_A --> CRM

    %% Competing Lead
    CL --> EMAIL -->|competing| NOCL_E --> XML
    CL --> API -->|competing| NOCL_A --> LP

    %% Walk-In Self
    WIS --> EMAIL -->|walk-in self| WKINS_E --> XML
    WIS --> API -->|walk-in self| WKINS_A --> LP

    %% Walk-In Competitor
    WIC --> EMAIL -->|walk-in comp| WKINC_E --> XML
    WIC --> API -->|walk-in comp| WKINC_A --> LP

    %% All links point to Dealer Web
    XML -.-> DW
    CRM -.-> DW
    LP -.-> DW

    style NL fill:#4A90D9,color:#fff
    style CL fill:#E6A23C,color:#fff
    style WIS fill:#67C23A,color:#fff
    style WIC fill:#F56C6C,color:#fff
    style DW fill:#9754E6,color:#fff,stroke:#9754E6,stroke-width:3px
    style XML fill:#606266,color:#fff
    style CRM fill:#606266,color:#fff
    style LP fill:#606266,color:#fff
    style SD0_E fill:#E8F0FE,color:#333
    style SD0_A fill:#E8F0FE,color:#333
    style NOCL_E fill:#FDF0E0,color:#333
    style NOCL_A fill:#FDF0E0,color:#333
    style WKINS_E fill:#E8F8E0,color:#333
    style WKINS_A fill:#E8F8E0,color:#333
    style WKINC_E fill:#FDE8E8,color:#333
    style WKINC_A fill:#FDE8E8,color:#333
  ```
## UTM Parameter Combinations by Scenario

All traction report URLs follow this structure:

```
{dealer_web_base_url}/dealers/{dealer_id}/leads/{lead_id}?utm_content={CODE}&utm_medium={MEDIUM}[&utm_source={SOURCE}]
```

### Scenarios

| Scenario | Alert Type | Delivery Channel | `utm_content` | `utm_medium` | `utm_source` | Builder Module | Destination(s) |
|---|---|---|---|---|---|---|---|
| New lead delivery | — | Email (XML) | `SD0` | `email` | — | `LeadDeliveryView.build_traction_profile_url/2`, `Formatter.get_traction_report_url/3` (`:email`) | Dealer XML Email |
| New lead delivery | — | API (CRM) | `SD0` | `api` | `api` | `Formatter.get_traction_report_url/3` (`:api`) | Major Account CRMs (direct API) |
| Competing lead alert (single competitor) | `:competing_lead` | Email (XML) | `NOCL` | `email` | — | `Formatter.build_lead_reengagement_traction_report_section/3` | Dealer XML Email |
| Competing lead alert (multiple competitors) | `:competing_lead` | Email (XML) | `NOCL` | `email` | — | `Formatter.build_lead_reengagement_traction_report_section/3` | Dealer XML Email |
| Competing lead alert | `:competing_lead` | API | `NOCL` | `api` | — | `AlertPayloadBuilder.build_traction_report_url/3` | Leads Platform API |
| Walk-in self alert | `:walk_in_self` | Email (XML) | `WKINS` | `email` | — | `Formatter.build_lead_reengagement_traction_report_section/3` | Dealer XML Email |
| Walk-in self alert | `:walk_in_self` | API | `WKINS` | `api` | — | `AlertPayloadBuilder.build_traction_report_url/3` | Leads Platform API |
| Walk-in competitor alert | `:walk_in_competitor` | Email (XML) | `WKINC` | `email` | — | `Formatter.build_lead_reengagement_traction_report_section/3` | Dealer XML Email |
| Walk-in competitor alert | `:walk_in_competitor` | API | `WKINC` | `api` | — | `AlertPayloadBuilder.build_traction_report_url/3` | Leads Platform API |

### `utm_content` Code Reference

| Code | Meaning |
|---|---|
| `SD0` | **S**hopper **D**etails — new lead delivery |
| `NOCL` | **N**ew **O**ther **C**ompeting **L**ead — competing lead reengagement alert |
| `WKINS` | **W**al**K**-**IN** **S**elf — shopper detected on dealer's own lot |
| `WKINC` | **W**al**K**-**IN** **C**ompetitor — shopper detected on a competitor's lot |

### Notes

- **`utm_source=api`** is only appended for new lead delivery via API (the `SD0` + API path). All other scenarios omit `utm_source`.
- **`utm_medium`** consistently reflects the delivery channel: `"email"` for email delivery, `"api"` for API delivery across all alert types.

## Lead & Alert Delivery Targets

### Email Targets

| Target | Description | How Resolved |
|---|---|---|
| Dealer XML Email | XML-formatted email sent to dealer's configured email addresses | Resolved from `Dealer.emails` filtered by lead type (`new_lead`, `used_lead`, `phone_lead`, `default_lead`) and `format == "xml"` via `xml_emails_for_leads_dealer/1`. Used in loud mode when `major_account_api` is not set. |

### API Targets

| Target | Description | How Resolved |
|---|---|---|
| Major Account CRMs (Carmax, AutoNation, Avis) | Direct API integrations for new lead delivery. Each has its own API client module. | Resolved from `Dealer.major_account_api`. Delivery handled directly by `Carmax.submit_lead/3`, `Autonation.submit_lead/3`, `Avis.submit_lead/3`. |
| Leads Platform API | Cars.com internal API that acts as a router to downstream CRMs. The `method` field on `LeadReengagementDestination` tells the Leads Platform which CRM to forward the alert to (e.g. `vin_solutions`). | Resolved from `Dealer.core_leads_target_crm`. Payload built by `AlertPayloadBuilder`. Delivery handled by `LeadsPlatform.submit_alert/1`. |

### Delivery Mode Resolution

The delivery channel for reengagement alerts is determined by `DealerShopperAlertConfiguration`:

| Delivery Mode | Behavior |
|---|---|
| `"loud"` (or `nil`) | Checks `Dealer.major_account_api` first — if set, delivers via that CRM's direct API. Otherwise falls back to dealer XML emails. |
| `"quiet"` | Delivers via Leads Platform API if `Dealer.core_leads_target_crm` is `"vin_solutions"`. Otherwise no delivery occurs. |

### Which Targets Receive Which Alert Types

| Target | New Lead | Competing Lead | Walk-In Self | Walk-In Competitor |
|---|---|---|---|---|
| Dealer XML Email | Yes | Yes (loud, no `major_account_api`) | Yes (loud, no `major_account_api`) | Yes (loud, no `major_account_api`) |
| Major Account CRMs (direct API) | Yes (if `major_account_api`) | Yes (loud, if `major_account_api`) | Yes (loud, if `major_account_api`) | Yes (loud, if `major_account_api`) |
| Leads Platform API | — | Yes (quiet, if `core_leads_target_crm`) | Yes (quiet, if `core_leads_target_crm`) | Yes (quiet, if `core_leads_target_crm`) |

### Delivery Worker Routing

The `LeadReengagementDestinationDelivery` worker in Manifold routes based on the `method` field stored on the `LeadReengagementDestination`:

| `method` value | Handler | Actual destination |
|---|---|---|
| `"email"` | `send_email/1` | Dealer XML Email (via Bamboo/Mailer) |
| `"autonation"` | `send_autonation_api/1` | AutoNation direct API |
| `"avis"` | `send_avis_api/1` | Avis direct API |
| `"carmax"` | `send_carmax_api/1` | Carmax direct API |
| `"vin_solutions"` | `send_core_leads_platform_api/1` | Leads Platform API (forwards to VinSolutions) |