# Power BI Dashboard Design and Implementation Guide

## 1. Data Model Architecture for Power BI

### 1.1 Data Source Configuration

**Primary Source**: Microsoft Dataverse via native connector
- Direct query mode for operational dashboards (real-time data)
- Import mode with scheduled refresh (hourly) for analytics dashboards
- Recommended refresh: Every 4 hours for Asset Portfolio, Daily for Compliance

**Secondary Source**: Azure SQL Database (optional)
- Historical data warehouse for archival records
- Decommissioned assets and old audit logs
- Aggregate tables for performance optimization

**Connection Configuration**:
```
Dataverse Connection:
- Environment URL: [Your Dynamics environment]
- Tables: Asset, AssetAssignment, Warranty, MaintenanceSchedule, 
          SoftwareLicense, Invoice, PurchaseOrder, Vendor, AuditLog
- Query Folding: Enable for optimal performance
- Incremental Refresh: Enable on large tables (>100k rows)

Azure SQL Connection (optional):
- Server: [Azure SQL server endpoint]
- Database: [Analytics DB]
- Authentication: Service Principal (recommended over user credentials)
```

---

## 2. Asset Portfolio Dashboard

**Purpose**: Executive overview of asset inventory, valuation, and lifecycle

### 2.1 Data Model

**Tables to Import**:
- Asset (all columns)
- AssetAssignment (AssetID, AssignedToUser, AssignedDate, IsActive)
- Department (for context if separate table)

**Calculated Columns** (DAX):

```dax
-- Asset Age (years)
Asset Age = DATEDIFF(Asset[PurchaseDate], TODAY(), YEAR)

-- Depreciation Value (Linear depreciation, 5-year useful life)
Depreciated Value = 
    VAR AssetAge = DATEDIFF(Asset[PurchaseDate], TODAY(), YEAR)
    VAR DepreciationRate = 0.2 -- 20% per year for 5-year assets
    VAR DepreciatedVal = Asset[OriginalCost] * (1 - DepreciationRate * AssetAge)
    RETURN IF(DepreciatedVal < 0, 0, DepreciatedVal)

-- Utilization Status
Utilization Status = 
    IF(
        COUNTROWS(
            FILTER(
                AssetAssignment,
                AssetAssignment[AssetID] = Asset[AssetID] &&
                AssetAssignment[IsActive] = TRUE()
            )
        ) > 0,
        "Assigned",
        "Unassigned"
    )

-- Days Since Purchase
Days Since Purchase = DATEDIFF(Asset[PurchaseDate], TODAY(), DAY)

-- Asset Status Color (for conditional formatting)
Status Color = 
    SWITCH(
        Asset[Status],
        "Active", "#70AD47",
        "Inactive", "#A6A6A6",
        "Decommissioned", "#C41E3A",
        "In Repair", "#FFC000",
        "#808080"
    )
```

**Measures** (DAX):

```dax
-- Total Assets
Total Assets = COUNTA(Asset[AssetID])

-- Active Assets
Active Assets = 
    CALCULATE(
        COUNTA(Asset[AssetID]),
        FILTER(Asset, Asset[Status] = "Active")
    )

-- Total Asset Value
Total Asset Value = SUM(Asset[CurrentValue])

-- Average Asset Value
Average Asset Value = AVERAGE(Asset[CurrentValue])

-- Total Original Cost (for ROI calculations)
Total Original Cost = SUM(Asset[OriginalCost])

-- Depreciation This Year
Depreciation This Year = 
    SUMX(
        FILTER(
            Asset,
            YEAR(Asset[PurchaseDate]) = YEAR(TODAY())
        ),
        Asset[OriginalCost] * 0.2  -- 20% depreciation in purchase year
    )

-- Asset Utilization Rate (%)
Utilization Rate = 
    DIVIDE(
        CALCULATE(
            COUNTA(AssetAssignment[AssetID]),
            FILTER(AssetAssignment, AssetAssignment[IsActive] = TRUE())
        ),
        COUNTA(Asset[AssetID]),
        0
    )

-- Assets Needing Replacement (Age > 5 years)
Assets Needing Replacement = 
    CALCULATE(
        COUNTA(Asset[AssetID]),
        FILTER(
            Asset,
            DATEDIFF(Asset[PurchaseDate], TODAY(), YEAR) > 5
        )
    )

-- Estimated Replacement Cost
Estimated Replacement Cost = 
    SUMX(
        FILTER(
            Asset,
            DATEDIFF(Asset[PurchaseDate], TODAY(), YEAR) > 5
        ),
        Asset[OriginalCost]  -- Assume full replacement at original cost
    )

-- YoY Asset Value Change (%)
YoY Value Change = 
    VAR CurrentValue = [Total Asset Value]
    VAR PriorYearValue = 
        CALCULATE(
            SUM(Asset[CurrentValue]),
            DATEADD(Asset[PurchaseDate], -1, YEAR)
        )
    RETURN DIVIDE(CurrentValue - PriorYearValue, PriorYearValue, 0)
```

