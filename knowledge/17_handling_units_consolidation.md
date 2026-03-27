# Handling Units, Consolidation, and Physical Cargo Hierarchy in International Logistics

> Clearance Intelligence Engine -- Operational Knowledge Base
> Last Updated: February 2026

## 1. Handling Unit (HU) -- Definition and Core Concept

### What Is a Handling Unit?

A **handling unit (HU)** is a physical, packaged entity that can be handled, moved, tracked, and stored as a single item during warehousing, shipping, and logistics operations. It is always a combination of **packaging material** (the load carrier -- pallet, box, crate, drum, etc.) and the **goods contained within it**.

Every distinct piece of packaged freight that will be moved with a forklift, pallet jack, or by hand is a handling unit. For example:
- Six boxes on one pallet = **one handling unit**
- Six boxes on one pallet plus one standalone palleted crate = **two handling units**
- A single parcel shipped via express courier = **one handling unit**

The concept originated in warehouse management (particularly in ERP systems like SAP) but has become standard across the entire logistics chain -- from warehouse to truck to port/airport to customs to final delivery.

### Key Properties of a Handling Unit

| Property | Description |
|----------|-------------|
| **Unique Identifier** | Every HU has a unique identification number (internal or external). Scannable via barcode (GS1-128, DataMatrix) or RFID. |
| **Packaging Material** | The physical container/carrier: pallet, carton, crate, drum, bag, envelope, etc. |
| **Contents** | The goods or other HUs packed inside. |
| **Dimensions** | Length, width, height of the unit. |
| **Weight** | Gross weight (goods + packaging) and tare weight (packaging only). |
| **Nesting/Hierarchy** | HUs can be nested -- a pallet HU can contain box HUs, which contain individual item HUs. |

### The Hierarchy: Items, Handling Units, Shipments

The physical cargo hierarchy in logistics follows this pattern:

```
Shipment (Bill of Lading / Air Waybill)
  └── Handling Unit (Pallet, Crate, or Parcel)
        └── Handling Unit (Box, Carton -- nested inside)
              └── Trade Items (individual goods/products)
```

**Critical relationships:**
- A **shipment** consists of one or more handling units.
- A **handling unit** can contain trade items (goods) or other handling units (nesting).
- There is **not** necessarily a 1:1 relationship between handling units and freight units. One freight unit can contain several handling units.
- When a delivery includes handling unit items, all product items packaged into one HU are included in one freight unit -- they are never split across multiple freight units during transportation planning.
- Multiple handling units from different shippers can be grouped into a **consolidation** (see Section 5).

### How Major Logistics Companies Use Handling Units

**FedEx Freight (LTL):**
- FedEx Freight offers **handling unit-level tracking** for multiple-handling-unit shipments.
- Each handling unit receives a **unique tracking number**, but the entire shipment moves on **one Bill of Lading, one Delivery Receipt, and one Invoice Statement**.
- FedEx Ground/Express tracking numbers are 12 digits (expandable to 14), with overall barcode length of 34 digits.

**UPS:**
- UPS tracking numbers start with "1Z" followed by a 6-character shipper number, a 2-digit service level indicator, and 8 digits identifying the package (last digit is a check digit) -- totaling 18 characters.
- Each package (handling unit) has its own tracking number.

**DHL Express:**
- DHL Express supports the ISO 15459-1 standard for individual package identification (uppercase characters and numerals, up to 35 digits).
- DHL also uses **10-digit numeric identifiers** to track transport orders (one order = one shipment of one or more pieces from A to B).
- Customers can track at both the piece level and transport order level.

**Kuehne+Nagel:**
- Tracking available by KN tracking number, KN reference, customer reference, container/package/shipment number, or H/AWB number.
- Public Tracking (myKN) provides shipment-level visibility.

**DB Schenker (now part of DSV):**
- Tracking via STT (Schenker Track and Trace) number, container number, Air Waybill number, or House Bill of Lading number.
- 32,000+ weekly scheduled services in Europe for general cargo operations.

**Key insight for software:** While each carrier uses proprietary tracking number formats, the universal concept is that every physically distinct piece has a unique scannable identifier, and shipments group multiple pieces under a single transport document (BOL, AWB).

---

## 2. Handling Unit Tracking and Identification

### Serial Shipping Container Code (SSCC) -- The GS1 Standard

The **SSCC** is the globally standardized identifier for logistics units, defined by GS1. It is the "license plate" for handling units.

**SSCC Structure (18 digits):**

```
AI(00) | Extension Digit | GS1 Company Prefix | Serial Reference | Check Digit
  00   |       1         |     7-10 digits     |    6-9 digits    |      1
```

- **Application Identifier (00)**: Indicates the data field contains an SSCC.
- **Extension Digit (1 digit)**: Increases the capacity of the Serial Reference. No defined logic -- assigned by the company constructing the SSCC.
- **GS1 Company Prefix (7-10 digits)**: Assigned by GS1 to the company.
- **Serial Reference (6-9 digits)**: Assigned by the company to uniquely identify the specific logistics unit. Cannot be reused for a minimum of 12 months.
- **Check Digit (1 digit)**: Calculated using the Modulo 10 algorithm.
- Combined length of GS1 Company Prefix + Serial Reference is always 16 digits.

**Example:** An SSCC might look like `00 0 0614141 000000001 8` (with spaces for readability).

### Barcode Encoding

- The SSCC is typically encoded into a **GS1-128 barcode** on a GS1 Logistics Label.
- **GS1 DataMatrix** and **GS1 QR Code** are also permitted (with some restrictions).
- The GS1-128 barcode includes a Function Code 1 (FNC1) character that signals GS1 Application Identifiers within the data, allowing scanning systems to parse the information correctly.

### GS1 Logistic Label

The GS1 Logistic Label has three sections:
1. **Top section**: Company name, logo, or other free-form information.
2. **Middle section**: SSCC barcode plus any additional data encoded using GS1 Application Identifiers (e.g., lot number, expiry date, GTIN).
3. **Bottom section (carrier segment)**: Transport-specific information known at time of shipment.

**Standard label sizes:**
- **Compact (A6)**: 105mm x 148mm / 4" x 6" -- suitable for SSCC-only labeling.
- **Large (A5)**: 148mm x 210mm / 6" x 8" -- suitable for pallet labels with additional trade item data.

