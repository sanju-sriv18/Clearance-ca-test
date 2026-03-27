# Ocean Container Consolidation, Handling Units, and Customs Clearance

## 1. Ocean Cargo Handling Unit Hierarchy

### 1.1 The Hierarchy: Item → Handling Unit → Shipment → Container → Vessel

Ocean cargo follows a multi-level hierarchy, particularly important for LCL (Less than Container Load) shipments:

```
VESSEL (carrying thousands of containers)
  └── CONTAINER (20' or 40' steel box, identified by ISO container number + seal number)
        └── SHIPMENT / CONSIGNMENT (identified by House Bill of Lading - HBL)
              └── HANDLING UNIT (pallet, crate, carton, drum, bundle)
                    └── ITEM / PIECE (individual product units within handling units)
```

**For FCL (Full Container Load):**
- One shipper → one container → one Bill of Lading
- The hierarchy is simpler: the container IS the shipment unit

**For LCL (Less than Container Load):**
- Multiple shippers → consolidated into one container → one Master Bill of Lading (MBL) with multiple House Bills of Lading (HBLs)
- Each HBL represents one consignment from one shipper to one consignee
- Within each consignment, there are handling units (pallets, cartons, crates)

### 1.2 Handling Unit Identifiers in Ocean Cargo

There are several layers of identification at the handling unit level:

**Shipping Marks (the primary identifier for LCL cargo)**
- A distinctive combination of letters, numbers, and symbols affixed to the exterior of cargo units
- Typically include: consignee initials/code, destination port code, package sequential number (e.g., "1/10" meaning package 1 of 10), and a reference or order number
- For LCL, the shipping mark column on the B/L must cover specific marks used for customs clearance, unpacking, distribution, and receiving
- Shipping marks are THE critical identifier when deconsolidating LCL containers — without them, cargo from different shippers cannot be distinguished

**SSCC (Serial Shipping Container Code)**
- An 18-digit GS1 standard identifier assigned to logistics units (pallets, cartons, cases)
- Encoded in GS1-128 barcodes on GS1 Logistics Labels
- Contains: extension digit + GS1 company prefix + serial reference + check digit
- Acts as a "license plate" for a handling unit through the supply chain
- Fully compatible with ISO/IEC 15459 (unique identifiers for transport units)
- Widely used in retail/distribution supply chains; less universally adopted in ocean freight but growing

**Transport Labels (affixed by consolidator)**
- Applied by the CFS/consolidator during stuffing
- Include: MBL number, HBL number, packing box number, destination CFS code
- Essential for identification during deconsolidation at the destination

**RFID Tags and Barcodes**
- Some cargo uses RFID tags or barcodes containing product information (season, size, color, breakdown of goods)
- Enables faster processing at CFS facilities

### 1.3 Container Content Manifests and Packing Lists

**Packing List (per shipment/HBL)**
- Created by the shipper/exporter for their specific consignment
- Contains: marks and numbers, piece count, dimensions, net and gross weight per package, packing method (crate, box, carton, drum), freight description, handling instructions
- Used by freight forwarders to prepare the B/L, by customs for compliance, by banks for L/C payment, and as evidence in disputes

**Container Manifest / Cargo Manifest**
- A summary document covering ALL cargo in a container
- Contains: container number, seal number, B/L numbers for all consignments, vessel name, ports of origin/destination, cargo descriptions, weights
- Prepared by the carrier (shipping line) based on data from shippers
- Filed electronically with CBP via AMS

**Stuffing Report / Container Load Plan**
- Created at the origin CFS during container stuffing
- Documents: container number, seal number, shipping bill number, number of packages per consignment, net weight, gross weight
- In some jurisdictions, customs officials stamp and sign the stuffing report after inspection
- The load plan shows physical placement of goods within the container for weight distribution and cargo compatibility

## 2. LCL Consolidation Process

### 2.1 CFS (Container Freight Station) Operations — Export/Origin

The consolidation process at an origin CFS follows these steps:

1. **Cargo Receipt**: Individual LCL shipments arrive at the CFS, typically by truck. Each shipment is checked against documentation (commercial invoice, packing list, B/L instructions)

2. **Inspection and Verification**: CFS staff verify cargo condition, quantity, marks and numbers, weight, and dimensions. Discrepancies are noted in a tally report

3. **Consolidation Queue**: Cargo is grouped by destination port and sailing schedule. The CFS consolidates all cargo going to a specific destination into one container