### 2.2 Key Visualizations

**Page 1: Asset Summary**

1. **KPI Cards** (Top section, 3 columns):
   - Total Assets: Display total count with sparkline trend
   - Total Asset Value: Display sum with % change vs. last month
   - Utilization Rate: Display percentage with conditional coloring

2. **Asset Count by Type** (Clustered Bar Chart):
   - Y-axis: AssetType
   - X-axis: Count of AssetID
   - Sort: Descending by count
   - Color by: AssetType
   - Tooltip: Include count and percentage of total

3. **Asset Value Distribution by Department** (Treemap):
   - Values: Sum of CurrentValue
   - Category: Department
   - Color: Asset Status
   - Tooltip: Show count + percentage

4. **Asset Age Distribution** (Histogram):
   - Bins: Asset Age (0-1yr, 1-2yr, 2-3yr, 3-5yr, 5+yr)
   - Y-axis: Count
   - Color: Asset Age category (green for <3yr, yellow for 3-5yr, red for 5+yr)
   - Highlight assets needing replacement (5+ years)

5. **Depreciation Trend** (Line & Column Combo Chart):
   - X-axis: Month
   - Primary Y-axis: Total Asset Value (Line)
   - Secondary Y-axis: Monthly Depreciation (Column)
   - Color: Line blue, Column red
   - Forecast: Optional - Add 12-month depreciation forecast

6. **Asset Status Distribution** (Donut Chart):
   - Values: Count of AssetID
   - Legend: Status
   - Tooltips: Show percentage and count

**Page 2: Asset Inventory Details**

1. **Interactive Asset Grid** (Table Visual):
   - Columns: AssetID, AssetName, AssetType, SerialNumber, Owner, Department, Status, PurchaseDate, CurrentValue, AssetAge
   - Sorting: By CurrentValue (descending)
   - Conditional formatting:
     - Status column: Green (Active), Gray (Inactive), Red (Decommissioned), Yellow (In Repair)
     - Asset Age: Red if > 5 years
   - Column widths: Auto-adjusted
   - Grouping: Optional by Department or AssetType
   - Search/Filter: On AssetName, Owner, Department

2. **Cost per Department** (Clustered Column Chart):
   - X-axis: Department
   - Y-axis: Sum of CurrentValue
   - Legend: AssetType
   - Sorting: By total cost descending
   - Data labels: Show values
   - Drill-through: Click to view detailed inventory for department

3. **Utilization by Department** (100% Stacked Bar):
   - X-axis: Department
   - Y-axis: Utilization Rate (%)
   - Stack: Assigned vs Unassigned assets
   - Color: Green (Assigned), Gray (Unassigned)

4. **Slicers** (Top of page):
   - Department (multi-select, buttons style)
   - Asset Type (multi-select, dropdown)
   - Status (multi-select, buttons)
   - Date Range (slider): PurchaseDate
   - Owner (dropdown with search)

---

## 3. Warranty and Maintenance Dashboard

**Purpose**: Monitor warranty coverage, expiration dates, and maintenance schedules

### 3.1 Calculated Columns and Measures