### ISO 15459 Compatibility

The SSCC is fully compatible with **ISO/IEC 15459** (unique identifiers for transport units), commonly referred to as the "ISO License Plate." This is a prerequisite for tracking and tracing logistics units internationally.

### Carrier-Specific vs. Universal Tracking Numbers

| System | Format | Scope |
|--------|--------|-------|
| **SSCC (GS1)** | 18-digit, AI(00) | Universal, cross-company |
| **FedEx tracking** | 12-14 chars | FedEx network only |
| **UPS tracking** | 18 chars, "1Z" prefix | UPS network only |
| **DHL tracking** | 10-digit numeric (transport order) or up to 35 chars (ISO 15459) | DHL network |
| **IATA ULD ID** | 3 letters + 4-5 digits + 2-char suffix | Air cargo ULDs |

**Software implication:** A handling unit may carry multiple identifiers simultaneously -- an SSCC for supply chain traceability, a carrier tracking number for transport visibility, and an internal warehouse HU number for WMS operations. The system must support multiple identifier types per handling unit.

---

## 3. Handling Unit Types and Classification

### Common Physical Types

| Type | Description | Typical Use |
|------|-------------|-------------|
| **Pallet** | Platform (usually wood or plastic) on which goods are stacked | Ground freight, warehouse storage, LCL/FCL |
| **Carton/Box** | Corrugated cardboard or rigid box | General merchandise, e-commerce |
| **Crate** | Wooden or metal open/slatted box | Heavy machinery, fragile goods |
| **Drum** | Cylindrical container (steel, plastic) | Chemicals, liquids, bulk powders |
| **Barrel** | Cylindrical container (traditional wood) | Beverages, bulk commodities |
| **Bag/Sack** | Flexible material container | Grain, chemicals, textiles |
| **Envelope** | Flat, flexible packaging | Documents, small flat items |
| **Roll/Reel** | Cylindrical wound material | Fabric, paper, cable, film |
| **Jerrycan** | Portable container (plastic/metal) | Fuels, chemicals |
| **IBC (Intermediate Bulk Container)** | Large rigid container (typically 1,000L) | Bulk liquids, chemicals |
| **Bundle** | Items strapped/banded together | Lumber, pipes, steel bars |
| **Loose piece** | Individual unpackaged item | Oversized/project cargo |

### UNECE Recommendation No. 21 -- International Package Type Codes

The United Nations Economic Commission for Europe (UNECE) Recommendation No. 21 establishes the international standard for coding package types. This is the code system used in customs declarations, bills of lading, and EDI messages worldwide.

**Structure:** Three independent code systems that can be used alone or in combination:
1. **Cargo type code** (numeric) -- classifies the type of cargo
2. **Package type code** (2-letter alphabetic) -- identifies the specific package type
3. **Packaging material code** (numeric) -- identifies the construction material

**Examples of 2-letter package type codes commonly seen in customs/trade:**

| Code | Package Type |
|------|-------------|
| PK | Package (generic) |
| CT | Carton |
| BX | Box |
| PL | Pallet |
| CS | Case |
| DR | Drum |
| BG | Bag |
| CR | Crate |
| EN | Envelope |
| RL | Reel |
| BA | Barrel |
| SK | Skeleton case |
| CY | Cylinder |
| JR | Jar |
| TB | Tub |
| PI | Pipe |
| BD | Bundle |
| NE | Unpacked / Loose |

**Cargo type classifications include:**
- **Large freight containers**: Goods in/on containers 20ft (6m) or more (includes standard shipping containers).
- **Other freight containers**: Goods in containers less than 20ft, including rigid IBCs and aircraft ULDs.
- **Break-bulk cargo**: General cargo such as boxes, drums, bags handled individually.

### UN Packaging Codes (for Dangerous Goods)

For hazardous materials, the UN Packaging Code system classifies packaging by:
- **First digit**: Packaging type (1=drum, 2=wooden barrel, 3=jerrycan, 4=box, 5=bag, 6=composite, 7=pressure receptacle)
- **Letter**: Material of construction (A=steel, B=aluminum, C=natural wood, D=plywood, G=fiberboard, H=plastic, etc.)
- **Additional digit**: Sub-type (1=closed head, 2=open head)
- **Packing Group**: X=Group I (high danger), Y=Group II (medium), Z=Group III (low)

Example: **1A1/X** = Steel drum, closed head, tested for Packing Group I.

### IATA Cargo Classification (Air Freight)

IATA classifies cargo into:
- **General cargo**: Standard goods with no special handling requirements.
- **Special cargo**: Goods requiring specific packaging, labeling, documentation, and handling:
  - Dangerous goods (DGR -- 9 hazard classes)
  - Live animals (LAR)
  - Perishable cargo (time-and-temperature-sensitive)
  - Valuable cargo (VAL)
  - Vulnerable cargo (VUN)
  - Heavy/outsized cargo
  - Human remains (HUM)
  - Wet cargo

### WCO Harmonized System (HS) Relationship

The WCO Harmonized System classifies **goods** (what is inside the handling unit), not the handling unit itself. However:
- Customs declarations reference **both** the HS code for the goods AND the package type code for the handling unit.
- The HS code determines duties and regulatory requirements; the package type code determines handling and inspection procedures.
- IATA and WCO collaborate to ensure standardized HS coding facilitates customs clearance for air cargo.

---

## 4. SAP/ERP Handling Unit Data Model

### Overview

SAP's Handling Unit Management (HUM) is part of the Logistics General (LO-HU) module. It provides a comprehensive data model for tracking physical packaging from warehouse to delivery.

### Core Database Tables

**VEKP -- Handling Unit Header Table:**

