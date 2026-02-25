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
        NOCL_API["NOCL / email"]
        WKINS_E["WKINS / email"]
        WKINS_A["WKINS / api"]
        WKINC_E["WKINC / email"]
        WKINC_A["WKINC / api"]
    end

    subgraph Dest["4. Destination"]
        XML["Dealer XML Email"]
        CRM["Major Account CRM<br/>(Carmax, AutoNation, Avis)"]
        VS["VinSolutions"]
        LP["Leads Platform API"]
    end

    DW(("Dealer Web<br/>Shopper Details"))

    %% New Lead
    NL --> EMAIL -->|new lead| SD0_E --> XML
    NL --> API -->|new lead| SD0_A --> CRM

    %% Competing Lead
    CL --> EMAIL -->|competing| NOCL_E --> XML
    CL --> API -->|competing| NOCL_API --> LP

    %% Walk-In Self
    WIS --> EMAIL -->|walk-in self| WKINS_E --> XML
    WIS --> API -->|walk-in self| WKINS_A --> LP
    WKINS_A --> VS
    WKINS_A --> CRM

    %% Walk-In Competitor
    WIC --> EMAIL -->|walk-in comp| WKINC_E --> XML
    WIC --> API -->|walk-in comp| WKINC_A --> LP
    WKINC_A --> VS
    WKINC_A --> CRM

    %% All links point to Dealer Web
    XML -.-> DW
    CRM -.-> DW
    VS -.-> DW
    LP -.-> DW

    style NL fill:#4A90D9,color:#fff
    style CL fill:#E6A23C,color:#fff
    style WIS fill:#67C23A,color:#fff
    style WIC fill:#F56C6C,color:#fff
    style DW fill:#9754E6,color:#fff,stroke:#9754E6,stroke-width:3px
    style XML fill:#606266,color:#fff
    style CRM fill:#606266,color:#fff
    style VS fill:#606266,color:#fff
    style LP fill:#606266,color:#fff
    style SD0_E fill:#E8F0FE,color:#333
    style SD0_A fill:#E8F0FE,color:#333
    style NOCL_E fill:#FDF0E0,color:#333
    style NOCL_API fill:#FDF0E0,color:#333
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
| New lead delivery | — | API (CRM) | `SD0` | `api` | `api` | `Formatter.get_traction_report_url/3` (`:api`) | Major Account CRMs (Carmax, AutoNation, Avis) |
| Competing lead alert (single competitor) | `:competing_lead` | Email (XML) | `NOCL` | `email` | — | `Formatter.build_lead_reengagement_traction_report_section/3` | Dealer XML Email |
| Competing lead alert (multiple competitors) | `:competing_lead` | Email (XML) | `NOCL` | `email` | — | `Formatter.build_lead_reengagement_traction_report_section/3` | Dealer XML Email |
| Competing lead alert | `:competing_lead` | API (Leads Platform) | `NOCL` | `email` | — | `AlertPayloadBuilder.build_traction_report_url/3` | Leads Platform API |
| Walk-in self alert | `:walk_in_self` | Email (XML) | `WKINS` | `email` | — | `Formatter.build_lead_reengagement_traction_report_section/3` | Dealer XML Email |
| Walk-in self alert | `:walk_in_self` | API | `WKINS` | `api` | — | `AlertPayloadBuilder.build_traction_report_url/3` | Leads Platform API, VinSolutions, Major Account CRMs |
| Walk-in competitor alert | `:walk_in_competitor` | Email (XML) | `WKINC` | `email` | — | `Formatter.build_lead_reengagement_traction_report_section/3` | Dealer XML Email |
| Walk-in competitor alert | `:walk_in_competitor` | API | `WKINC` | `api` | — | `AlertPayloadBuilder.build_traction_report_url/3` | Leads Platform API, VinSolutions, Major Account CRMs |

### `utm_content` Code Reference