```dax
-- Warranty Status
Warranty Status = 
    VAR DaysUntilExpiry = DATEDIFF(TODAY(), Warranty[EndDate], DAY)
    RETURN SWITCH(
        TRUE(),
        DaysUntilExpiry < 0, "Expired",
        DaysUntilExpiry < 30, "Critical (< 30 days)",
        DaysUntilExpiry < 60, "Warning (30-60 days)",
        DaysUntilExpiry < 90, "Monitor (60-90 days)",
        "Active"
    )

-- Days Until Warranty Expiry
Days Until Warranty Expiry = DATEDIFF(TODAY(), Warranty[EndDate], DAY)

-- Warranty Coverage %
Warranty Coverage % = 
    DIVIDE(
        CALCULATE(
            COUNTA(Warranty[WarrantyID]),
            FILTER(Warranty, Warranty[EndDate] >= TODAY())
        ),
        COUNTA(Asset[AssetID]),
        0
    )

-- Active Maintenance Count
Active Maintenance = 
    CALCULATE(
        COUNTA(MaintenanceSchedule[ScheduleID]),
        FILTER(
            MaintenanceSchedule,
            MaintenanceSchedule[Status] IN {"Planned", "In Progress"}
        )
    )

-- Overdue Maintenance Count
Overdue Maintenance = 
    CALCULATE(
        COUNTA(MaintenanceSchedule[ScheduleID]),
        FILTER(
            MaintenanceSchedule,
            MaintenanceSchedule[Status] = "Planned" &&
            MaintenanceSchedule[ScheduledDate] < TODAY()
        )
    )

-- Average Maintenance Cost
Average Maintenance Cost = 
    CALCULATE(
        AVERAGE(MaintenanceSchedule[ActualCost]),
        FILTER(
            MaintenanceSchedule,
            MaintenanceSchedule[Status] = "Completed"
        )
    )

-- MTTR (Mean Time To Repair) - in hours
MTTR Hours = 
    VAR CompletedMaintenance = 
        FILTER(
            MaintenanceSchedule,
            MaintenanceSchedule[Status] = "Completed"
        )
    RETURN AVERAGEX(
        CompletedMaintenance,
        INT((MaintenanceSchedule[CompletedDate] - MaintenanceSchedule[ScheduledDate]) * 24)
    )

-- Maintenance Cost by Type YTD
Maintenance Cost YTD = 
    CALCULATE(
        SUM(MaintenanceSchedule[ActualCost]),
        FILTER(
            MaintenanceSchedule,
            YEAR(MaintenanceSchedule[CompletedDate]) = YEAR(TODAY()) &&
            MaintenanceSchedule[Status] = "Completed"
        )
    )
```

### 3.2 Key Visualizations

**Page 1: Warranty Status**

1. **Warranty Expiration Timeline** (Clustered Column Chart):
   - X-axis: Month of expiration (next 12 months)
   - Y-axis: Count of warranties expiring
   - Color: Red (< 30 days), Yellow (30-60 days), Blue (61+ days)
   - Data labels: Show count
   - Annotations: Mark today's date with vertical line

2. **Warranty Status Gauge**:
   - Value: Warranty Coverage %
   - Min: 0%, Target: 95%, Max: 100%
   - Color: Red < 80%, Yellow 80-95%, Green > 95%

3. **Warranty by Type** (Donut Chart):
   - Values: Count of WarrantyID
   - Legend: WarrantyType
   - Tooltip: Show count and coverage duration

4. **Critical Expiry Alerts** (Alert Card):
   - Display count of warranties expiring within 30 days (RED)
   - KPI: Show breakdown: Critical (0-30d), Warning (30-60d), Monitor (60-90d)

5. **Warranty Expiration Calendar** (Table with conditional formatting):
   - Columns: Asset Name, Warranty Type, Expiration Date, Days Remaining, Warranty Provider
   - Sorting: By expiration date (ascending)
   - Conditional formatting: Cell background color based on Days Remaining (Red if <30)
   - Action buttons: "Renew Warranty" link

**Page 2: Maintenance Dashboard**

1. **Maintenance Status Kanban** (Card Grid or Alternative Matrix Visual):
   - Status: Planned | In Progress | Completed | Overdue
   - Cards showing count and % of total
   - Color-code: Overdue in red, In Progress in orange, Completed in green

2. **Maintenance Cost Trend** (Line Chart):
   - X-axis: Month (past 12 months)
   - Y-axis: Actual Cost
   - Line: Monthly trend with forecast (optional)
   - Fill area under line
   - Add reference line for average

3. **Maintenance by Provider** (Clustered Bar):
   - Y-axis: Vendor name
   - X-axis: Total Cost and Count (dual axis)
   - Sort: By cost descending
   - Tooltip: Show cost, count, average cost per job

