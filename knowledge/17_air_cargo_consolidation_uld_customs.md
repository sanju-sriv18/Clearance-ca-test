# Air Cargo Consolidation, ULDs, and Customs Clearance

## Table of Contents
1. [Air Cargo Consolidation Process](#1-air-cargo-consolidation-process)
2. [ULD Types, Sizes, and Identification](#2-uld-types-sizes-and-identification)
3. [MAWB to HAWB to Piece-Level Tracking](#3-mawb-to-hawb-to-piece-level-tracking)
4. [Air Cargo Manifest and Documentation](#4-air-cargo-manifest-and-documentation)
5. [Express Carrier vs. Freight Forwarder Consolidation](#5-express-carrier-vs-freight-forwarder-consolidation)
6. [Customs Clearance of Air Cargo Consolidations](#6-customs-clearance-of-air-cargo-consolidations)

---

## 1. Air Cargo Consolidation Process

### What Is Air Cargo Consolidation?

Air freight consolidation is the practice of combining multiple smaller shipments from different shippers into a single larger shipment for air transport. Instead of booking individual air freight space for each shipment, freight forwarders consolidate compatible cargo and issue a **Master Air Waybill (MAWB)** for the entire load, with each individual shipment assigned a **House Air Waybill (HAWB)**.

This enables shippers to share transportation costs, reduce cargo space waste, and often access better rates than standalone bookings -- savings of 30-50% compared to individual air freight are common.

### The Step-by-Step Consolidation Process

#### Step 1: Shipment Collection
Freight forwarders gather shipments from various shippers, typically bound for the same destination or region. Freight is sorted and grouped based on:
- Destination airport/region
- Weight and dimensions
- Commodity type and compatibility
- Required transit time and service level
- Dangerous goods classification (if applicable)

#### Step 2: Consolidation at the Hub (Build-Up)
At the freight forwarder's warehouse or consolidation hub:
- Shipments are packed together to optimize space within ULDs
- Each individual shipment is documented under a **House Air Waybill (HAWB)**
- The consolidated load is documented under a **Master Air Waybill (MAWB)**
- A **consolidation manifest** accompanies the consolidation, listing all HAWBs, pieces, weights, and commodity descriptions

#### Step 3: ULD Build-Up Process
The physical process of loading cargo into ULDs:

1. **Serviceability Check**: Before build-up, the ULD and all accessories are inspected to ensure they are fit for use. Only personnel with valid certification are permitted to build ULD consols.

2. **Dimension and Weight Verification**: Since several pieces make up a consolidation, a few wrong weights can accumulate to form a large difference. An overloaded ULD might collapse. Floor load weight tolerance limits must be considered.

3. **Physical Loading**: Cargo handlers check dimensions of pieces, ensure no sharp edges, check the ULD before loading, ensure pieces are neutralized effectively (balanced) and labeled properly.

4. **Documentation Matching**: The completed build-up must match the booking list, pre-manifest, and status/priority of special cargo.

5. **Screening**: Any freight moving on a passenger aircraft (rather than a cargo-only plane) must be screened by a **Certified Cargo Screening Facility (CCSF)**. Forwarders without screening facilities must tender cargo loose to airlines, requiring additional handling.

**Airlines strongly prefer freight from forwarders that has already been loaded onto ULDs** -- this makes loading the aircraft fast and efficient.

#### The BUP (Bulk Unitization Program)
Freight forwarders can offer ready-made ULD solutions known as a **BUP** -- ready-to-load packages built at origin and sent as ready-to-ship units:
- BUPs significantly reduce handling time at cargo terminals by combining separate items into a single unit
- BUPs benefit from preferential conditions and rapid handling at busy airports
- At destination, BUPs are turned over directly to the forwarder for unloading, often **3-4 days faster** than airline-loaded ULDs

#### Step 4: Transport to Airport and Tendering
The consolidated shipment is transported to the airport. There are two tendering options:
- **ULD Booking**: The forwarder puts all consignments into a ULD container and tenders the sealed unit to the airline
- **Loose Consolidation**: The forwarder tenders all consignments belonging to one consolidation without a ULD container; the airline then decides whether to containerize or fly loose

#### Step 5: Air Transport
The consolidated load travels as a single MAWB shipment to its destination.

#### Step 6: Break-Down at Destination (Deconsolidation)

Upon arrival at the destination:

1. **Transfer to CFS/Bonded Warehouse**: ULDs are moved from the aircraft to a **Container Freight Station (CFS)** or customs-bonded facility. CBP-bonded CFS facilities allow cargo to be transferred directly from the airport while still under customs bond.

2. **ULD Breakdown**: Import ULDs are broken down based on AWB (Air Waybill) data and handling rules. Each step can be manual, semi-automated, or software-supported.
   - **Fast breakdown** of ULDs is usually completed within **3 hours**
   - Structured and reliable deconsolidation is key -- while transport between zones can be automated, the physical break-down requires human execution

3. **Sorting and Segregation**: Cargo is sorted by consignee/final destination. Each shipment is cataloged and entered into an inventory management system for tracking.

4. **Customs Clearance**: Individual HAWBs are cleared through customs. Cargo remains in the bonded facility until released by customs.

5. **Final Delivery**: After customs release, individual shipments are dispatched for last-mile delivery to consignees.

### Pivot Weight Pricing

Transportation charges for a ULD depend on whether the total weight exceeds a threshold called the **pivot weight**:
- Below pivot weight: charged at the **under-pivot rate** (a flat rate for the ULD)
- Above pivot weight: the excess is charged at the **over-pivot rate** (per kg)

This pricing scheme creates strong incentives for forwarders to optimize how they fill each ULD.

---

## 2. ULD Types, Sizes, and Identification

### ULD Identification System (IATA Resolution 686)

Every ULD in the world carries a unique **10-character identification code** governed by **IATA Resolution 686**:

```
[3-letter prefix] [4-5 digit serial number] [2-3 character owner code]
Example: AKN 12345 DL
         ^^^  ^^^^^  ^^
         |    |      |
         |    |      Owner: Delta Air Lines
         |    Unique serial number
         Type: Forkliftable LD3 container
```

#### Position 1 (Category):
- **A** = Certified container (aircraft ULD container)
- **P** = Certified pallet (aircraft ULD pallet)
- **R** = Thermal (refrigerated) container
- **F** = Non-certified flat pallet
- **Q** = Kevlar/blast-resistant container

#### Position 2 (Base Dimensions):
Each letter corresponds to a specific length and width of the base, regardless of whether it's a container or pallet.

#### Position 3 (Contour/Forklift):
Indicates the contour shape and whether forklift pockets are present.

#### Serial Number (4-5 digits):
The airline's unique number for that specific ULD.

#### Owner Code (2-3 characters):
The airline's IATA 2-letter code (e.g., DL = Delta, AA = American, LH = Lufthansa).

### Scale and Value of Global ULD Fleet
- Approximately **900,000 ULDs** in service worldwide
- Worth more than **US $1 billion** total (averaging ~$1,100 each)
- Annual repair and loss costs: **~$400 million**
- Annual damage costs: **$300 million** in repairs, **$20 million** in lost ULDs

### Common ULD Types -- Complete Reference

#### Lower Deck Containers (Fit under the main deck of wide-body aircraft)

| ATA Type | IATA Code | Base Dimensions | Height | Compatible Aircraft | Notes |
|----------|-----------|-----------------|--------|-------------------|-------|
| **LD1** | AKC (no forks) | 60.4" x 61.5" | 64" | 747 only | Designed specifically for 747 lower deck |
| **LD2** | DPN (forks) / DPE (no forks) | 47" x 61.5" | 64" | 767 | Half-width container for narrower fuselage |
| **LD3** | AKE (no forks) / AKN (forks) | 60.4" x 61.5" | 64" | 787, 777, 747, A300/330/340/350/380, MD-11 | **Most common ULD type worldwide** |
| **LD3** | QKE | Same as AKE | Same | Same | **Kevlar blast-resistant** version |
| **LD3 Reefer** | RKN | Same as AKE | Same | Same | With built-in refrigeration unit |
| **LD3-45** | AKH / AKW | 60.4" x 61.5" | **45"** | A320/A321 | Reduced height for narrow-body lower decks |
| **LD4** | DQP | 96" x 61.5" | 64" | 767 | Like an LD8 without contours |
| **LD6** | ALF | 125" x 61.5" | 64" | 787, 777, 747, wide-body Airbus | Full-width = 2x LD3 positions |
| **LD8** | DQF (forks) | 96" x 61.5" | 64" | 767 | Full-width for 767 |
| **LD11** | ALP (no forks) | 125" x 96" | 64" | 787, 777, 747, wide-body Airbus | Largest lower deck container |
| **LD39** | AMU | ~125" x 96" | 64"+ | Wide-body | Biggest lower-deck container, deep extensions |

#### Pallets

| ATA Type | IATA Code | Base Dimensions | Notes |
|----------|-----------|-----------------|-------|
| **LD7 (88")** | PAG / P1P | 88" x 125" | Standard large pallet |
| **LD7 (96")** | **PMC** | 96" x 125" | **Most common pallet type** -- fits all wide-bodies |
| **LD8 Pallet** | FQA | 96" x 61.5" | Same floor dimensions as DQF container |
| **LD11 Pallet** | FLA / PLA | 125" x 96" | Large pallet |
| **20-ft Pallet** | PGE | 96" x 238.5" | Extra-large pallet |

#### Main Deck Containers

| IATA Code | Type | Dimensions | Compatible |
|-----------|------|------------|------------|
| AAA | LD7 Container | 88" x 125" x 81" | Narrow-body main deck |
| AAD | LD7 Container | 88" x 125" x 96" | Wide-body main deck (A1) |
| AMJ | LD7 Container | 96" x 125" x 96" | Wide-body main deck (M1) |

### Aircraft Compatibility Summary
- **LD3s, LD6s, LD11s**: Fit 787, 777, 747, MD-11, L-1011, all Airbus wide-bodies
- **LD2s, LD8s**: Used on 767s (narrower fuselage)
- **LD1**: Designed for 747 but LD3s are more commonly used due to ubiquity
- **LD3 (45-inch height)**: Also fits Airbus A320 family
- **PMC pallets (96" x 125")**: Fit 787, 777, 747, late-model 767s (larger doors), all wide-body Airbus

### ULD Positioning
- Container capacity is measured in **positions**
- Each half-width container (LD1/LD2/LD3) occupies **one position**
- Each row in a cargo compartment typically has **two positions**
- A full-width container (LD6/LD8/LD11) takes **two positions**
- An LD6 or LD11 can occupy the space of 2x LD3
- A PMC pallet occupies approximately **3x LD3 positions** or **4x LD2 positions**

### ULD Tracking and Management Systems

#### UCM (ULD Control Message)
The UCM is an IATA-standard message that forms the backbone of ULD messaging. When implemented accurately, it provides ULD controllers with all necessary location information. The primary challenge is managing inventory imbalances between locations due to lack of visibility.

#### ULD CARE / IULDUG System
**ULD CARE** is a Toronto-based not-for-profit industry organization (started in the 1970s, independent from IATA since 2011) that manages the **IULDUG (Interline ULD User Group) System**:
- Facilitates interline movements of ULDs amongst member airlines
- Tracks ULDs worldwide when they are outside the control of their owner carrier
- **Demurrage System**: After 5 free days, a demurrage charge is credited to the ULD owner, compensating for the temporary absence
- One airline reduced average ULD return time from **22 days to 9 days** after joining
- Repair frequency for ULDs given to IULDUG participants is **half** that of units given to non-participants

#### Modern Tracking Technologies
- **RFID (Radio-Frequency Identification)**: Does not require line-of-sight scanning, enabling seamless tracking throughout the supply chain
- **BLE (Bluetooth Low Energy)**: Several airlines implementing for improved real-time location
- **LTE/GPS**: Cellular-connected trackers on high-value ULDs
- **Barcoding**: A middle-ground solution -- relatively easy to attach to containers
- **Blockchain**: Being explored for secure, transparent transfer tracking

#### ULD Transfer Documentation
During transfer, the transferring party provides a receipt (paper or electronic) to the receiving party -- the **ULD Control Receipt** is vital for ownership tracking. IATA's ULD Regulations cover identification, registration, transfer documentation, and stock control messaging.

### ULD Ownership Models
1. **Airline-Owned**: Most ULDs are owned by airlines and identified by their IATA code in the ULD ID
2. **Pool ULDs**: Airlines participate in shared pools (via IULDUG) allowing interline transfers with demurrage charges
3. **Leased ULDs**: Some airlines lease ULDs from specialty providers
4. **Freight Forwarder ULDs**: Large forwarders may own their own ULDs for BUP operations

---

## 3. MAWB to HAWB to Piece-Level Tracking

### The Air Cargo Documentation Hierarchy

```
AIRLINE (Carrier)
  |
  +-- MAWB (Master Air Waybill) -- Entire consolidated shipment
  |     Issued by: Airline to Freight Forwarder
  |     Contains: Forwarder name, total pieces/weight, routing
  |     Cargo description: "Consolidation as per Attached Manifest"
  |
  +---- HAWB #1 (House Air Waybill) -- Shipper A's consignment
  |       Issued by: Freight Forwarder to Shipper A
  |       Contains: Actual shipper/consignee, precise cargo description
  |       +-- Piece 1 (carton, pallet, etc.)
  |       +-- Piece 2
  |       +-- Piece 3
  |
  +---- HAWB #2 -- Shipper B's consignment
  |       +-- Piece 1
  |       +-- Piece 2
  |
  +---- HAWB #3 -- Shipper C's consignment
          +-- Piece 1
          +-- ... (up to hundreds of pieces)
```

### Master Air Waybill (MAWB)
- Issued by the airline (or its authorized agent) to the freight forwarder
- Represents the contract of carriage between the airline and the forwarder
- Contains a unique **IATA-standard 11-digit air waybill number** (e.g., 172-12345678)
  - First 3 digits = airline prefix (e.g., 172 = Cathay Pacific)
  - Remaining 8 digits = unique serial number + check digit
- Covers the entire consolidated shipment loaded onto the aircraft
- For consolidations, the MAWB shipper is the forwarder, and the consignee is the destination agent/CFS

### House Air Waybill (HAWB)
- Issued by the freight forwarder to each individual shipper
- Acts as the contract between the shipper and the forwarder
- Contains the **actual shipper and consignee** names and addresses
- Contains a **precise cargo description** (not "consolidation" or "FAK")
- Each HAWB has its own unique number assigned by the freight forwarder
- A single MAWB can represent **many HAWBs** (commonly 5 to 50+)

### Typical Volumes

| Metric | Typical Range | Notes |
|--------|--------------|-------|
| **HAWBs per MAWB** | 1 to 50+ | Depends on trade lane volume and cargo type |
| **HAWBs per ULD** | 5 to 40+ | Highly variable; depends on ULD type and shipment sizes |
| **Pieces per HAWB** | 1 to 100+ | From a single carton to hundreds of boxes on pallets |

There is **no industry-standard fixed number** -- counts depend on forwarder volume, ULD size, and the nature of cargo on a given route.

### Piece-Level Tracking

#### What "Piece Count" Means
- A **piece** is the smallest external packing unit (carton, bag, drum, etc.)
- If a shipment consists of 2 skids, each containing 25 cartons, it is reported as:
  - **No. of Pieces**: 2 (handling units / outer packages)
  - **SLAC (Shipper Load and Count)**: 50 pieces (inner piece count)
- For ULD-level reporting, the quantity must reflect the **smallest external packing unit** (not the ULD itself)

#### Current State of Piece-Level Tracking
Historically, air cargo has been tracked at the **HAWB level**, not the piece level. However, the industry is transitioning:

**IATA ONE Record** (target implementation: January 2026):
- Creates a **single record view** of the shipment using standardized web APIs
- Uses JSON-LD for data exchange with a federated trust network for security
- Over 30 pilot projects active worldwide
- Cathay Cargo became the first carrier to adopt ONE Record in production operations (2025)
- Enables **piece-level tracking** to enhance transparency
- Over 70% of industry respondents showed awareness; ~50% indicated readiness

**Cargo iQ** (Quality Standard):
- Industry system of shipment planning and performance monitoring based on common milestones
- The **Master Operating Plan (MOP)** describes the standard end-to-end process
- Currently developing piece-level tracking capabilities
- Key performance indicators allow members to benchmark carriers
- Integration with ONE Record enables real-time visibility of milestones

**IATA Interactive Cargo**:
- Pilot projects validate real-time tracking for special handling cargo
- Piece-level visibility, tracking, and alerts
- Particularly crucial for pharmaceuticals, perishables, live animals, and high-value items

### How Pieces are Tracked Within a ULD

#### Consolidation Manifest
A detailed consolidation manifest accompanies each consolidation, listing:
- HAWB number
- Number of pieces per HAWB
- Weight per HAWB
- Full shipper and consignee information
- Complete commodity description

For BUP (Built-Up Pallet) shipments, the forwarder must indicate on the consolidation manifest or pallet tag **which HAWB (including number of pieces) is loaded in each ULD**.

#### Reconciliation Requirements
Pieces, weights, and routing must reconcile cleanly between HAWB and MAWB levels to avoid discrepancies and airline disputes.

---

## 4. Air Cargo Manifest and Documentation

### Types of Manifests in Air Cargo

#### 1. Flight Cargo Manifest (CBP Form 7509)
The master document listing all cargo on board a specific flight:
- Required by law for all aircraft entering the United States
- Must contain all required information regarding all cargo on board
- Contains: airline name, flight number, date, routing, nature of goods, shipper/consignee info
- When air waybills are attached, the statement **"Cargo as per air waybills attached"** must appear

The paper CBP Form 7509 is no longer required to be filed aboard aircraft (since electronic AMS replaced it), but other documents like the **General Declaration (CBP Form 7507)** must still be presented upon arrival.

#### 2. MAWB Manifest / Consolidation Manifest
The freight forwarder's document listing all HAWBs within a consolidation:
- Lists each HAWB number, shipper, consignee, pieces, weight, and commodity description
- Accompanies the consolidation to prevent the need to open the document pouch
- Must be provided to airlines for AMS filing purposes
- For BUPs: must show which HAWBs are loaded in each ULD

#### 3. ULD Content List
Documents which specific HAWBs/shipments are loaded in each ULD:
- Essential for targeted inspection (knowing which ULD contains which HAWB)
- Created during the build-up process
- Used during break-down to systematically sort cargo

### Electronic Filing Systems

#### AMS (Automated Manifest System) -- Air
The primary electronic system for pre-arrival cargo data submission to CBP:

**Who Files:**
- **Airlines** file MAWB-level data (carrier, routing, total weight/pieces)
- **Freight forwarders / deconsolidators** file HAWB-level data (actual shipper/consignee, cargo descriptions)
- Any deconsolidator or entry filer with Type 1, 2, or 3 Customs bond can send AMS eManifests

**Filing Deadlines (Air):**
- From distant foreign areas: **4 hours before arrival** in the US
- From nearby foreign areas (Mexico, Central America, Caribbean, northern South America, Bermuda): **At time of departure** for the US

**Required Data Elements (per 19 CFR 122.48a):**
- Air waybill number (IATA standard 11-digit)
- Carrier/ICAO code
- Airport of arrival (3-alpha ICAO code)
- For consolidations: MAWB number AND all associated HAWB numbers
- Shipper name and address
- Consignee name and address
- Cargo description (must be precise -- vague descriptions like "gifts," "parts," "consolidation" are rejected)
- Piece count (smallest external packing unit)
- Weight

**Consolidation-Specific Rules:**
- For the MAWB, shipper = forwarder name, consignee = destination CFS/deconsolidator
- MAWB cargo description reads: **"Consolidation as per Attached Manifest"**
- Each HAWB must have the **actual shipper/consignee** and **precise cargo description**
- Even if a MAWB has only ONE associated HAWB, **both must be reported separately**
- Descriptions no longer allowed: FAK (Freight All Kinds), "General Cargo," "STC," "Consol within Consol," "Co-Load"

**AMS Deconsolidator Option:**
- Deconsolidators with a **FIRMS code** can send HAWB data directly to CBP independent of the carrier's MAWB data
- The carrier must indicate the deconsolidator's FIRMS code or ABI filer code with the MAWB data

**Split Shipments:**
When cargo under a single MAWB is split across multiple flights, the carrier must report additional routing information for each HAWB.

**Penalties:**
- Per irregularity per MAWB/HAWB: **$5,000 - $10,000**
- Late or inaccurate filings: shipment detention and increased scrutiny

#### ACAS (Air Cargo Advance Screening)
A separate security-focused pre-screening system, required **in addition to AMS**:

**Purpose:** Enables CBP and TSA to identify and prevent high-risk cargo from being loaded onto aircraft destined for the US.

**Filing Requirement:** Data must be submitted at the **lowest air waybill level**:
- For consolidated shipments: **HAWB level** (mandatory)
- For non-consolidated shipments: Regular AWB level
- Must be filed **prior to loading cargo onto the aircraft** (earlier is better)

**Required ACAS Data Elements (Original 6):**
1. Shipper name and address
2. Consignee name and address
3. Cargo description (precise -- not "gifts," "daily necessities," "accessories," etc.)
4. Total quantity (smallest external packing unit)
5. Total weight
6. Air waybill number
Plus: MAWB number as a conditional element for consolidations

**Enhanced ACAS (November 2025 Rule):**
Triggered by **July 2024 incendiary device incidents** at European air cargo facilities, CBP published substantially expanded requirements:
- Additional data elements about parties involved in cargo transactions
- Financial data related to cargo shipments
- Information about online marketplaces
- Data on shipments from individuals with unknown risk profiles
- **12-month phased enforcement** period for industry compliance

**Who Files ACAS:**
- If no other eligible party elects to file, the **inbound air carrier must file**
- Eligible filers also include freight forwarders and Foreign Indirect Air Carriers (FIACs)

**Consequences of Non-Compliance:**
- Cargo **will not be allowed to be loaded onto the aircraft** until ACAS is submitted and verified
- If anomalies are discovered, a **Do Not Load (DNL)** order may be issued

### Transmission Formats
- **XFFM**: XML Flight Manifest Message (IATA Cargo-XML standard)
- **IATA Cargo IMP**: Cargo Interchange Message Procedures (legacy format)
- **EDI**: Electronic Data Interchange
- **API**: Application Programming Interfaces (for modern integrations)

---

## 5. Express Carrier vs. Freight Forwarder Consolidation

### Fundamental Difference: Integrators vs. Intermediaries

**Express Carriers (Integrators)** -- FedEx, UPS, DHL:
- Own and operate their **own aircraft, trucks, sorting hubs, and delivery networks**
- Maintain **full custody** of packages from pickup to delivery (door-to-door)
- Operate on a **hub-and-spoke model** with massive central sorting facilities
- Sort by **destination** using automated systems, not by shipper/consignee relationship
- Operate their own **customs brokerage** services

**Freight Forwarders:**
- Are **intermediaries** that arrange transport using airlines' cargo capacity
- **Do not own aircraft** (generally)
- Consolidate by **trade lane** (origin-destination pairs) grouping multiple shippers' cargo
- Issue HAWBs to shippers and hold the MAWB with the airline
- Use BUP bookings or tender loose cargo to airlines
- May use external customs brokers

### How Express Carriers Build ULDs

Express carriers build ULDs as part of an **automated, destination-based sorting operation**:

1. **Pickup**: Individual packages are collected from customers via trucks/couriers
2. **Local Sort**: Packages arrive at local stations, are scanned, and trucked/flown to the central hub
3. **Hub Arrival**: ULDs arrive at the central hub, are offloaded from aircraft, and opened at unloading nodes
4. **Automated Sorting**: Packages enter conveyor systems with barcode scanning tunnels:
   - **Six-sided scanning** reads all sides of each package
   - Systems capture dimensions, weight, barcode, and destination zip code
   - Packages are sorted to **outbound positions** based on destination routing
   - The entire process takes ~13 minutes per package (UPS Worldport)
5. **ULD Build**: Sorted packages are loaded into outbound ULDs organized by destination
6. **Outbound Flight**: ULDs are loaded onto aircraft and dispatched

### Express Hub Operations in Detail

#### FedEx -- Memphis World Hub
- **Scale**: 940 acres, 171 aircraft gates, 84 miles of conveyor belt, 13,000 team members
- **Capacity**: 484,000 packages per hour
- **New "Secondary 25" Facility** (opened 2024): 1.3M sq ft, 4 levels, 11 miles of conveyor, sorts 56,000 packages/hour
- **Technology**: Six-sided scanning, 460 spiral chutes between floors, 1,000 monitoring cameras
- **Operations**: Peak 10 PM - 5 AM; Memphis becomes the **busiest airport in the world** during this window
- **Hub Operations Command Center (HOCC)**: Centralized control overseeing all operations with visibility into every package

#### UPS -- Louisville Worldport
- **Scale**: 5.2 million sq ft, 70 aircraft docks, 155 miles of conveyor belt
- **Capacity**: 416,000 packages per hour (7,000/minute at peak)
- **Technology**: 546 camera tunnels scan barcodes and route packages; $100M of homegrown software
- **ULD Handling**: Up to 39 ULDs per 747-400; 1.2 million wheels and bearings on floors/lifts/aircraft
- **Three sort streams**: Smalls (letters), regular parcels, and "incompatibles" (heavy/odd-shaped)
- **Automation**: Most packages touched by humans only **twice** (unload and reload); rest is fully automated
- **Operations**: Primarily 11 PM - 3 AM; up to 125 aircraft arrive nightly (~1 per minute at peak)
- **Weight & Balance**: Networked **Distributed Weight and Balance System** ensures ULDs are loaded in proper aircraft sequence

#### DHL -- Leipzig Hub
- **Scale**: Largest facility in DHL Express network; largest air freight hub in the world
- **Capacity**: 150,000 shipments per hour; up to 500,000 items per night
- **Technology**: 47 km (29 miles) of main sorting system; Vanderlande automated sortation
- **Turnaround**: Average **2 hours** from unloading to re-sorted and reloaded onto outbound aircraft
- **Operations**: Starts at 9 PM, peaks midnight-1 AM; virtually empty during the day
- **Network**: Flies to almost 50 destinations from Leipzig alone

### How Freight Forwarders Build ULDs Differently

| Aspect | Express Carriers | Freight Forwarders |
|--------|-----------------|-------------------|
| **ULD Control** | Build ULDs at their own hub/sort facilities with automated systems | Either build ULDs at origin warehouse (BUP) or tender cargo loose to airline |
| **Shipment Type** | Primarily small parcels and documents; sorted by destination | Multiple shippers' larger freight consolidated under one MAWB; sorted by trade lane |
| **Sorting Method** | Automated barcode/scanning systems sort millions of packages by zip/destination | Manual or semi-manual process; forklift operators load cargo into ULDs by hand |
| **Network** | Own aircraft, trucks, and hubs; hub-and-spoke topology | Book space on passenger and cargo airlines; linear routing |
| **Documentation** | Carrier AWB for each package; may have millions of AWBs per flight wave | MAWB from airline + individual HAWBs for each shipper's consignment |
| **Pricing** | Per-package rates with volume/contract discounts | Pivot weight pricing per ULD; under-pivot and over-pivot rates |
| **Speed** | Packages touch humans only at load/unload; 13-minute sort-to-sort | Manual consolidation process; days of cargo accumulation before tendering |
| **Customs** | In-house brokerage; pre-clearance while aircraft is in flight | May use external customs brokers; clearance after arrival |

### Express Consolidation Services for International Shipments
Express carriers offer branded consolidation services for international customs efficiency:
- **FedEx International Priority Distribution**
- **UPS World Ease**
- **DHL BBX**

In these services, the shipper consolidates all packages destined for a specific country through the same clearance facility. Packages clear customs as a **single shipment** instead of individual entries, speeding clearance and potentially reducing duties.

---

## 6. Customs Clearance of Air Cargo Consolidations

### How CBP Processes Consolidated Air Cargo Arrivals

#### Pre-Arrival Phase (ACAS + AMS)
1. **ACAS filing** at HAWB level before loading at foreign origin (for risk assessment)
2. **AMS filing** at both MAWB and HAWB levels, 4 hours before arrival
3. **Automated Targeting System (ATS)** analyzes data for risk indicators
4. CBP determines: cleared, hold, or Do Not Load (DNL)

#### Arrival Phase
1. Aircraft arrives at US port of entry
2. Cargo is transferred to airline cargo facility or CFS/bonded warehouse
3. ULDs are broken down
4. Individual HAWB shipments are sorted and staged for customs clearance
5. CBP releases cleared shipments; holds targeted shipments for examination

### ACAS and Its Relationship to Consolidations

ACAS data must be filed at the **lowest air waybill level** -- meaning HAWB level for consolidations. This is because:
- The MAWB alone does not provide sufficient targeting information (shipper is just the forwarder, description is "consolidation")
- The **HAWB provides the actual shipper, consignee, and commodity description** that CBP needs for risk assessment
- Each of the 6 required ACAS data elements provides CBP with crucial targeting information

**Pre-departure screening:**
- ACAS data must be submitted **before loading** cargo onto the aircraft
- If ACAS filing is not submitted/verified: cargo **cannot be loaded**
- If anomalies detected: **Do Not Load (DNL)** order issued

**Enhanced ACAS (2025):**
- Requires additional data elements about all parties in the transaction
- Financial data and marketplace information now required
- Triggered by July 2024 incendiary incidents at European cargo facilities

### When CBP Targets a Specific HAWB Within a Consolidated ULD

This is where consolidation creates significant operational challenges:

#### The Cascading Hold Problem

**"If a hold is placed on one House Air Waybill (HAWB), the entire Master Air Waybill (MAWB) will not be uplifted."**

Furthermore:

**"If a MAWB/HAWB is co-loaded on a ULD with other MAWBs, the entire ULD will be held."**

This means:
1. **One suspicious HAWB** triggers a hold on the **entire MAWB**
2. If that MAWB is in a ULD with other MAWBs, the **entire ULD is held**
3. All other shippers' cargo in that ULD is delayed
4. The ULD must be broken down to extract the targeted shipment

#### Types of Examinations
- **Non-Intrusive Inspection (NII) / X-Ray / VACIS**: 1-2 days typical; ULD or cargo is scanned
- **Tail Gate Exam**: Physical opening and visual inspection; 3-5 days
- **Intensive Examination**: Full physical exam of contents; **1 week to 30 days** depending on queue

#### Impact on Other Shipments
- All shipments in the same MAWB are held until the targeted HAWB is examined and released (or seized)
- If co-loaded in a ULD, all MAWBs in that ULD may be affected
- Other shippers' transit times can increase by days or weeks
- Additional fees may be incurred: demurrage, storage, re-handling
- For LCL shipments, exam fees are **proportionally divided** between all shippers using the same container

### The Extraction Process
When CBP targets a specific HAWB within a ULD:
1. The entire ULD is held at the cargo facility / CFS
2. The ULD must be broken down to locate the specific packages for the targeted HAWB
3. The consolidation manifest / ULD content list identifies which packages belong to the targeted HAWB
4. Targeted packages are extracted for examination
5. Non-targeted cargo may remain held pending the outcome of the exam
6. Examination may involve X-ray, physical inspection, or lab testing
7. Results are documented in the **Cargo Examination Reporting and Tracking System (CERTS)**

### Reporting Requirements During Examination
- When reporting quantity for cargo in ULDs, AMS participants must report based on the **smallest external packing unit** (not the ULD or pallet count)
- Precise cargo descriptions are required -- "gift," "daily necessities," "accessories," "parts," and "consolidated" are **rejected**
- A precise narrative description must allow CBP to identify the shapes, physical characteristics, and likely packaging of manifested cargo

### Section 321 (De Minimis) Entries Within Consolidated ULDs

#### Background
Section 321 allows importation of articles free of duty and tax where the aggregate fair retail value does not exceed **$800 per person per day**.

#### How De Minimis Applies to Consolidated Air Cargo

CBP evaluates Section 321 eligibility based on the **document used to file or support entry**:

1. If an **individual bill of lading/HAWB** is submitted: de minimis is applied to the value of that individual shipment
2. If a **master bill of lading/MAWB** is submitted: de minimis is applied to the **total value** of all shipments on the MAWB

This means: **Each HAWB within a consolidation can independently qualify for de minimis** if filed at the HAWB level, as long as each individual HAWB value is under $800.

#### Key Requirements for De Minimis Entries in Consolidations
- **Ultimate consignee must be a US entity** (not the foreign shipper)
- The foreign shipper and US ultimate consignee **cannot be the same entity**
- Individual US addresses are required (not generic warehouse addresses)
- Per-person aggregation: If one person imports >$800 on one day, de minimis is denied for the excess

#### Entry Type 86
- Created in 2019 specifically for de minimis shipments requiring data submission
- Filing deadline changed to **"upon or prior to arrival"** (effective February 15, 2024)
- Previously, Type 86 could be filed within 15 days of arrival

#### Volume and Enforcement
- De minimis entries grew from **153 million (2015)** to over **1 billion (2023)**
- De minimis comprises **92% of all US cargo entries**
- CBP processes **~4 million de minimis shipments daily**
- **88% arrive via air**, international mail, or express couriers (FedEx, UPS, DHL)

#### ACE Aggregation Enhancement (August 2025)
CBP deployed an enhancement to ACE that **automatically withholds release** of de minimis shipments that exceed the $800 per-person/per-day threshold.

#### Restrictions on Chinese-Origin Goods
If China or Hong Kong is the country of manufacture, production, or growth, **Section 321 entry is not available** (as of the most recent enforcement actions). This is a significant change affecting the massive volume of e-commerce from China.

#### Recent Elimination of De Minimis (2025)
The Trump administration eliminated the de minimis exemption for sub-$800 imports (effective August 29, 2025). This means:
- Every shipment now undergoes the formal customs process
- ACAS and AMS filings are required for all shipments regardless of value
- Express carriers like FedEx, UPS, and DHL have been developing **consolidated destination clearance** solutions
- Some US-bound packages have been held for prolonged periods as shippers adjust

### Express Carrier Customs Clearance Advantages

Express carriers achieve rapid customs clearance through several mechanisms:

#### 1. Express Consignment Carrier Facilities (ECCFs)
- Specialized CBP-authorized facilities for clearing express shipments
- Essentially bonded warehouses with high-volume processing capability
- Integrators operate ECCFs at major ports: LAX, SFO, JFK, ORD, MIA, CVG
- Most parcel shipments in ECCFs historically fell under Section 321

#### 2. Pre-Clearance Technology
- FedEx **EXPRESSCLEAR**: Fully automated customs electronic system transmitting data while aircraft is in flight
- DHL: **Over 95% of products are approved for delivery while the plane is still in the air**
- UPS: Brokerage team clears **more than 90% of packages on first day of entry**

#### 3. In-House Customs Brokerage
- FedEx: One of the largest customs entry filers worldwide
- DHL: Over 700 dedicated customs services experts
- UPS: Integrated brokerage with automated classification and compliance

#### 4. Consolidated Customs Processing
Express carriers process ULDs containing thousands of individual packages through their ECCFs, using:
- Automated manifest systems connected to CBP
- Pre-arrival data submission (ACAS + AMS) for every package
- Rapid scanning and identification at the facility
- Exception-based processing (only flagged shipments are pulled for exam)

### How to Reduce Risk of Customs Delays in Consolidations

1. **Precise cargo descriptions** on all HAWBs (avoid vague terms)
2. **Accurate HS codes** (6-digit minimum)
3. **Complete shipper/consignee information** with valid addresses
4. **C-TPAT certification** -- trusted shippers experience fewer exams and faster clearance
5. **Timely ACAS/AMS filing** -- file as early as possible before loading
6. **Proper piece counts** matching physical cargo
7. **Avoid co-loading sensitive commodities** with routine goods in the same ULD

---

## Sources

### Air Cargo Consolidation Process
- [Supath - Planning Air Freight Consolidation](https://www.supath.net/planning-air-freight-consolidation/)
- [iContainers - Air Freight Consolidation](https://www.icontainers.com/understanding-air-freight-consolidation-benefits-process/)
- [Dimerco - Air Cargo Consolidation Services](https://dimerco.com/blog-post/air-cargo-consolidation-services/)
- [Visiwise - Air Freight Forwarding Process](https://www.visiwise.co/blog/air-freight-forwarding-process/)
- [IATA - Tips for Shippers & Freight Forwarders](https://www.iata.org/en/publications/newsletters/iata-knowledge-hub/tips-for-shippers--freight-forwarders-on-how-to-prepare-cargo-for-air-transport/)

### ULD Types, Sizes, and Identification
- [Wikipedia - Unit Load Device](https://en.wikipedia.org/wiki/Unit_load_device)
- [VRR Aero - ULD Identification Code](https://vrr.aero/knowledge-center/uld-info/uld-id-code/)
- [VRR Aero - ATA/IATA Conversion Table](https://vrr.aero/knowledge-center/uld-info/conversion-table/)
- [ULD CARE - ULD Types](https://www.uldcare.com/uld-tools-and-solutions/uld-types/)
- [ULD CARE - ULD Identification Codes Demystified](https://www.uldcare.com/articles/library/care/uld-identification-codes-demystified/)
- [IATA - ULD Regulations (ULDR)](https://www.iata.org/en/publications/manuals/uld-regulations/)
- [Xerafy - Tracking Aircraft ULD with RFID](https://xerafy.com/case-study/uld-rfid-visibility-sustainability-air-freight/)
- [ULD CARE - ULD Management & Control](https://www.uldcare.com/uld-explained-book/uld-management-and-control/)
- [ULD CARE - ULD Transfers Between Different Parties](https://www.uldcare.com/articles/library/care/uld-transfers-between-different-parties/)
- [ULD CARE - IULDUG System](https://www.uldcare.com/iuldug-system/)

### MAWB / HAWB / Piece-Level Tracking
- [Ship4wd - MAWB vs HAWB](https://ship4wd.com/import-guides/mawb-vs-hawb)
- [CargoEZ - HAWB vs MAWB](https://cargoez.com/blog/hawb-mawb-freight-forwarders-guide)
- [American Airlines Cargo - US Customs Information](https://www.aacargo.com/learn/customs.html)
- [Emotrans - Airway Bill Guide](https://www.emotrans-global.com/blog/airway-bill-guide/)
- [IATA - ONE Record](https://www.iata.org/en/programs/cargo/e/one-record/)
- [IATA - Cargo iQ](https://www.iata.org/en/programs/cargo/cargoiq/)
- [Cathay Cargo - ONE Record Interchange](https://www.cathaycargo.com/en-us/stories/our-business/iata-one-record-data-exchange.html)
- [Cathay Cargo - How Cargo iQ Is Shaping the Future](https://www.cathaycargo.com/en-us/stories/cargo-in-action/how-cargo-iq-is-shaping-the-future.html)

### Air Cargo Manifest and Documentation
- [CBP - AMS Air Features](https://www.cbp.gov/trade/acs/ams/air-features)
- [CBP - Air AMS FAQ (PDF)](https://www.cbp.gov/sites/default/files/documents/air_faq_cargo.pdf)
- [CBP - Form 7509 Air Cargo Manifest](https://www.cbp.gov/document/forms/form-7509-air-cargo-manifest)
- [19 CFR 122.48 - Air Cargo Manifest](https://www.law.cornell.edu/cfr/text/19/122.48)
- [19 CFR 122.48a - Electronic Information for Air Cargo](https://www.ecfr.gov/current/title-19/chapter-I/part-122/subpart-E/section-122.48a)
- [CustomsCity - What is the CBP Automated Manifest System?](https://customscity.com/what-is-the-cbp-automated-manifest-system-ams/)
- [Magaya - AMS Filing](https://www.magaya.com/automated-manifest-system/)
- [Trade Finance Global - Air Cargo Manifest](https://www.tradefinanceglobal.com/freight-forwarding/air-cargo-manifest/)
- [Korean Air Cargo - AMS Transmission Guide](https://cargo.koreanair.com/en/services/AMS-Transmission-Guide)

### ACAS
- [Federal Register - Enhanced ACAS (Nov 2025)](https://www.federalregister.gov/documents/2025/11/21/2025-20606/enhanced-air-cargo-advance-screening-acas)
- [Federal Register - ACAS (2018)](https://www.federalregister.gov/documents/2018/06/12/2018-12315/air-cargo-advance-screening-acas)
- [19 CFR 122.48b - ACAS](https://www.law.cornell.edu/cfr/text/19/122.48b)
- [CBP - ACAS Implementation Guide (PDF)](https://www.cbp.gov/sites/default/files/2024-09/ACAS%20IG%20v2.3.1_508.pdf)
- [CBP - Enhanced ACAS FAQ (PDF)](https://www.cbp.gov/sites/default/files/2024-09/Enhanced%20ACAS%20FAQ%20CBP%20V2.3%20Sep%2023%20(508%20compliant)_0.pdf)
- [Lufthansa Cargo - Enhanced ACAS Requirements](https://www.lufthansa-cargo.com/en/l/52350040)
- [ALS - Enhanced ACAS Requirements](https://www.als-int.com/insights/posts/enhanced-acas-requirements-cbp-air-cargo-security-compliance/)
- [Descartes - ACAS FAQs](https://www.descartes.com/resources/knowledge-center/air-cargo-advance-screening-acas-faqs)

### Express Carrier Operations
- [FedEx Newsroom - Automated Sorting Facility at Memphis](https://newsroom.fedex.com/newsroom/global-english/fedex-unveils-new-automated-sorting-facility-at-memphis-world-hub)
- [Supply Chain Dive - FedEx Memphis Hub Automation](https://www.supplychaindive.com/news/fedex-memphis-hub-secondary-25-automation/731938/)
- [AeroXplorer - Memphis at Midnight: Inside FedEx's Global Superhub](https://aeroxplorer.com/articles/memphis-at-midnight-inside-fedexs-global-superhub.php)
- [AeroSavvy - Inside Louisville's UPS Worldport](https://aerosavvy.com/ups-worldport/)
- [Engadget - Inside UPS Worldport](https://www.engadget.com/2013-01-03-inside-ups-worldport-sorting-hub.html)
- [Gizmodo - Inside the Automated UPS Complex](https://gizmodo.com/inside-the-automated-ups-complex-that-sorts-7-000-packa-1705368971)
- [DHL - Hub Leipzig Logistics](https://www.dhl.com/de-en/microsites/express/hubs/hub-leipzig/discover/logistics.html)
- [DHL - Sorting at the Speed of Yellow](https://www.dhl.com/global-en/delivered/life-at-dhl/sorting-at-the-speed-of-yellow-dhls-hub-leipzig-is-10.html)
- [Vanderlande - DHL Leipzig](https://www.vanderlande.com/references/dhl-leipzig/)
- [FreightWaves - DHL to Consolidate Airport Hubs](https://www.freightwaves.com/news/dhl-to-consolidate-airport-hubs-add-regional-sort-centers)
- [FedEx - Freight Forwarders vs Integrators](https://www.fedex.com/en-de/shipping-channel/preparing-shipment/freight/freight-forwarders-vs-integrators.html)

### Customs Clearance of Consolidations
- [CBP - Cargo Examination](https://www.cbp.gov/border-security/ports-entry/cargo-security/examination)
- [OIG DHS - Cargo Targeting and Examination (PDF)](https://www.oig.dhs.gov/sites/default/files/assets/Mgmt/OIG_10-34_Jan10.pdf)
- [Air Cargo News - US Customs Clamps Down on Vague Cargo Descriptions](https://www.aircargonews.net/us-customs-clamps-down-on-vague-cargo-descriptions/1073613.article)
- [CustomsCity - What is an ECCF?](https://customscity.com/what-is-an-express-consignment-carrier-facility-eccf/)
- [Easyship - FedEx Customs Clearance](https://www.easyship.com/blog/fedex-customs-clearance-process)
- [Supply Chain Dive - How UPS, FedEx and DHL Handle Customs Post-De-Minimis](https://www.supplychaindive.com/news/package-disposal-policies-ups-fedex-dhl/803468/)

### Section 321 / De Minimis
- [CBP Ruling H290219](https://rulings.cbp.gov/ruling/H290219)
- [CBP - Section 321 Programs](https://www.cbp.gov/trade/trade-enforcement/tftea/section-321-programs)
- [CBP - Section 321 Aggregated Shipments Notice (PDF)](https://www.cbp.gov/sites/default/files/2025-06/trade_information_notice_cbp-209_508c_0.pdf)
- [Congress.gov - Section 321 De Minimis CRS Report](https://www.congress.gov/crs-product/R48380)
- [CustomsCity - Historical Overview of Section 321](https://customscity.com/from-past-to-present-a-historical-overview-of-section-321-and-its-influence-on-customs-compliance/)
- [UPS - Entry Type 86 Changes](https://www.ups.com/us/en/supplychain/resources/news-and-market-updates/entry-type-86-changes)
- [Jet Worldwide - New Regulations for Section 321](https://www.jetworldwide.com/blog/changes-to-section-321)

### ULD Build-Up and Breakdown
- [ULD CARE - Build-up & Breakdown](https://www.uldcare.com/uld-explained-book/buildupbreakdown/)
- [Lodige - Build & Break Workstations](https://www.lodige.com/en-global/products/airport-logistics/air-cargo-terminal-solutions/buildbreak/)
- [Dimerco - Building Airline ULDs](https://dimerco.com/blog-post/building-airline-uld-need-to-know/)
- [Ship4wd - Deconsolidation](https://ship4wd.com/resource-center/glossary/deconsolidation)
- [Shapiro - Deconsolidation](https://www.shapiro.com/resources/deconsolidation/)
- [Ground Air Co - CFS Services](https://groundairco.com/services/cfs/)