| Field | Description | Data Type |
|-------|-------------|-----------|
| **VENUM** | Internal HU Number (system-generated primary key) | NUMC 10 |
| **EXIDV** | External HU Identification (scannable number, e.g., SSCC) | CHAR 20 |
| **EXIDA** | Type of External HU Identifier | CHAR 2 |
| **VHILM** | Packaging Material (material number from material master) | CHAR 18 |
| **VHILM_KU** | Customer description of the packaging material | CHAR 22 |
| **VPOBJ** | Packing Object Type ('03'=delivery, '12'=non-assigned HU) | CHAR 2 |
| **VPOBJKEY** | Packing Object Key (e.g., delivery number) | CHAR 20 |
| **BRGEW** | Gross Weight | QUAN |
| **NTGEW** | Net Weight | QUAN |
| **TARAG** | Tare Weight | QUAN |
| **GEWEI** | Weight Unit | UNIT 3 |
| **LAENG** | Length | QUAN |
| **BREIT** | Width | QUAN |
| **HOEHE** | Height | QUAN |
| **MEABM** | Unit of Dimension | UNIT 3 |
| **LGNUM** | Warehouse Number | CHAR 3 |
| **LGTYP** | Storage Type | CHAR 3 |
| **LGPLA** | Storage Bin | CHAR 10 |

**VEPO -- Handling Unit Item (Contents) Table:**

| Field | Description | Data Type |
|-------|-------------|-----------|
| **VENUM** | Internal HU Number (FK to VEKP) | NUMC 10 |
| **VEPOS** | HU Item Number (line item within the HU) | NUMC 6 |
| **VELIN** | Type of HU Item Content (1=delivery item, 2=nested HU) | CHAR 1 |
| **VBELN** | Delivery Number | CHAR 10 |
| **POSNR** | Delivery Item Number | NUMC 6 |
| **MATNR** | Material Number | CHAR 18 |
| **VEMNG** | Packed Quantity | QUAN |
| **VEMEH** | Unit of Measure | UNIT 3 |
| **UNVEL** | Lower-Level HU Number (child HU VENUM when VELIN=2) | NUMC 10 |

### Nesting / Parent-Child Hierarchy

The multi-level packaging hierarchy works through the interplay of VEKP and VEPO:

1. Each handling unit has **one header record in VEKP** with a unique VENUM.
2. Its contents are stored as **line items in VEPO**, linked by VENUM.
3. The **VELIN** field in VEPO determines the content type:
   - **VELIN = 1**: The line is a delivery item (actual goods/material).
   - **VELIN = 2**: The line is a nested child handling unit.
4. When VELIN = 2, the **UNVEL** field contains the VENUM of the child HU, which itself has its own header in VEKP and potentially its own VEPO items.

**Example hierarchy:**

```
Pallet (VEKP: VENUM=100, EXIDV="PALLET001", VHILM="EUR_PALLET")
├── VEPO: VENUM=100, VEPOS=1, VELIN=2, UNVEL=200  --> Box A
│     └── VEKP: VENUM=200, EXIDV="BOX_A_001", VHILM="CARDBOARD_BOX"
│           ├── VEPO: VENUM=200, VEPOS=1, VELIN=1, MATNR="WIDGET_X", VEMNG=50
│           └── VEPO: VENUM=200, VEPOS=2, VELIN=1, MATNR="WIDGET_Y", VEMNG=30
├── VEPO: VENUM=100, VEPOS=2, VELIN=2, UNVEL=300  --> Box B
│     └── VEKP: VENUM=300, EXIDV="BOX_B_001", VHILM="CARDBOARD_BOX"
│           └── VEPO: VENUM=300, VEPOS=1, VELIN=1, MATNR="GADGET_Z", VEMNG=100
└── VEPO: VENUM=100, VEPOS=3, VELIN=2, UNVEL=400  --> Box C
      └── VEKP: VENUM=400, EXIDV="BOX_C_001", VHILM="WOODEN_CRATE"
            └── VEPO: VENUM=400, VEPOS=1, VELIN=1, MATNR="PART_Q", VEMNG=25
```

### Supporting Configuration Tables

| Table | Description |
|-------|-------------|
| **TVTY** | Packaging Material Types |
| **TVTYT** | Packaging Material Types: Descriptions |
| **TVEGR** | Material Group: Packaging Materials |
| **TERVH** | Allowed Packaging Materials for each Material Group |
| **HUSSTAT** | Individual Status for Each Handling Unit |
| **VEKPVB** | Work Structure for Handling Unit Header |
| **VEKPD** | Dynamic Section of Handling Unit Header |

### Labels

- **Internal HU labels**: Used when HUs remain within the same organization (e.g., transfers within company codes or plants). No relabeling needed.
- **External labels**: Required when goods are transacted with external parties (vendors, customers). Must be relabeled to the receiving party's system.

### Integration with Other SAP Modules

- **SD (Sales & Distribution)**: HUs are packed during outbound delivery processing.
- **MM (Materials Management)**: HUs are created during inbound goods receipt.
- **EWM (Extended Warehouse Management)**: HU types in EWM must mirror ERP configuration; if they differ, HU creation in EWM fails.
- **TM (Transportation Management)**: HUs in deliveries generate freight units for transportation planning. All items in one HU stay in one freight unit.
- **Inventory Management**: HU-managed storage locations ensure total HU stock equals stock in the IM storage location.

### Software Modeling Implications