4. **Load Planning**: Professional consolidators plan cargo placement considering:
   - Weight distribution for container stability during ocean transit
   - Cargo compatibility (chemicals separated from foodstuffs, heavy items secured before fragile)
   - Moisture sensitivity
   - Maximum container weight limits
   - Space optimization

5. **Stuffing (Loading)**: Workers follow the load plan, placing protective materials between items, using dunnage (airbags, wood) to fill gaps, strapping/securing pallets and individual items

6. **Sealing and Documentation**: Container is sealed with a tamper-evident seal. Seal number is recorded. All documentation is completed

### 2.2 Documentation Created During Consolidation

| Document | Created By | Purpose |
|----------|-----------|---------|
| **House Bill of Lading (HBL)** | NVOCC/Freight Forwarder | Contract of carriage for each individual consignment |
| **Master Bill of Lading (MBL)** | Ocean Carrier | Covers the entire container; lists the NVOCC/consolidator as shipper |
| **Packing List** | Each shipper | Details contents of each shipment |
| **Container Load Plan / Stuffing Report** | CFS operator | Documents what was loaded, how, container/seal numbers |
| **Dock Receipt** | CFS/Terminal | Acknowledges receipt of cargo at the CFS |
| **Commercial Invoice** | Each shipper | Value declaration for each consignment |
| **Certificate of Origin** | Each shipper (if required) | Country of origin for each consignment |
| **Tally Sheet** | CFS operator | Piece count verification during loading |

### 2.3 Stuffing Report / Container Load Plan Structure

A stuffing report typically contains:
- **Header**: Container number, container type/size, seal number, vessel name, voyage number, port of loading, port of discharge
- **Per-Consignment Section** (repeated for each HBL):
  - HBL number
  - Shipper name
  - Consignee name
  - Number of packages
  - Package type (cartons, pallets, drums, etc.)
  - Marks and numbers
  - Cargo description
  - Gross weight
  - Volume (CBM)
- **Totals**: Total packages, total weight, total volume, remaining container capacity
- **Customs Endorsement** (in some jurisdictions): Stamp and signature of customs inspector confirming proper inspection before sealing

### 2.4 Consignment Identification Within a Container

Each consignment within a consolidated container is identified through:
1. **HBL Number**: The unique reference linking documentation to physical cargo
2. **Shipping Marks**: Physical markings on each handling unit matching the HBL
3. **Transport Labels**: Applied by the consolidator with MBL/HBL reference numbers
4. **Piece Count**: Exact number of handling units per consignment, reconciled at stuffing and destuffing
5. **Weight and Volume**: Each consignment's measurements are recorded separately

## 3. Ocean Container Examination by CBP

### 3.1 When CBP Targets a Specific HBL Within a Consolidated LCL Container

CBP's Automated Targeting System (ATS) scores each shipment based on risk factors. When a specific HBL within an LCL container is flagged, the examination process depends on the type of exam ordered:

**VACIS/NII (Non-Intrusive Inspection / X-Ray):**
- The ENTIRE container is scanned — there is no way to x-ray only one shipment within a container
- The container is driven through a high-energy imaging system at the ocean terminal
- The container seal is NOT broken
- Officers compare the image to the cargo manifest
- If anomalies are found, it may escalate to a tailgate or intensive exam
- Cost: $150–$350 per container, typically split proportionally among all importers in the container
- Timeline: 1–3 days

**Tailgate Examination:**
- Container seal is broken and doors opened for visual inspection
- Does NOT require complete unloading
- Officers can visually inspect accessible cargo
- Limited usefulness for LCL if the targeted shipment is in the back of the container