| Code | Meaning |
|---|---|
| `SD0` | **S**hopper **D**etails — new lead delivery |
| `NOCL` | **N**ew **O**ther **C**ompeting **L**ead — competing lead reengagement alert |
| `WKINS` | **W**al**K**-**IN** **S**elf — shopper detected on dealer's own lot |
| `WKINC` | **W**al**K**-**IN** **C**ompetitor — shopper detected on a competitor's lot |

### Delivery Mode Resolution

The delivery channel is determined by `DealerShopperAlertConfiguration` and dealer attributes:

| Delivery Mode | Behavior |
|---|---|
| `"loud"` (or `nil`) | Checks `Dealer.major_account_api` first — if set, delivers via that CRM API. Otherwise falls back to dealer XML emails. |
| `"quiet"` | Only delivers via API if `Dealer.core_leads_target_crm` is `"vin_solutions"`. Otherwise no delivery occurs. |

### Which Targets Receive Which Alert Types

| Target | New Lead | Competing Lead | Walk-In Self | Walk-In Competitor |
|---|---|---|---|---|
| Dealer XML Email | Yes | Yes (loud, no `major_account_api`) | Yes (loud, no `major_account_api`) | Yes (loud, no `major_account_api`) |
| Major Account CRMs (Carmax, AutoNation, Avis) | Yes (if `major_account_api`) | Yes (loud, if `major_account_api`) | Yes (loud, if `major_account_api`) | Yes (loud, if `major_account_api`) |
| VinSolutions | Yes (if `core_leads_target_crm`)

## Delivery Targets

### Email Targets

| Target | Description | How Resolved |
|---|---|---|
| Dealer XML Email | XML-formatted email sent to dealer's configured email addresses | Resolved from `Dealer.emails` filtered by lead type (`new_lead`, `used_lead`, `phone_lead`, `default_lead`) and `format == "xml"` via `xml_emails_for_leads_dealer/1`. Used in loud mode when `major_account_api` is not set. |

### API (CRM) Targets

| Target | Identifier | Description |
|---|---|---|
| VinSolutions | `vin_solutions` | CRM integration; used in **quiet** mode only (resolved from `Dealer.core_leads_target_crm`). |
| Carmax | `carmax` | Major account API integration. Used in **loud** mode (resolved from `Dealer.major_account_api`). |
| AutoNation | `autonation` | Major account API integration. Used in **loud** mode (resolved from `Dealer.major_account_api`). |
| Avis | `avis` | Major account ADF-style API integration. Used in **loud** mode (resolved from `Dealer.major_account_api`). |
| Leads Platform | `leads_platform` | Cars.com internal leads platform API. Used for reengagement alert delivery via `AlertPayloadBuilder`. |

### Delivery Mode Resolution

| Delivery Mode | Behavior |
|---|---|
| `"loud"` (or `nil`) | Checks `Dealer.major_account_api` first — if set, delivers via that CRM API. Otherwise falls back to dealer XML emails. |
| `"quiet"` | Only delivers via API if `Dealer.core_leads_target_crm` is `"vin_solutions"`. Otherwise no delivery occurs. |

### Which Targets Receive Which Alert Types

| Target | New Lead | Competing Lead | Walk-In Self | Walk-In Competitor |
|---|---|---|---|---|
| Dealer XML Email | Yes | Yes (loud, no `major_account_api`) | Yes (loud, no `major_account_api`) | Yes (loud, no `major_account_api`) |
| Major Account CRMs (Carmax, AutoNation, Avis) | Yes (if `major_account_api`) | Yes (loud, if `major_account_api`) | Yes (loud, if `major_account_api`) | Yes (loud, if `major_account_api`) |
| VinSolutions | — | — | Yes (quiet mode) | Yes (quiet mode) |
| Leads Platform API | — | Yes (if `core_leads_target_crm`) | Yes (if `core_leads_target_crm`) | Yes (if `core_leads_target_crm`) |