For a customs clearance system, the SAP model teaches us:
1. **Separate header from contents** -- a handling unit has header data (packaging, dimensions, weight) and content data (what's inside).
2. **Support recursive nesting** -- HUs inside HUs is a fundamental requirement.
3. **Multiple identifier types** -- internal system IDs vs. external scannable IDs (SSCC, carrier tracking numbers).
4. **Link to transport documents** -- HUs must be traceable to delivery/shipment documents.
5. **Packaging material as master data** -- define reusable packaging types with standard dimensions, weights, and capacities.

---

## 5. Consolidation (Cons) in Logistics

### What Is a Consolidation?

A **consolidation** (frequently abbreviated to **"consol"** or **"cons"** in freight forwarding operations) is the operational process of combining multiple individual shipments from different shippers into a single, larger transport unit for more efficient and cost-effective transportation.

**Important distinction:** The term "consolidation" in operational logistics has a specific, narrow meaning that differs from the broader concept of "shipment consolidation":

| Concept | Meaning |
|---------|---------|
| **Operational Consolidation (Consol/Cons)** | The specific act of physically grouping multiple shipments into one transport unit (one ULD, one container, one truck) under a single master transport document, managed by a freight forwarder or NVOCC. |
| **Shipment Consolidation (broader)** | Any strategy to combine shipments for efficiency -- including buyer's consolidation, warehouse consolidation, cross-docking, multi-stop routing, etc. |

### The Cons Number

A **"cons number"** (consolidation number) is a freight forwarder's **internal reference number** used to identify and manage a specific consolidation booking. Key facts:
- There is **no international standard** for cons number formats. Each forwarder uses their own proprietary numbering system.
- The cons number ties together all the individual House Waybills (HAWBs/HBLs) and the Master Waybill (MAWB/MBL) under one operational grouping.
- It is purely an internal operational reference -- shippers and consignees typically interact with the HAWB/HBL number, not the cons number.

### How Consolidation Works

**Step-by-step process:**

1. **Collection**: The freight forwarder collects individual shipments from multiple shippers at their warehouse or receives them at a Container Freight Station (CFS).
2. **Documentation**: Each individual shipment gets a **House Waybill** (HAWB for air, HBL for ocean) -- a contract between the forwarder and the individual shipper.
3. **Grouping**: Compatible cargo (same destination region, compatible goods types, timing alignment) is grouped together.
4. **Physical loading**: The grouped shipments are physically loaded into a single transport unit (ULD for air, container for ocean, pallet/truck for ground).
5. **Master document**: The forwarder obtains a **Master Waybill** (MAWB for air, MBL for ocean) from the carrier -- a contract between the forwarder and the carrier covering the entire consolidated load.
6. **Transit**: The consolidated unit moves as a single entity through the transport network.
7. **Deconsolidation (break-bulk)**: At destination, the consolidated unit is broken down. The forwarder or their agent separates individual shipments per HAWB/HBL.
8. **Final delivery**: Individual shipments are customs-cleared and delivered to their respective consignees.

### Types of Consolidation by Mode

#### Air Freight Consolidation

- **Transport unit**: ULD (Unit Load Device) or loose cargo.
- **Documentation**: Master Air Waybill (MAWB) covers the entire consolidation. Individual House Air Waybills (HAWBs) cover each shipper's cargo.
- **MAWB description**: Must state "CONSOL" or "CONSOLIDATION" (not "CNSL") in the goods description.
- **Piece reporting**: The MAWB must report the total number of inner pieces (the smallest external packaging unit). Example: 1 pallet loaded with 100 boxes = quantity of 100.
- **ULD booking vs. loose**: A forwarder can book a ULD (putting all consol pieces into one container) or tender pieces "loose" for the airline to containerize.
- **Cost savings**: 30-50% compared to individual air freight shipments.
- **Electronic messaging**: In FWB (Freight Waybill message), NC is the goods data identifier for consolidation; NG is for non-consolidated shipments.

#### Ocean Freight Consolidation (LCL)

- **Transport unit**: Shipping container (20' or 40').
- **Documentation**: Master Bill of Lading (MBL) for the container. House Bills of Lading (HBLs) for individual shipments.
- **Operator**: Usually an NVOCC (Non-Vessel Operating Common Carrier) or freight forwarder.
- **Physical location**: Consolidation happens at a **Container Freight Station (CFS)** -- a bonded warehouse near the port.
- **Process**: Shipments arrive separately at the CFS, are inspected and documented, then loaded ("stuffed") into the container.
- **Deconsolidation**: At destination, the container goes to a CFS where it is "de-stuffed" and individual shipments are separated, inspected, and cleared.

#### Ground/Road Freight Consolidation (LTL)

- **Transport unit**: Truck trailer, pallet positions.
- **Documentation**: Master Bill of Lading or carrier's freight bill.
- **Common term**: Groupage (particularly in Europe).
- **Process**: LTL (Less-than-Truckload) shipments are collected at a distribution center, sorted by destination, loaded onto shared trailers.

### Consolidation Documentation Hierarchy

```
Master Transport Document (MAWB / MBL / Master BOL)
  ├── House Waybill #1 (HAWB / HBL) -- Shipper A
  │     ├── Commercial Invoice
  │     ├── Packing List
  │     └── Certificate of Origin
  ├── House Waybill #2 (HAWB / HBL) -- Shipper B
  │     ├── Commercial Invoice
  │     └── Packing List
  └── House Waybill #3 (HAWB / HBL) -- Shipper C
        ├── Commercial Invoice
        ├── Packing List
        └── Phytosanitary Certificate

Consolidation Manifest / FHL (Consolidation List Message)
  └── Check-list of all HAWBs under the MAWB
```

**FHL (Consolidation List Message)**: A standard CargoIMP message that provides a checklist of all HAWBs associated with a MAWB. It is the digital manifest of a consolidation.

### MAWB vs. HAWB -- Key Differences

| Feature | MAWB (Master Air Waybill) | HAWB (House Air Waybill) |
|---------|---------------------------|--------------------------|
| **Issued by** | Airline or its agent | Freight forwarder |
| **Contract between** | Forwarder and airline | Forwarder and shipper |
| **Scope** | Entire consolidated shipment | Individual shipment within consolidation |
| **Goods description** | "CONSOLIDATION" or "CONSOL" | Precise cargo description required |
| **Customs** | Referenced for overall shipment | Usually required for customs clearance of individual shipments |
| **Number format** | 3-digit airline prefix + 8-digit number (last digit is check digit) | Forwarder's proprietary format |

**Rule**: Even if a MAWB has only one associated HAWB, both must still be reported separately to customs (per US AMS requirements).

**Weight rule**: Total HAWB weight and MAWB weight must be identical.

---

## 6. ULD (Unit Load Device) Specifics

### What Is a ULD?

A **Unit Load Device (ULD)** is a standardized container or pallet used in air cargo transportation to efficiently consolidate, protect, and transport freight, baggage, and mail. ULDs are designed to lock into aircraft cargo holds, enabling rapid loading/unloading and ensuring cargo is secured during flight.

- Approximately **1 million ULDs** are in service worldwide, worth over **USD 1 billion**.
- ULDs are classified as **removable aircraft parts** -- they are safety-critical aviation equipment.
- The industry loses approximately **USD 400 million annually** on ULD repair and replacement.

### Types of ULDs

ULDs come in two main categories: **containers** (enclosed) and **pallets** (flat platforms with nets).

| ATA Code | IATA Code | Type | Description | Common Aircraft |
|----------|-----------|------|-------------|-----------------|
| **LD3** | AKE / AKN | Container | Most common ULD. Half-width lower deck container. | 787, 777, 747, A330/340/350, MD-11 |
| **LD3-45** | AKH | Container | Temperature-controlled LD3 variant | Wide-body aircraft |
| **LD1** | AKC | Container | Narrow lower deck container | Wide-body aircraft |
| **LD6** | ALF | Container | Double-width (2x LD3) lower deck | 777, 747, A380 |
| **LD7** | P1P / xAK | Pallet+Net | Double-width flat pallet with net | 787, 777, 747, wide-body Airbus |
| **LD11** | ALP | Container | Full-width lower deck container | Larger wide-body aircraft |
| **PMC** | P6P | Pallet | Universal general-purpose flat pallet, 96"x125" | Lower hold and main decks |

**ATA vs. IATA codes**: The ATA (Air Transport Association) codes (LD3, LD7, etc.) are legacy codes still commonly used colloquially. The IATA coding system is the current international standard.

### ULD Identification Numbers

The **IATA ULD Identification Code** is the standard format:

```
[Type Prefix (3 letters)] [Serial Number (4-5 digits)] [Owner Code (2 chars)]
```

Example: **AKN 12345 DL**
- **AKN**: Forkliftable LD3 container
- **12345**: Unique serial number
- **DL**: Delta Air Lines (owner)

**Type prefix structure:**
- 1st letter: Container type (A=container, P=pallet, R=cool container)
- 2nd letter: Base dimensions
- 3rd letter: Contour/fork-lifting capability

**Spacing rule**: On containers and pallets, a single blank space between positions 3-4 and positions 8-9 (e.g., AKE 12345 ZZ or PMC 12345 ZZ).

**Certified vs. non-certified ULDs:**
- Identified by the first letter of the code.
- **Certified ULDs** must be used in aircraft holds with insufficient structural strength to contain cargo during extreme flight conditions (approved under TSO-C90, ETSO-C90, or CTSO-C90).
- **Non-certified ULDs** may be used in structurally capable holds.

### ULD Tracking Technologies

- **Bar codes**: Most ULDs carry barcodes replicating the IATA standard identification code.
- **RFID tags**: Increasingly integrated for real-time tracking.
- **IoT sensors**: Emerging technology for temperature, humidity, and shock monitoring.
- **GPS tracking**: Some specialized ULDs carry GPS for high-value or sensitive cargo.

### ULD Manifest and Documentation

Each ULD has its own **packing list/manifest** that inventories its contents:
- The manifest lists all shipments (by HAWB or MAWB reference) loaded into the ULD.
- It includes piece counts, weights, and goods descriptions.
- This speeds up tracking and customs processing since each ULD travels under one ID number and manifest.
- All required documents must be available: "manifest (list) of consolidation, invoices, packing lists for personal belongings, other documents required for export, import or transit by customs."

### ULD Build-Up and Breakdown Operations

**Build-up** (loading cargo into/onto ULD):
- Most manpower-intensive operation at air cargo terminals.
- Remains almost entirely manual despite technological advances since the 1970s.
- Shipments must typically be tendered to the airline **6 hours before departure**.
- Loading rules: heavier items at bottom, lighter items at top; items must be interlocked or secured with ropes/straps.
- Every ULD must observe contour limitations -- minimum **50mm (2 inches)** air space between ULD and aircraft hold liner (for fire suppression airflow).
- Every ULD must comply with the aircraft's Weight and Balance Manual (WBM).

**Breakdown** (unloading cargo from ULD):
- Occurs at destination cargo terminal.
- Consolidated cargo is separated per HAWB for individual customs clearance.
- Airline or contracted ground handling agent performs the break-bulk.

### ULD Customs Interactions

**How customs interacts with ULDs:**

1. **Pre-arrival screening**: Cargo information (MAWB, HAWB data) is submitted electronically to customs (e.g., US Air AMS, ACAS -- Air Cargo Advance Screening) **before** the aircraft arrives.

2. **Risk targeting**: Customs uses automated systems (e.g., CBP's Automated Targeting System) to flag high-risk shipments based on HAWB-level data.

3. **Hold on HAWB vs. hold on ULD**: This is critical --
   - If customs places a hold on **one HAWB**, the **entire MAWB** may not be released for uplift.
   - If a MAWB/HAWB is co-loaded on a ULD with other MAWBs, **the entire ULD** can be held from uplift.
   - This means one flagged shipment can delay all other shipments in the same ULD.

4. **Examination process**: If inspection is required, the ULD must be broken down so individual shipments can be examined. The examination may range from document review to full physical inspection.

5. **ACAS compliance**: US CBP requires precise cargo descriptions in advance data. Vague descriptions ("gift," "daily necessities," "accessories," "parts," "consolidated") are no longer accepted and will trigger holds.

---

## 7. Container Consolidation for Ocean Freight

### How LCL (Less-than-Container Load) Works

LCL allows shippers without enough cargo to fill an entire container to share container space with other shippers, splitting costs proportionally.

**The LCL Process Flow:**

```
Origin CFS (Container Freight Station)
  │
  ├── Receive individual shipments from multiple shippers
  ├── Inspect and verify documentation for each shipment
  ├── Assign House Bills of Lading (HBLs) to each shipment
  ├── Group compatible shipments by destination
  ├── "Stuff" (load) grouped shipments into a single container
  ├── Seal the container
  └── Issue Master Bill of Lading (MBL) for the full container
        │
        ▼
Ocean Transit (container moves as FCL to the carrier)
        │
        ▼
Destination Port → Container Yard
        │
        ▼
Destination CFS
  │
  ├── Break container seal (in presence of customs agents)
  ├── "De-stuff" (unload) the container
  ├── Separate individual shipments per HBL
  ├── Each shipment undergoes customs inspection and clearance
  ├── Importer files Bill of Entry for their goods
  ├── Customs validates, duty is paid
  ├── Customs issues import pass / gate pass
  └── Individual shipments released for final delivery
```

### Container Manifest Requirements

**Master Bill of Lading (MBL):**
- Issued by the ocean carrier (shipping line).
- Covers the entire container.
- Lists the NVOCC/forwarder as shipper (not individual shippers).
- Container number, seal number, vessel, voyage, port of loading, port of discharge.

**House Bills of Lading (HBLs):**
- Issued by the NVOCC or freight forwarder.
- One per individual shipper/consignment.
- Contains detailed cargo description, weight, piece count, value.
- Required for individual customs clearance at destination.

**Automated Manifest System (AMS) / eManifest:**
- Both MBL and HBL data must be submitted electronically to customs before vessel arrival (e.g., US requires 24-hour advance manifest rule for ocean cargo).
- Container contents are declared at the HBL level with detailed descriptions.

**Permit to Transfer (PTT):**
- In the US, an NVO (Non-Vessel Operating Common Carrier) obtains a PTT against the MBL to move cargo from the MBL destination to a deconsolidation warehouse (CFS).
- The PTT authorizes the physical movement of the sealed container from the terminal to the CFS.

### Container Customs Inspection

**How customs inspects consolidated containers:**

1. **VACIS/NII (Non-Intrusive Inspection):**
   - The sealed container is scanned via gamma-ray imaging (X-ray) at the port.
   - The container seal is **not** broken.
   - If the scan looks normal, the container is released.
   - Costs are spread across all cargo owners in the container.

2. **Tailgate Exam:**
   - If VACIS results are questionable, CBP breaks the seal and opens the container doors for visual inspection.
   - Officers look inside without fully unloading.
   - If nothing suspicious, the container is released.

3. **Intensive Exam (Strip/Full Devanning):**
   - The entire container is moved to a **Centralized Examination Station (CES)** -- a privately operated facility contracted by customs (see 19 CFR Part 118).
   - The container is completely unloaded (devanned).
   - Shipments are separated by parcels and boxes for piece-by-piece physical inspection.
   - CBP officers inspect with narcotics dogs, X-ray equipment, and manual review.
   - Complete devanning can cost several hundred dollars plus storage, transport, and reloading costs.
   - The importer bears these costs (19 CFR 151.6, 19 USC 1467).

### Impact on Other Shipments in a Consolidated Container

This is one of the most critical operational risks of LCL consolidation:

- **If any shipment in the container triggers a customs hold, the entire container can be held.** All other shipments inside are delayed regardless of their compliance status.
- **Customs does not reveal** which specific HAWB/HBL triggered the exam. Costs are split among all cargo owners "for security purposes."
- **Higher inspection risk**: LCL containers inherently face higher inspection risk than FCL because they contain more diverse products from more shippers. More product types = more reasons for customs to flag the container.
- **Deconsolidation delay**: LCL cargo typically takes **5-7 days** for deconsolidation and clearance at destination, compared to **2-3 days** for FCL.
- **Documentation cascade**: One shipper's documentation problems or prohibited items can delay the entire container, affecting all consolidated shipments.
- **Discrepancy reporting**: During de-stuffing, the CFS warehouse may discover piece count discrepancies on an HBL and prepare a Manifest Discrepancy Report (MDR), which is sent to the NVO.
- **Liability**: Manifest discrepancy liability belongs to the Master of the vessel or their agent (19 CFR 4.12). Once cargo leaves the terminal, liability falls on the bonded carrier on behalf of the NVOCC.

**Mitigation strategies:**
- Work with reputable freight forwarders (customs authorities favor established consolidators with consistent compliance records, reducing inspection rates).
- Ensure accurate, complete documentation for every HAWB/HBL.
- Build buffer time (5-7 days for LCL clearance) into supply chain schedules.
- Consider FCL for time-sensitive or high-value shipments despite higher per-unit costs.

---

## 8. Customs and Handling Units -- Regulatory Framework

### CBP Authority to Examine

**Legal basis (United States):**

| Regulation | Subject |
|------------|---------|
| **19 USC 1467** | CBP's right to examine any imported shipment |
| **19 CFR Part 151** | Examination, sampling, and testing of merchandise |
| **19 CFR 151.6** | Importer bears expense of examination |
| **19 CFR Part 118** | Centralized Examination Stations (CES) |
| **19 CFR Part 141** | Entry of merchandise |
| **19 CFR Part 142** | Entry process (including consolidated entries) |
| **19 CFR 10.153** | Consolidated shipments treated as one importation |
| **19 CFR 122.48** | Air cargo manifest requirements |
| **19 CFR 4.12** | Manifest discrepancy liability |
| **CBP Directive 3290-007B** | National Maritime Targeting Policy |

### Minimum Examination Requirements

Under 19 CFR Part 151:
- Not less than **1 package in every 10 packages** shall be examined.
- Special authorization allows less than 1-in-10 but **not less than 1 package per invoice**, for merchandise in uniform packages.
- The port director determines the number of packages to examine for duty determination and compliance.

### Examination Types and Their Impact on Handling Units

**1. VACIS / NII (Non-Intrusive Inspection)**
- X-ray/gamma-ray scan of the sealed container or ULD.
- Handling units are NOT individually examined.
- Container/ULD seal is NOT broken.
- Fastest exam type; minimal delay.
- Cost shared across all parties in a consolidated shipment.

**2. Tailgate Examination**
- Container/ULD is opened and visually inspected.
- Some outer handling units may be inspected without full unloading.
- Moderate delay and cost.

**3. Intensive / CET / A-TCET Examination**
- Full physical examination at a Centralized Examination Station (CES).
- The container/ULD is **completely emptied**.
- Every handling unit is **separated, staged, and individually inspected**.
- May involve piece-by-piece examination, narcotics dogs, and X-ray of individual packages.
- This is the most disruptive exam type. Complete devanning costs several hundred dollars per container plus storage, transport, and reloading.
- The importer must "bear any expense involved in preparing the merchandise for shipment customs examination and in the closing of packages" (19 CFR 151.6).

### When Customs Targets a Handling Unit Inside a Consolidation

**What happens operationally:**

1. **Targeting at the document level**: Customs flags a specific HAWB or HBL (not a physical handling unit) based on advance manifest data, risk assessment, intelligence, or random selection.

2. **Impact on the consolidation unit**:
   - For **air cargo**: If a HAWB is flagged, the entire MAWB may be held. If the MAWB is on a ULD with other MAWBs, the entire ULD can be held.
   - For **ocean cargo**: If an HBL triggers a hold, the entire container (all HBLs under the MBL) is typically held for examination.

3. **Examination process**:
   - For an intensive exam, the entire consolidation unit (ULD or container) is stripped/devanned at a CES.
   - Customs then examines only the targeted shipment(s) but the physical process of unloading affects everything.
   - For VACIS, only the sealed unit is scanned; no individual handling units are disturbed.

4. **Costs and liability**:
   - Examination costs are borne by the importer of the targeted shipment.
   - However, in consolidations, costs may be spread across all parties (particularly for VACIS and container transport to/from CES).
   - Customs does not disclose which specific HAWB triggered the exam.
   - Delays affect all shipments in the consolidation, not just the targeted one.

5. **Regulatory basis for inspecting entire consolidation units**:
   - 19 USC 1467 gives CBP the unrestricted right to examine any imported shipment.
   - There is no legal limitation requiring CBP to examine only the flagged shipment within a consolidation -- they have authority over the entire load.
   - The National Maritime Targeting Policy (CBP Directive 3290-007B) and Non-Intrusive Inspection Directive provide the operational framework.
   - For air cargo, the Air Cargo Advance Screening (ACAS) program mandates pre-arrival data submission that drives targeting decisions.

### Consolidated Entry Summaries

Under 19 CFR Part 142, for consolidated entries:
- The broker must **list separately** on the face of the consolidated entry summary the merchandise for each ultimate consignee.
- Each must include appropriate entry or special permit numbers.
- Under 19 CFR 10.153, consolidated shipments addressed to one consignee are treated as one importation.

### Record Retention

CBP requires importers to maintain entry documentation for **up to 5 years** from date of entry (19 CFR 163.4). This applies to all documentation related to examined shipments, including handling unit records.

---

## 9. Software Modeling Implications for Customs Clearance Systems

Based on the research above, a customs clearance software system should model the following entities and relationships:

### Core Entity Model

```
Shipment (transport-level)
  ├── Master Transport Document (MAWB / MBL / Master BOL)
  │     ├── House Transport Document #1 (HAWB / HBL)
  │     │     ├── Commercial Invoice
  │     │     ├── Packing List
  │     │     └── Customs Entry
  │     ├── House Transport Document #2
  │     └── House Transport Document #3
  ├── Consolidation (cons) [optional]
  │     ├── Cons Number (internal forwarder reference)
  │     ├── Consolidation Manifest (FHL for air)
  │     └── Link to all HAWBs/HBLs in the consol
  └── Physical Cargo Hierarchy
        ├── Consolidation Unit (ULD / Container / Pallet)
        │     ├── ULD ID / Container Number
        │     ├── ULD/Container Manifest
        │     └── Handling Units
        │           ├── Handling Unit #1 (linked to HAWB #1)
        │           │     ├── HU Type (pallet, carton, crate, drum, etc.)
        │           │     ├── Identifiers (SSCC, carrier tracking #, internal HU #)
        │           │     ├── Dimensions (L x W x H, weight)
        │           │     ├── Contents (nested HUs or trade items)
        │           │     └── Package Type Code (UNECE Rec 21)
        │           ├── Handling Unit #2 (linked to HAWB #2)
        │           └── Handling Unit #3 (linked to HAWB #2)
        └── Examination Events
              ├── Exam Type (VACIS, tailgate, intensive)
              ├── CES Location
              ├── Affected HAWBs/HBLs
              ├── Status (pending, in progress, released)
              └── Cost allocation
```

### Key Relationships

1. **Shipment to Handling Units**: One-to-many. A shipment has one or more handling units.
2. **Handling Unit to Handling Unit**: Recursive/hierarchical. HUs can contain other HUs (nesting).
3. **Handling Unit to House Waybill**: Many-to-one. Multiple HUs can belong to the same HAWB/HBL.
4. **House Waybill to Master Waybill**: Many-to-one. Multiple HAWBs under one MAWB (consolidation).
5. **Handling Unit to Consolidation Unit**: Many-to-one. Multiple HUs are loaded into one ULD/container.
6. **Consolidation Unit to Examination**: One-to-many. An exam event affects the entire ULD/container.
7. **Handling Unit to Customs Entry**: Many-to-one through the HAWB/HBL. Entry is filed at the HAWB/HBL level, covering all HUs under that document.

### Identifier Registry

The system should maintain a flexible identifier registry:

| Identifier Type | Format | Scope | Example |
|----------------|--------|-------|---------|
| SSCC | 18-digit (GS1) | Universal | 00 0 0614141 000000001 8 |
| Carrier Tracking # | Varies by carrier | Carrier-specific | 1Z999AA10123456784 |
| ULD ID | 3-letter + 5-digit + 2-char | IATA standard | AKE 12345 DL |
| Container # | ISO 6346 format | Universal | MSKU1234567 |
| MAWB # | 3-digit airline + 8-digit | IATA standard | 176-12345675 |
| HAWB # | Forwarder proprietary | Forwarder-specific | KN123456789 |
| MBL # | Carrier proprietary | Carrier-specific | MAEU123456789 |
| HBL # | NVOCC proprietary | NVOCC-specific | FF-HBL-2026-001 |
| Cons # | Forwarder proprietary | Internal | CONSOL-LAX-2026-0042 |
| Entry # | Government format | Country-specific | 123-1234567-8 (US) |

---

## Sources

### Handling Units -- Definition and Hierarchy
- [FreightCenter: Handling Unit](https://www.freightcenter.com/help/glossary/handling-unit/)
- [prologistik: What is a handling unit?](https://www.prologistik.com/en/logistics-lexicon/handling-unit-hu/)
- [REMIRA: Handling Unit](https://www.remira.com/en/glossary/handling-unit)
- [Interwf: What is a Handling Unit?](https://interwf.com/freight-glossary/handling-unit/)

### GS1 / SSCC Standards
- [GS1: Serial Shipping Container Code (SSCC)](https://www.gs1.org/standards/id-keys/sscc)
- [GS1 US: Serialized Shipping Container Codes](https://www.gs1us.org/upcs-barcodes-prefixes/serialized-shipping-container-codes)
- [GS1: Logistic Label Guideline](https://www.gs1.org/standards/gs1-logistic-label-guideline/1-3)
- [GS1 US: Identify Logistic Units with SSCC](https://www.gs1us.org/resources/data-hub-help-center/about-the-serial-shipping-container-code-sscc)
- [Commport: Complete Guide to Shipping Labels](https://www.commport.com/shipping-labels/)

### SAP Handling Unit Data Model
- [SAP Community: Handling Unit in EWM](https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-members/handling-unit-in-ewm-configuration-ewm-blog-4/ba-p/13772434)
- [SAP Learning: Creating Handling Units](https://learning.sap.com/courses/planning-and-execution-in-sap-s-4hana-transportation-management/creating-handling-units)
- [SAPStack: VEKP Table](https://sapstack.com/tables/tabledetails.php?table=VEKP)
- [SAPStack: VEPO Table](https://sapstack.com/tables/tabledetails.php?table=VEPO)
- [SAP Datasheet: VEKP](https://www.sapdatasheet.org/abap/tabl/vekp.html)
- [SAP Datasheet: VEPO](https://www.sapdatasheet.org/abap/tabl/vepo.html)
- [WeTechIdeas: Handling Unit Quick Guide](https://wetechideas.com/2022/12/03/handling-unit/)
- [se80.co.uk: SAP LO-HU Tables](https://www.se80.co.uk/sapmodules/l/lo-h/lo-hu-tables-all.htm)

### ULD (Unit Load Device)
- [ULD CARE: Build-up & Breakdown](https://www.uldcare.com/uld-explained-book/buildupbreakdown/)
- [ULD CARE: ULD Identification Codes](https://www.uldcare.com/articles/library/care/uld-identification-codes-demystified/)
- [IATA: Unit Load Devices](https://www.iata.org/en/programs/cargo/cargo-operations/unit-load-devices/)
- [IATA: ULD Regulations (ULDR)](https://www.iata.org/en/publications/manuals/uld-regulations/)
- [VRR Aero: ULD Identification](https://vrr.aero/knowledge-center/uld-info/uld-id-code/)
- [VRR Aero: ATA/IATA Conversion Table](https://vrr.aero/knowledge-center/uld-info/conversion-table/)
- [SKYbrary: Unit Load Devices](https://skybrary.aero/articles/unit-load-devices-uld)
- [Wikipedia: Unit Load Device](https://en.wikipedia.org/wiki/Unit_load_device)
- [Dimerco: Building Airline ULDs](https://dimerco.com/resources/building-airline-uld-need-to-know/)

### Consolidation
- [DHL Freight Connections: Consolidation](https://dhl-freight-connections.com/en/logistics-dictionary/consolidation/)
- [Logos 3PL: Consolidation](https://www.logos3pl.com/glossary/consolidation/)
- [Lotus Containers: Consolidation in Logistics](https://www.lotus-containers.com/en/consolidation-in-logistics/)
- [Supath: Air Freight Consolidation](https://www.supath.net/air-freight-consolidation/)
- [Ship4wd: MAWB vs HAWB](https://ship4wd.com/import-guides/mawb-vs-hawb)
- [GoCargoNet: MAWB vs HAWB](https://www.gocargonet.com/master-air-waybill-mawb-vs-house-air-waybill-hawb/)
- [Emotrans: Airway Bill Guide](https://www.emotrans-global.com/blog/airway-bill-guide/)

### Ocean Freight / CFS / LCL
- [Freightos: LCL Shipping Guide](https://www.freightos.com/freight-resources/what-is-lcl-shipping-the-complete-guide/)
- [Ship4wd: Deconsolidation](https://ship4wd.com/resource-center/glossary/deconsolidation)
- [Marine Insight: Container Freight Stations](https://www.marineinsight.com/maritime-law/understanding-container-freight-stations-purpose-location-and-types/)
- [Red Stag Fulfillment: LCL Meaning](https://redstagfulfillment.com/lcl-meaning/)
- [CBP: Ocean House Bill of Lading FAQ](https://www.cbp.gov/trade/automated/ohbol-faq)

### Customs Examination and Regulatory
- [CBP: Cargo Examination](https://www.cbp.gov/border-security/ports-entry/cargo-security/examination)
- [eCFR: 19 CFR Part 151 -- Examination](https://www.ecfr.gov/current/title-19/chapter-I/part-151)
- [eCFR: 19 CFR Part 118 -- Centralized Examination Stations](https://www.ecfr.gov/current/title-19/chapter-I/part-118)
- [eCFR: 19 CFR 122.48 -- Air Cargo Manifest](https://www.ecfr.gov/current/title-19/chapter-I/part-122/subpart-E/section-122.48)
- [Shapiro: Customs Inspection Behind-the-Scenes](https://www.shapiro.com/examining-the-issue-customs-inspection/)
- [Ship4wd: Customs Holds & Exams](https://ship4wd.com/import-guides/customs-holds-customs-exams)
- [iContainers: Types of US Customs Inspections](https://www.icontainers.com/help/types-of-us-customs-inspections-and-holds/)
- [DHS OIG: Cargo Targeting and Examinations](https://www.oig.dhs.gov/sites/default/files/assets/Mgmt/OIG_10-34_Jan10.pdf)
- [GAO: Cargo Container Inspections](https://www.govinfo.gov/content/pkg/GAOREPORTS-GAO-06-591T/html/GAOREPORTS-GAO-06-591T.htm)
- [Air Cargo News: US Customs Clamps Down on Vague Descriptions](https://www.aircargonews.net/us-customs-clamps-down-on-vague-cargo-descriptions/1073613.article)

### Package Type Codes
- [UNECE Recommendation No. 21](https://unece.org/sites/default/files/2023-10/rec21rev4_ecetrd309.pdf)
- [Sharps Container: UN Packaging Code Guide](https://sharpsvillecontainer.com/blog/understanding-un-packaging-codes/)
- [ISO 780:2015](https://www.iso.org/standard/59933.html)

### Carrier Tracking
- [FedEx Developer Portal: Freight LTL API](https://developer.fedex.com/api/en-bo/catalog/ltl-freight/docs.html)
- [Kuehne+Nagel: Public Tracking](https://mykn.kuehne-nagel.com/public-tracking/)
- [Wikipedia: Tracking Number](https://en.wikipedia.org/wiki/Tracking_number)

### Freight Forwarding Software
- [CargoWise: International Forwarding Solutions](https://www.cargowise.com/solutions/cargowise-forwarding/)
- [CargoWise: Container Automation](https://www.cargowise.com/solutions/cargowise-forwarding/ocean/container-automation/)