**Intensive Examination:**
- The entire container is trucked to a CES (Centralized Examination Station)
- The container is FULLY devanned/unstuffed — this is required to access any specific shipment
- However, CBP then **examines only the designated cargo** (the targeted HBL's shipment)
- CES personnel segregate each set of parcels, open designated boxes, and ready the specific cargo for CBP officer inspection
- Other consignments are separated but not examined
- Cost: $1,000–$2,500+, split among all importers
- Timeline: 5–7 days, but can extend to 3 weeks depending on CES congestion

### 3.2 CES Procedures for Examining LCL Containers

A Centralized Examination Station (CES) is a privately owned, CBP-bonded facility. The process:

1. **Container Transport**: The entire container is drayed from the port/terminal to the CES
2. **Seal Verification**: CBP performs seal verification on all containers prior to unloading
3. **Full Devanning**: The container is completely unstuffed. For LCL, all consignments must be removed
4. **Segregation**: CES personnel separate cargo by consignment/HBL, using shipping marks and labels
5. **Notification**: CES notifies CBP that cargo is on the floor and ready for examination
6. **Examination**: CBP determines when the exam begins (they have 5 working days). They examine only the designated cargo
7. **Possible Sampling**: CBP officers may take samples for lab testing
8. **Release Decision**: CBP decides to release or place additional holds
9. **Reloading**: Cargo is repacked (though not necessarily with the same care as original packing)
10. **Damage Reporting**: Any damage noticed during devanning is documented via photos and a damage report

### 3.3 Must the Entire Container Be Devanned?

**Yes, for intensive exams.** Even if only one HBL's cargo is targeted, the entire container must be stripped because:
- The targeted shipment may be anywhere in the container (not necessarily accessible from the doors)
- CBP requires full visibility to verify the targeted shipment
- The CES must segregate all consignments to identify and isolate the designated cargo

**No, for VACIS/NII exams** — the container passes through the scanner intact.

**Partial for tailgate exams** — only the doors are opened for visual inspection; no full devanning required, though this limits what can be inspected.

### 3.4 Impact on Other Consignments in the Same Container

- **All consignments are delayed** when the container goes for any exam, regardless of which HBL was targeted
- For VACIS exams, the delay is typically 1–3 days
- For intensive exams, delays can range from 5 days to 3 weeks
- Other consignments are NOT physically examined, but they must wait for the container to be processed
- Once CBP releases the targeted cargo, the remaining consignments can proceed to normal deconsolidation and release
- One problematic shipment in an LCL container can hold up every other shipment

### 3.5 Who Bears Examination Costs?

**The Importer of Record is financially responsible** for exam costs. For LCL containers:

| Cost Component | Who Pays | How Split |
|---------------|----------|-----------|
| **VACIS/NII exam fee** | Split among all importers in container | Proportionally (usually by CBM) |
| **Drayage to/from CES** | Split among all importers | Proportionally |
| **CES devanning/handling** | Split among all importers | Proportionally |
| **CES storage** | Split among all importers | Proportionally |
| **Demurrage/detention** | Split among all importers | Proportionally |
| **Intensive exam labor** | Typically allocated to targeted importer or split | Varies by forwarder |

Key points:
- CBP does NOT charge directly — fees come from private entities (CES, carriers, drayage companies)
- CBP does NOT reveal which shipment triggered the exam (for security reasons)
- The freight forwarder/arrival agent typically calculates and collects fees from all importers
- Costs are generally divided proportionally by CBM (cubic meters)
- Payment is required before cargo release

### 3.6 Exam Cost Ranges

| Exam Type | Cost Range | Timeline |
|-----------|-----------|----------|
| **VACIS/NII (X-Ray)** | $80–$1,000 per container | 1–3 days |
| **Tailgate** | $150–$350 per container | 1–2 days |
| **Intensive (Full Devanning)** | $1,000–$2,500+ per container | 5–7 days (up to 3 weeks) |

Additional potential charges: overtime surcharges ($150/hr), overweight surcharges ($100–$225), cancellation fees ($250 if container already picked up).

## 4. Container-Level Documentation for Customs

### 4.1 Container Manifest Requirements (19 CFR 4.7 / 4.7a)

Under the 24-Hour Advance Manifest Rule, the cargo declaration filed with CBP must include:

| Data Element | Description |
|-------------|------------|
| **Carrier SCAC Code** | Unique Standard Carrier Alpha Code |
| **Vessel Name, Country of Documentation, Official Number** | Vessel identification |
| **Voyage Number** | Specific voyage identifier |
| **Date of Departure / Date of Arrival** | Sailing and expected arrival dates |
| **First Foreign Port** | Where carrier takes possession of US-bound cargo |
| **Last Foreign Port** | Last port before departing for US |
| **Shipper Name and Address** | Full name and valid foreign address with city/province/country/postal code |
| **Consignee Name and Address** | Full name and valid US address |
| **Notify Party** | Additional contact party (required for "to order" shipments) |
| **Bill of Lading Number** | Unique identifier starting with carrier's SCAC code |
| **Cargo Description** | Precise description or HTS number to 6-digit level. "FAK," "general cargo," and "STC" are NOT acceptable |
| **Total Quantity** | Based on smallest external packaging unit (NOT containers or pallets) |
| **Weight** | Gross weight in lbs or kg |
| **Container Number** | ISO container identification |
| **Seal Number** | Tamper-evident seal identification |

**Critical rule on quantity**: A container with 10 pallets containing 200 cartons must be manifested as 200 cartons, not 10 pallets or 1 container.

### 4.2 How Container Information Is Filed with CBP (AMS)

The Automated Manifest System (AMS) is the electronic system for filing manifest data:

**Filing Hierarchy:**
```
MANIFEST (= one vessel sailing)
  └── MASTER BILL OF LADING (filed by ocean carrier / shipping line)
        └── HOUSE BILL OF LADING (filed by NVOCC / freight forwarder)
              └── COMMODITY LINES (one or more per HBL)
                    └── CONTAINER ASSOCIATIONS (one or more per commodity)
```

**Who Files What:**
- **Ocean Carriers** file AMS for the Master Bill of Lading
- **NVOCCs/Freight Forwarders** file AMS for House Bills of Lading they issue
- Both must file at least 24 hours before loading at the foreign port

**Regular vs. Non-Regular AMS:**
- **Regular AMS**: Information on AMS matches MBL. No HBL involved (typically FCL with direct carrier relationship)
- **Non-Regular AMS**: AMS information differs from MBL. Both MBL and HBL are present (typical for LCL/consolidation)

**MBL-HBL Matching:**
- AMS MBL (carrier filing) and AMS HBL (NVOCC filing) must match before goods can be loaded
- If they don't match by the deadline, goods are held at the transshipment port with penalty fees
- A new check verifies that the sum of all HBL amended quantities matches the MBL amended quantity

### 4.3 Relationship Between MBL, HBLs, and Container Manifest

```
CONTAINER #ABCU1234567 (Seal #SL789456)
├── MBL: MAEU123456789 (filed by carrier, e.g., Maersk)
│   ├── HBL: EXFR001122 (filed by NVOCC/forwarder)
│   │   └── Consignment A: 50 cartons, 2,000 kg, "electronic components"
│   ├── HBL: EXFR003344 (filed by NVOCC/forwarder)
│   │   └── Consignment B: 30 cartons, 1,500 kg, "textile garments"
│   └── HBL: EXFR005566 (filed by NVOCC/forwarder)
│       └── Consignment C: 20 pallets, 5,000 kg, "auto parts"
```

- The **MBL** covers the entire container. The carrier files this with CBP via AMS
- Each **HBL** represents one consignment within the container. The NVOCC/forwarder files these with CBP via AMS
- CBP can place or remove holds on: entire manifests, individual bills of lading, or specific containers
- Container details in the HBL filing may initially be "TBN" (to be nominated) for LCL, updated once the container is assigned

### 4.4 ISF Filing for Consolidated Containers

**Each importer in a consolidated container must file their own separate ISF** (Importer Security Filing, aka "10+2").

**The 10 elements from the importer/broker:**
1. Importer of Record number (EIN/tax ID)
2. Consignee number
3. Seller name and address
4. Buyer name and address
5. Manufacturer name and address
6. Ship-to party name and address
7. Country of origin
8. Commodity HTS number (minimum 6 digits)
9. Container stuffing location
10. Consolidator (stuffer) name and address

**The 2 elements from the carrier:**
1. Vessel stow plan (within 48 hours after departure)
2. Container Status Messages (CSMs)

**Key rules for LCL ISF:**
- There is NO combined ISF filing — each importer files separately even if they share a container
- Filing must occur at least 24 hours before loading at the foreign port
- The importer carries ultimate liability even if a broker/forwarder files on their behalf
- Container stuffing location may not be known until late in the booking process (common challenge for LCL)
- Multiple shippers in one HBL: assign primary shipper for ISF, list others in supplemental notes
- Split shipments across HBLs: file one ISF per HBL, referencing the same MBL
- Penalty: $5,000 per violation, plus potential cargo holds, delays, and Do Not Load (DNL) orders

## 5. FCL vs. LCL Customs Implications

### 5.1 Examination Procedures

| Aspect | FCL | LCL |
|--------|-----|-----|
| **Exam frequency** | Lower (single importer, single origin typically) | Higher (multiple shippers increase the probability that at least one triggers an exam) |
| **VACIS/NII** | Container scanned; cost borne by single importer | Container scanned; cost split among all importers |
| **Tailgate** | Doors opened; single importer's cargo accessible | Doors opened; only cargo near doors accessible; limited utility |
| **Intensive** | Full devanning of single importer's cargo | Full devanning required; all consignments removed but only targeted one examined |
| **Exam cost allocation** | 100% to the single importer | Proportionally split among all importers in the container |
| **Delay impact** | Only affects one importer | Delays ALL importers sharing the container |

### 5.2 Entry Filing Procedures

| Aspect | FCL | LCL |
|--------|-----|-----|
| **B/L structure** | One MBL (or one simple B/L) per container | One MBL per container + multiple HBLs (one per consignment) |
| **Entry filing** | One entry filed against the B/L by the single importer | Each importer files a separate entry against their own HBL |
| **ISF filing** | One ISF filed by the importer | Each importer files their own ISF separately |
| **Customs broker** | Importer chooses their own broker | Each importer may use their own broker, or the forwarder's broker handles all |
| **Clearance independence** | Importer controls their own timeline | Each importer clears independently, but container release depends on ALL holds being resolved |

### 5.3 One Entry Per Bill — How It Works Differently

**FCL — One Entry Per B/L:**
- One container = one B/L = one customs entry
- The B/L is a simple or regular bill (not a master/house structure unless the importer uses a forwarder)
- If using a forwarder: one MBL (carrier to forwarder) and one HBL (forwarder to importer), but still one entry

**LCL — One Entry Per HBL:**
- One container = one MBL = multiple HBLs = multiple customs entries
- Each HBL represents a separate importation by a separate importer of record
- Under 19 CFR § 141.52–141.54, separate entries may be filed for different portions of a consolidated shipment
- The consignee named on the MBL (typically the NVOCC/consolidator) executes a certificate allowing separate entries
- CBP now supports HBL-level release, enabling granular customs processing

**Buyer Consolidation (one importer, multiple suppliers):**
- Special case: one importer consolidates goods from multiple suppliers into one container
- Each supplier may have its own HBL, but the importer can file a single customs entry under the MBL
- This is different from standard LCL where multiple unrelated importers share a container

### 5.4 Container Hold vs. Bill-Level Hold

**Container/Equipment-Level Hold:**
- Applied to the PHYSICAL container, regardless of contents
- Generated by CBP online input
- Affects ALL bills of lading associated with that container
- The entire container cannot be moved until the hold is released
- Container holds are transmitted to the carrier/MTO

**Bill of Lading-Level Hold (1H disposition code):**
- Applied to a specific B/L (either MBL or HBL)
- Affects only the cargo associated with that particular B/L
- Other B/Ls in the same container may or may not be affected:
  - If the HBL hold prevents the master bill from getting a 1Z, the physical container remains at the terminal
  - The container cannot be released until ALL HBLs have a CBP-authorized movement

**Key Disposition Codes:**

| Code | Meaning | Level |
|------|---------|-------|
| **1C** | Entered and Released | BOL level — cargo cleared |
| **1H** | CBP Hold Placed at Port of Discharge | BOL level — release denied |
| **1I** | CBP Hold Removed at Port of Discharge | BOL level — hold lifted |
| **1W** | Permit to Transfer (PTT) | BOL level — authorized movement (e.g., to CFS) |
| **1Z** | Ocean Terminal Release | Master BOL level — all HBLs have authorized movements; container can leave terminal |
| **4Z** | Cancel 1Z | Master BOL level — one or more HBLs no longer have authorized movement |
| **7H** | VACIS Exam Required | Equipment/container level |

**The 1Z workflow for LCL:**
1. Individual HBLs receive their disposition codes (1C, 1W, 1J, etc.)
2. Marine Terminal Operators (MTOs) do NOT receive HBL-level notifications
3. Once ALL HBLs under a MBL have a CBP-authorized movement, the MBL receives a 1Z
4. The 1Z tells the terminal the physical container can be released
5. If any HBL loses its authorized movement, a 4Z cancels the 1Z

## 6. Deconsolidation at Destination

### 6.1 CFS Operations for Breaking Down Consolidated Containers

The destination CFS deconsolidation process follows these steps:

1. **Container Arrival**: After vessel discharge, the container is drayed from the port terminal to the destination CFS. This can only happen once the MBL receives a 1Z (or the container has a PTT for CFS movement)

2. **Seal Verification and Documentation Check**: CFS verifies the container seal number matches documentation. Container condition is noted. Incoming paperwork (MBL, HBL manifest, packing lists) is checked

3. **Unstuffing/Stripping**: The container is opened and all cargo is methodically removed. Professional CFS workers follow handling protocols to prevent damage

4. **Tally and Verification**: As each handling unit comes out, CFS workers:
   - Read shipping marks and transport labels
   - Match marks/labels to HBL references on the house manifest
   - Count pieces and verify against the documented quantity
   - Note any damage or discrepancies
   - Issue a tally report / discrepancy report if counts don't match

5. **Sorting and Segregation**: Cargo is physically sorted and grouped by:
   - HBL number (one pile/area per consignment)
   - Final destination (if cargo is being transshipped)
   - Consignee name

6. **Inventory Entry**: Each consignment is entered into the CFS inventory management system with:
   - HBL number
   - Piece count
   - Weight
   - Condition (any noted damage)
   - Storage location within the CFS

7. **Bonded Storage**: Goods at the CFS are in-bond — they are not considered entered into US commerce until customs approves release. This allows customs examination before clearance

### 6.2 How Individual Handling Units Are Identified and Sorted

The CFS uses multiple identification methods during deconsolidation:

1. **Shipping Marks** (Primary Method): The marks on each carton/pallet/crate are the primary way CFS workers identify which consignment a handling unit belongs to. This is why accurate, prominent marks are critical for LCL

2. **Transport Labels**: Labels applied at the origin CFS with MBL/HBL numbers and sequential package numbers

3. **Packing List Cross-Reference**: The house manifest and individual packing lists provide the expected marks, piece counts, and descriptions that CFS workers verify against

4. **Physical Characteristics**: Weight, dimensions, and packaging type serve as secondary verification

5. **SSCC Barcodes** (where used): Scanning SSCC labels provides automated identification, though this is not universal in ocean freight

**Common problems during sorting:**
- Damaged or missing marks (cargo may be set aside as "unidentified")
- Marks that don't match any HBL on the house manifest
- Piece count discrepancies (more or fewer handling units than documented)
- Cargo that has shifted and damaged other consignments

### 6.3 Tracking Through the Deconsolidation Process

**Manifest Submission Flow:**
1. Shipping Agent submits the Ocean Manifest (MBL level) to Customs/Port Operator
2. Freight Forwarder submits the House Manifest (HBL level) to Customs/Port Operator
3. The freight forwarder presents the Original Bill of Lading (OBL) to the shipping agent to exchange for a Delivery Order (DO)

**Physical Tracking:**
- Container tracked by container number from port to CFS
- At CFS, tracking transitions to HBL-level as cargo is deconsolidated
- CFS inventory system tracks each consignment by HBL number, piece count, and storage location
- Consignees/brokers can typically check status with the CFS or forwarder

**Electronic Status Updates:**
- CBP disposition codes (1C, 1H, 1W, 1Z) provide release status visibility
- Carriers and forwarders provide tracking through their digital platforms
- MTOs receive master bill-level status updates

### 6.4 Release Procedures for Individual Consignments After Customs Clearance

The release process for each individual consignment within an LCL container:

1. **Customs Entry Filing**: Each importer (or their broker) files a customs entry against their HBL. This can happen before the container arrives or after

2. **Customs Review**: CBP processes the entry. Outcomes:
   - **Released (1C)**: Entry is cleared, no exam needed
   - **Held (1H)**: CBP requires additional information, documentation, or examination
   - **Exam Ordered (7H or other)**: Shipment selected for VACIS, tailgate, or intensive exam

3. **Duty Payment**: Import duties and fees must be paid (or bonded) before release

4. **Individual Release**: Once CBP issues a release for a specific HBL:
   - The CFS is authorized to release that consignment to the consignee or their agent
   - Other consignments in the same container may still be held if they haven't cleared
   - Each consignment is released independently based on its own customs status

5. **Delivery Order (DO) Issuance**: The freight forwarder issues a Delivery Order authorizing the CFS to release the cargo to the named party

6. **Gate Pass**: After customs release + DO + any outstanding charges are paid (CFS handling, storage, etc.), a gate pass is issued

7. **Pickup / Last-Mile Delivery**: The consignee arranges pickup from the CFS, or the forwarder provides last-mile delivery. The CFS verifies the identity of the pickup party against the DO

**Important timing considerations:**
- CFS "free time" (typically 3–5 days) before storage charges accrue
- Demurrage and storage fees accumulate quickly if clearance is delayed
- One consignment's customs hold does NOT prevent other cleared consignments from being released (once they are physically separated at the CFS)
- However, if the container hasn't been deconsolidated yet (e.g., still at terminal waiting for 1Z), ALL consignments wait

### 6.5 The Complete LCL Import Flow (US)

```
1. Vessel arrives at US port
2. Container discharged from vessel to terminal yard
3. CBP processes manifest and entry data
4. If ALL HBLs have authorized movements → 1Z issued → container released to CFS
   If ANY HBL has a hold → container stays at terminal (or goes for exam)
5. Container drayed to destination CFS
6. CFS breaks seal, unstuffs container
7. Cargo sorted by HBL, tallied, inventoried
8. Each consignment sits in bonded storage at CFS
9. Each importer's broker files entry (if not already done)
10. CBP releases individual HBLs as entries are processed
11. For each released consignment:
    a. Forwarder issues Delivery Order
    b. Importer pays CFS charges
    c. Gate pass issued
    d. Importer arranges pickup or delivery
12. Empty container returned to carrier
```

## Sources

### Ocean Cargo Handling and LCL
- [DHL Global Forwarding - LCL Ocean Freight](https://www.dhl.com/us-en/home/global-forwarding/products-and-solutions/ocean-freight/less-than-container-load.html)
- [Maersk - Less-than-Container Load](https://www.maersk.com/transportation-services/lcl)
- [GEODIS - LCL Consolidation Service](https://geodis.com/gb-en/transport-services/ocean-freight/lcl-consolidation-service)
- [CEVA Logistics - FCL LCL Shipping](https://www.cevalogistics.com/en/glossary/fcl-lcl)

### CFS Operations
- [Midstate Containers - Container Freight Station](https://midstatecontainers.com/blogs/news/container-freight-station-cfs)
- [Freightos - CFS Meaning and Charges](https://www.freightos.com/freight-resources/container-freight-station-cfs/)
- [ECU360 - Container Freight Station Functions](https://ecu360.com/contentHub/blog/container-freight-station-meaning-functions-and-role-in-shipping/)
- [Flexport Glossary - CFS](https://www.flexport.com/glossary/container-freight-station/)
- [FreightAmigo - Understanding CFS](https://www.freightamigo.com/blog/understanding-container-freight-stations-cfs-and-consolidated-shipments-a-comprehensive-guide)

### CBP Customs Examination
- [Ship4wd - Customs Holds and Exams](https://ship4wd.com/import-guides/customs-holds-customs-exams)
- [Flexport - List of Customs Holds and Exams](https://www.flexport.com/help/299-customs-holds-and-exams/)
- [Shapiro - A Behind-the-Scenes Look at Customs Inspection](https://www.shapiro.com/examining-the-issue-customs-inspection/)
- [Flexport - Types of Customs Exams](https://support.portal.flexport.com/hc/en-us/articles/16543331771671-Types-of-Customs-Exams)
- [LegalClarity - Customs Examination Procedures](https://legalclarity.org/customs-examination-procedures-types-steps-and-costs/)

### CES Operations
- [WTCFS - What is a CES?](https://www.wtcfs.com/2014/04/14/what-is-a-c-e-s/)
- [Ship4wd - Centralized Examination Station](https://ship4wd.com/resource-center/glossary/centralized-examination-station-ces)
- [Flexport Glossary - CES](https://www.flexport.com/glossary/centralized-examination-station/)
- [Georgia Ports Authority - CES Rule 34-525](https://gaports.com/blog/mto/government-agency-inspections-centralized-examination-station-ces/)
- [19 CFR Part 118 - Centralized Examination Stations](https://www.ecfr.gov/current/title-19/chapter-I/part-118)
- [East Coast Warehouse - CES Fee Schedule](https://eastcoastwarehouse.com/wp-content/uploads/2022/04/CES-PA-Fee-Schedule_060122.pdf)

### Examination Costs
- [Flexport - Who is Responsible for Customs Exam Costs?](https://www.flexport.com/help/421-customs-exams-who-pays/)
- [FreightRight - Exam Charges](https://www.freightright.com/kb/exam-charges)
- [Forest Leopard - Complete Guide to US Customs Exams](https://forestleopard.com/knowledgedetail/Stuck-at-the-Border--A-Complet)

### AMS Manifest and Documentation
- [CBP - ACE Import Manifest Documentation](https://www.cbp.gov/trade/automated/technical/ace-import-manifest-documentation)
- [CBP - Ocean House Bill of Lading FAQ](https://www.cbp.gov/trade/automated/ace-faq/ohbol-faq)
- [Magaya - AMS Filing](https://www.magaya.com/automated-manifest-system/)
- [GoFreight - AMS Filing Guide](https://gofreight.com/blog/solution/what-is-the-automated-manifest-system-ams-simple-ams-filing-guide.html)
- [CustomsCity - CBP Automated Manifest System](https://customscity.com/what-is-the-cbp-automated-manifest-system-ams/)

### ISF Filing
- [US Import Bond - ISF Filing for LCL Shipments](https://usimportbond.com/isf-filing-for-lcl-shipments-compliance-template-for-customs-brokers-updated-for-2025/)
- [ISF Filer - Impact of ISF on LCL Shipments](https://isffiler.com/import-customs/the-impact-of-isf-filing-on-lcl-shipments/)
- [US Customs Clearing - ISF Filing for Consolidated Shipments](https://uscustomsclearing.com/isf-filing-for-consolidated-shipments/)
- [Ship4wd - ISF Filing Requirements](https://ship4wd.com/resource-center/glossary/isf-filing)

### FCL vs LCL
- [Citrus Freight - FCL and LCL Differences](https://www.citrusfreight.com/resource/blog/fcl-and-lcl)
- [eezyimport - FCL Shipment Regulations](https://www.eezyimport.com/regulations-concerning-full-container-load-fcl-shipments/)
- [JingSourcing - FCL vs LCL](https://jingsourcing.com/b-fcl-vs-lcl/)

### CBP Disposition Codes
- [CBP - CSMS 53345870 - Disposition Code 1Z](https://content.govdelivery.com/accounts/USDHSCBP/bulletins/32dfe4e)
- [CBP - CATAIR Appendix N Disposition Codes](https://www.cbp.gov/sites/default/files/assets/documents/2023-Feb/ACE%20Appendix%20N%20-%202.15.2023_508c.pdf)
- [USA Customs Clearance - Customs Hold Types](https://usacustomsclearance.com/process/customs-hold/)
- [APM Terminals - Disposition Codes](https://www.apmterminals.com/en/miami/practical-information/disposition-codes)

### Shipping Marks and Identifiers
- [JingSourcing - Shipping Marks Guide](https://jingsourcing.com/b-shipping-marks-in-internatioal-trading/)
- [Ocean Cargo UK - Marks and Numbers](https://oceancargo.co.uk/blog/shipping-instructions-marks-and-numbers)
- [iContainers - How to Label Your Shipment](https://www.icontainers.com/help/how-to-properly-label-your-shipment/)
- [GS1 - SSCC Standard](https://www.gs1.org/standards/id-keys/sscc)
- [GS1 US - Serialized Shipping Container Codes](https://www.gs1us.org/upcs-barcodes-prefixes/serialized-shipping-container-codes)

### Deconsolidation and Release
- [Ship4wd - Deconsolidation Process](https://ship4wd.com/resource-center/glossary/deconsolidation)
- [JCtrans - LCL Customs Clearance Process](https://m.jctrans.com/en/blog/21378)
- [Westports - Import Process Flow for LCL](https://www.westportsholdings.com/wp-content/uploads/files/Import_Process_Flow-LCL.pdf)
- [CrimsonLogic - CFS Container Freight Station Guide](https://crimsonlogic-northamerica.com/cfs-container-freight-station-guide/)

### Customs Entry Regulations
- [19 CFR Part 141 - Entry of Merchandise](https://www.ecfr.gov/current/title-19/chapter-I/part-141)
- [19 CFR 4.7a - Inward Manifest Requirements](https://www.ecfr.gov/current/title-19/chapter-I/part-4/subject-group-ECFR9356f0b8b5be866/section-4.7a)

### Container Stuffing
- [Maersk - Container Stuffing 101](https://www.maersk.com/logistics-explained/transportation-and-freight/2024/07/12/container-stuffing)
- [OOCL - Stuffing Plan](https://www.oocl.com/eng/ourservices/containers/Pages/stuffingplan.aspx)