4. **MTTR by Asset Type** (Horizontal Bar):
   - Y-axis: Asset Type
   - X-axis: Average MTTR (hours)
   - Color: Orange for MTTR > 24 hours, Green for <24 hours

5. **Overdue Maintenance Alert** (Card + Table):
   - Card: Count of overdue maintenance tasks (RED if >0)
   - Table: List overdue tasks with:
     - Asset name
     - Scheduled date
     - Days overdue
     - Maintenance type
     - Assigned provider

---

## 4. Compliance and Risk Dashboard

**Purpose**: Monitor compliance with policies, identify risks and audit gaps

### 4.1 Calculated Columns and Measures

```dax
-- Compliance Status
Compliance Status = 
    IF(
        COUNTROWS(
            FILTER(
                AuditLog,
                AuditLog[EntityID] = Asset[AssetID]
            )
        ) > 0,
        "Compliant",
        "Not Verified"
    )

-- Policy Adherence Score (0-100)
Policy Adherence Score = 
    VAR TotalAssets = COUNTA(Asset[AssetID])
    VAR CompliantAssets = 
        CALCULATE(
            COUNTA(Asset[AssetID]),
            FILTER(Asset, Asset[Compliance Status] = "Compliant")
        )
    RETURN DIVIDE(CompliantAssets, TotalAssets, 0) * 100

-- Missing Documentation Count
Missing Documentation = 
    SUMX(
        Asset,
        IF(
            ISBLANK(RELATED(Warranty[WarrantyID])),
            1,
            0
        )
    ) +
    SUMX(
        Asset,
        IF(
            ISBLANK(RELATED(MaintenanceSchedule[ScheduleID])),
            1,
            0
        )
    )

-- High-Risk Assets (no warranty, overdue maintenance)
High Risk Asset Count = 
    CALCULATE(
        COUNTA(Asset[AssetID]),
        FILTER(
            Asset,
            ISBLANK(RELATED(Warranty[WarrantyID])) ||
            COUNTA(
                FILTER(
                    MaintenanceSchedule,
                    MaintenanceSchedule[AssetID] = Asset[AssetID] &&
                    MaintenanceSchedule[Status] = "Overdue"
                )
            ) > 0
        )
    )

-- Assets in Security Policy Breach
Security Breach Count = 
    CALCULATE(
        COUNTA(Asset[AssetID]),
        FILTER(
            Asset,
            DATEDIFF(Asset[PurchaseDate], TODAY(), YEAR) > 3 &&
            Asset[Status] = "Active"
        )
    )

-- Audit Issues Outstanding
Audit Issues Open = 
    CALCULATE(
        COUNTA(AuditLog[AuditID]),
        FILTER(AuditLog, AuditLog[Status] = "Open")
    )

-- Audit Resolution Time (days)
Avg Issue Resolution Time = 
    AVERAGEX(
        FILTER(
            AuditLog,
            AuditLog[Status] = "Resolved"
        ),
        DATEDIFF(
            AuditLog[CreatedDate],
            AuditLog[ResolvedDate],
            DAY
        )
    )

-- Compliance Score by Policy (0-100 per policy)
Compliance Score by Policy = 
    VAR RelatedAssets = 
        CALCULATE(
            COUNTA(Asset[AssetID]),
            FILTER(
                Asset,
                Asset[AssetType] IN VALUES(CompliancePolicy[RelatedAssetTypes])
            )
        )
    VAR CompliantAssets = 
        CALCULATE(
            COUNTA(Asset[AssetID]),
            FILTER(
                Asset,
                Asset[Compliance Status] = "Compliant" &&
                Asset[AssetType] IN VALUES(CompliancePolicy[RelatedAssetTypes])
            )
        )
    RETURN DIVIDE(CompliantAssets, RelatedAssets, 0) * 100
```

### 4.2 Key Visualizations

**Page 1: Compliance Overview**

1. **Overall Compliance Score Gauge** (Gauge Visual):
   - Value: Policy Adherence Score
   - Min: 0%, Target: 100%, Max: 100%
   - Color: Red < 70%, Yellow 70-90%, Green > 90%
   - Large prominent display for executive view

2. **Compliance Score by Policy** (Horizontal Bar Chart):
   - Y-axis: Policy name
   - X-axis: Compliance score (0-100%)
   - Color: Green > 90%, Yellow 70-90%, Red < 70%
   - Hover tooltip: Show compliant count vs. required

