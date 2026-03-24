# XpressMobile Bonus System - Executive Summary

## Overview

The XpressMobile bonus system provides drivers with visibility into their performance-based compensation through multiple web interfaces. The system integrates data from SQL Server (PSA database), caches it in Orleans grains, displays it to drivers via web views, and replicates bonus data to DB2 (QTOPS) for payroll processing by legacy RPG systems.

## Business Purpose

Drivers can view and manage three types of bonus information:
- **Performance Bonus** - Safety and fuel efficiency bonuses based on monthly performance metrics
- **Vacation & Bonus (TeamMAX)** - Accrued vacation days and bonus miles eligibility
- **Fast Track Rewards** - Monthly choice between bonus pay or paid vacation

## System Architecture

```mermaid
graph TB
    subgraph "Data Sources"
        PSA[(SQL Server PSA)]
        DB2[(DB2 QTOPS)]
    end
    
    subgraph "XpressMobile Backend"
        Repos[Repositories]
        Grains[Orleans Grains]
        Cache[Grain State Cache]
    end
    
    subgraph "Web Layer"
        Controllers[MVC Controllers]
        Views[Razor Views]
    end
    
    subgraph "Driver Interface"
        Mobile[Mobile App Browser]
    end
    
    subgraph "Payroll Integration"
        Replication[Bonus Replication Grain]
        RPG[RPG Payroll System]
    end
    
    PSA -->|Read Bonus Data| Repos
    Repos -->|Populate| Grains
    Grains -->|Store| Cache
    Grains -->|Serve Data| Controllers
    Controllers -->|Render| Views
    Views -->|Display| Mobile
    
    Replication -->|Nightly Sync| PSA
    Replication -->|Write to DB2| DB2
    DB2 -->|Consumed by| RPG
    
    style PSA fill:#e1f5ff
    style DB2 fill:#e1f5ff
    style Grains fill:#fff4e1
    style Replication fill:#ffe1e1
```

## Key Components

### 1. Driver-Facing Views

| View | Purpose | URL Pattern |
|------|---------|-------------|
| **Bonus/Index** | TeamMAX vacation & bonus summary | `/Bonus/Index` |
| **FastTrackReward/Index** | Monthly reward selection (bonus vs vacation) | `/FastTrackReward/Index` |
| **Payroll/PerformanceBonus** | Detailed performance bonus breakdown with charts | `/Payroll/PerformanceBonus/{company}/{driverId}` |

### 2. Data Flow

1. **Source Data**: Bonus calculations performed in SQL Server (PSA database)
2. **Caching**: Orleans grains cache bonus data for fast retrieval
3. **Display**: MVC controllers fetch from grains and render views
4. **Replication**: Nightly job syncs bonus data to DB2 for payroll processing

### 3. Payroll Integration

The `DriverBonusReplicationGrain` runs nightly to:
- Read safety and fuel bonus data from SQL Server
- Truncate and repopulate DB2 tables (XPSFILE.PRDPBNT and XPSFILE.PRDPBFT)
- Enable RPG payroll systems to process driver bonuses

## Data Tables

### SQL Server (PSA)
- `PSA.dbo.DriverSafetyScoreBonus` - Safety performance bonus data
- `PSA.dbo.DriverSafetyMPGScoreBonus` - Fuel efficiency bonus data

### DB2 (QTOPS)
- `XPSFILE.PRDPBNT` - Safety bonus data for payroll
- `XPSFILE.PRDPBFT` - Fuel bonus data for payroll

## Key Features

### Performance Bonus
- **Safety Score** - Based on event points, driving hours, safety incidents
- **Paid Miles Level 1 & 2** - Tiered mileage-based bonuses
- **Historical View** - 3-month to 3-year performance history
- **Visual Charts** - Donut charts and line graphs for trend analysis

### TeamMAX Bonus
- **Accrued Miles** - Track progress toward bonus/vacation eligibility
- **Target Miles** - Goal thresholds for earning rewards
- **Increments** - Number of bonus/vacation increments earned

### Fast Track Rewards
- **Monthly Selection** - Drivers choose between bonus pay or vacation
- **Lock-in Date** - Selection deadline for upcoming month
- **Driver Stats Integration** - Tracks driver reward preferences

## Technical Stack

- **Backend**: ASP.NET Core MVC, Orleans (distributed actor framework)
- **Data Access**: ADO.NET with SQL Server and DB2 connections
- **Frontend**: Razor views, Bootstrap, Chart.js
- **Authentication**: Token-based authentication via WebToken
- **Caching**: Orleans grain persistence with state management

## Security & Access Control

- All bonus views require authentication via `[AuthenticateRequest(AuthorizationTokenType.WebToken)]`
- Driver-specific data filtered by authenticated driver identity
- No cross-driver data access permitted

## Monitoring & Logging

- Timed operations logged for performance tracking
- Correlation IDs for request tracing
- Error logging with structured data (Serilog)
- Feature flags control bonus feature availability

## Business Rules

1. **Performance Bonus**: Only shown to USX OTR drivers; Dedicated/Total drivers see historical data only
2. **TeamMAX Bonus**: Requires specific company membership (companies 01, 63)
3. **Fast Track**: Drivers must select reward before monthly lock-in date
4. **Replication**: Runs nightly to ensure payroll has current bonus data

## Dependencies

- **Feature Flags**: `DriverBonusSummary`, `PerformanceBonus`, `CacheDriverStats`
- **External Systems**: SQL Server PSA, DB2 QTOPS, RPG Payroll
- **Orleans Cluster**: Requires healthy grain cluster for caching