3. **Risk Heat Map** (Matrix Visual):
   - Rows: Department
   - Columns: Risk Category (Security, Warranty, Maintenance, Audit)
   - Values: Count of assets at risk (color intensity increases with count)
   - Interactive: Click to drill through to asset list

4. **Non-Compliant Assets by Reason** (Clustered Bar):
   - X-axis: Count
   - Y-axis: Non-compliance reason (Missing warranty, Overdue maintenance, etc.)
   - Color: Different color per reason
   - Data labels: Show count

**Page 2: Audit and Issues**

1. **Audit Issue Status Funnel**:
   - Stages: Open → In Progress → Resolved
   - Width represents count at each stage
   - Color: Red (Open), Orange (In Progress), Green (Resolved)
   - Show time in stage as tooltip

2. **Issues by Category** (Pie Chart):
   - Values: Count
   - Legend: Issue category (Documentation, Compliance, Security, etc.)
   - Tooltip: Count and percentage

3. **Issue Resolution Timeline** (Line Chart):
   - X-axis: Issue created date (past 6 months)
   - Y-axis: Average resolution time (days)
   - Trend: Show if resolution time improving or degrading

4. **Outstanding Issues Table**:
   - Columns: Issue ID, Description, Category, Created Date, Days Outstanding, Assigned To, Status
   - Sorting: By Days Outstanding (descending) - oldest first
   - Conditional formatting: Red if > 30 days outstanding
   - Action column: Quick link to issue detail

---

## 5. Software License Optimization Dashboard

**Purpose**: Monitor license utilization, identify cost optimization opportunities

### 5.1 Calculated Columns and Measures

```dax
-- License Utilization %
License Utilization % = 
    DIVIDE(
        SoftwareLicense[UsedCount],
        SoftwareLicense[LicenseCount],
        0
    ) * 100

-- Licenses Over-Provisioned (Used < 20% of provisioned)
Over Provisioned = 
    CALCULATE(
        COUNTA(SoftwareLicense[LicenseID]),
        FILTER(
            SoftwareLicense,
            DIVIDE(
                SoftwareLicense[UsedCount],
                SoftwareLicense[LicenseCount],
                0
            ) < 0.2
        )
    )

-- Licenses Under-Provisioned (Used > 90% of provisioned)
Under Provisioned = 
    CALCULATE(
        COUNTA(SoftwareLicense[LicenseID]),
        FILTER(
            SoftwareLicense,
            DIVIDE(
                SoftwareLicense[UsedCount],
                SoftwareLicense[LicenseCount],
                0
            ) > 0.9
        )
    )

-- Potential Cost Savings (over-provisioned licenses)
Potential Cost Savings = 
    SUMX(
        FILTER(
            SoftwareLicense,
            DIVIDE(
                SoftwareLicense[UsedCount],
                SoftwareLicense[LicenseCount],
                0
            ) < 0.2
        ),
        (SoftwareLicense[LicenseCount] - SoftwareLicense[UsedCount]) * 
        DIVIDE(SoftwareLicense[MonthlyRecurringCost], SoftwareLicense[LicenseCount])
    ) * 12  -- Annual savings

-- Monthly Recurring Cost (MRC) Total
Total MRC = SUM(SoftwareLicense[MonthlyRecurringCost])

-- Annual License Cost
Annual License Cost = [Total MRC] * 12

-- Expiring Licenses (within 90 days)
Licenses Expiring Soon = 
    CALCULATE(
        COUNTA(SoftwareLicense[LicenseID]),
        FILTER(
            SoftwareLicense,
            SoftwareLicense[ExpiryDate] >= TODAY() &&
            SoftwareLicense[ExpiryDate] <= TODAY() + 90
        )
    )

-- License by Vendor Cost (for contract negotiations)
Cost by Vendor = 
    SUMX(
        VALUES(SoftwareLicense[VendorName]),
        CALCULATE([Total MRC])
    )
```

### 5.2 Key Visualizations

**Page 1: License Portfolio**

1. **License Utilization Gauge** (Gauge Chart):
   - Value: Average License Utilization %
   - Target: 80%
   - Color: Red < 60%, Yellow 60-80%, Green > 80%

2. **License Status Summary** (KPI Cards - 4 columns):
   - Total Licenses: Count
   - Total MRC: Annual cost
   - Licenses Expiring Soon: Count (RED if > 5)
   - Optimization Opportunity: Potential savings

3. **License Utilization by Product** (Horizontal Bar Sorted):
   - Y-axis: License product name
   - X-axis: Utilization % (0-100%)
   - Color: Green if > 80%, Yellow if 20-80%, Red if < 20%
   - Data labels: Show percentage and used/total count
   - Trend: Add small sparkline showing last 3 months trend

4. **License Cost Breakdown** (Treemap):
   - Values: Annual Cost (MRC * 12)
   - Groups: Vendor
   - Color: License type
   - Size: Cost
   - Tooltip: Show MRC, renewal date, utilization %

**Page 2: Optimization Analysis**

1. **Over vs Under-Provisioned** (Dumbbell/Scatter Chart alternative):
   - X-axis: License product
   - Y-axis: Utilization %
   - Points: Each license showing current usage
   - Reference lines: 20% (over-provisioned), 90% (under-provisioned)
   - Color: Red if over-provisioned, Yellow if under-provisioned, Green if optimal
   - Show bubble size = Annual cost

2. **Renewal Calendar** (Table with conditional formatting):
   - Columns: Product, License Type, Current Count, Used Count, Utilization %, Renewal Date, MRC
   - Sorting: By renewal date (ascending)
   - Conditional formatting:
     - Renewal date within 30 days: RED background
     - Renewal date 30-60 days: YELLOW background
     - Utilization: Red if < 20%, Green if > 80%

3. **Cost per User Analysis** (Table):
   - Columns: Product, Total Cost, Assigned Users, Cost per User, Utilization %
   - Sorting: By cost per user (descending)
   - Identify products with high cost per user for optimization

4. **Potential Savings Opportunities** (Table + Card):
   - Card: Total potential annual savings
   - Table showing:
     - Product name
     - Current licenses
     - Used licenses
     - Unused licenses
     - Annual savings if reduced
     - Action button: "Request reduction"

---

## 6. Performance Optimization Recommendations

### 6.1 Query Optimization

**Large Tables (>100k rows)**: Implement incremental refresh
```
Configuration:
- Detect data changes: PurchaseDate (or LastModifiedDate)
- Only refresh current and previous 12 months of data
- Archive older data to Azure SQL Database
- Reduces refresh time and Power BI capacity usage
```

**Aggregation Tables**: Pre-aggregate high-granularity data
```
Create aggregate tables for:
- Asset Cost by Department (monthly)
- Maintenance Cost by Provider (monthly)
- License Cost by Vendor (monthly)

Use these aggregated tables for summary visualizations
```

**Calculated Columns vs. Measures**: Use measures for comparisons
```
Avoid calculated columns for:
- Time-based calculations (use measures instead)
- Functions requiring filter context

Use calculated columns only for:
- Static categorizations
- One-time calculations
```

### 6.2 Refresh Strategy

**Recommended Refresh Schedule**:
- Operational Dashboard: Every 4 hours (during business hours)
- Analytics Dashboard: Daily at 2 AM (off-peak)
- Executive Summary: Weekly snapshot (Sunday 6 AM)
- Archive cleanup: Monthly (1st of month at 3 AM)

**Monitoring**:
- Set up email alerts for refresh failures
- Track refresh duration (alert if > 2x normal)
- Monitor capacity utilization (alert if > 80%)

---

## 7. Sharing and Permissions

**Role-Based Access**:
- IT Director: All dashboards, edit access
- IT Managers: Portfolio, Warranty, Maintenance, Compliance dashboards
- Asset Managers: Portfolio, Warranty, Maintenance dashboards only
- Finance: License Optimization, Invoice dashboards
- Compliance Officer: Compliance dashboard only
- All IT Staff: Read-only access to Portfolio summary

**Implementation**:
- Use Power BI Row-Level Security (RLS) based on Department
- Create roles for each department with department-filtered data
- Use Azure AD security groups for efficient access management

---

## Conclusion

These Power BI dashboards provide comprehensive visibility into IT assets, enabling data-driven decision-making for maintenance, compliance, and cost optimization. The combination of operational and analytical views supports both day-to-day management and strategic planning.
