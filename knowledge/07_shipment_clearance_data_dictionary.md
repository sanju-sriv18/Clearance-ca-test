# International Shipment and Customs Clearance: Comprehensive Data Dictionary

> **FedEx Global Clearance Knowledge Base**
> Last Updated: February 2026

---

## Purpose of This Document

This document catalogs every significant data attribute used across international shipment and customs clearance operations. For each attribute, it explains: what it is, why it exists, who creates it, when it is created, whether it can change after creation, and what operational decision or regulatory requirement it enables.

This is a reference for engineers, business analysts, product managers, and domain specialists building or evaluating clearance systems. Every field described here has a real-world consequence -- a wrong value, a missing value, or a stale value will cause a shipment to be delayed, a duty to be miscalculated, or a compliance violation to be triggered.

---

## 1. Shipment / Consignment Record

The shipment record is the master entity. It represents the physical movement of goods from origin to destination and is the anchor to which all other records (parties, line items, customs entries, events, financials) attach.

---

### 1.1 Shipment Identifiers

#### Tracking Number / Shipment ID

- **What it is:** A unique identifier assigned by the carrier to a single physical shipment. For express carriers (FedEx, UPS, DHL), this is the tracking number printed on the label. For freight shipments, this may be an internal reference number.
- **Format:** Carrier-specific. FedEx uses 12- or 15-digit numeric tracking numbers. UPS uses 18-character "1Z" prefixed alphanumeric codes. DHL uses 10-digit numeric codes.
- **Who creates it:** The originating carrier, at the time of shipment creation or label generation.
- **When created:** At booking or label print, before the package is tendered to the carrier.
- **Mutable:** No. Once assigned, it is permanent and immutable for the life of the shipment. This is the primary key for customer-facing tracking and internal operations.
- **Why it matters:** This is the single most referenced identifier in the entire shipment lifecycle. It links the physical package to all electronic records -- customs entries, events, financials, customer communications. Without it, there is no way to correlate the piece of cardboard on the conveyor belt with the data in the system.

#### House Air Waybill (HAWB)

- **What it is:** The transport contract between the shipper and the freight forwarder or express carrier for a specific shipment. The HAWB identifies a single consignment within a larger consolidated shipment.
- **Format:** Typically 8-12 alphanumeric characters. For IATA-standard air waybills, the format is an 11-digit number (3-digit airline prefix + 8-digit serial number).
- **Who creates it:** The freight forwarder or express carrier.
- **When created:** At booking or shipment acceptance.
- **Mutable:** No. The HAWB number is fixed once issued.
- **Why it matters:** In consolidated shipments (where multiple consignments from different shippers are grouped into a single container or pallet for transport), the HAWB identifies which consignment belongs to which shipper. CBP and other customs authorities use the HAWB to link an individual customs entry to its physical shipment within a consolidated load. If the HAWB is wrong, customs cannot match the entry to the goods, causing holds and delays.

#### Master Air Waybill (MAWB)

- **What it is:** The transport contract between the freight forwarder/consolidator and the airline for the entire consolidated shipment. A single MAWB covers many HAWBs.
- **Format:** 11-digit IATA standard (3-digit airline code + 8-digit serial).
- **Who creates it:** The airline or its agent, at the time the consolidated shipment is accepted for carriage.
- **When created:** When the consolidation is tendered to the airline.
- **Mutable:** No.
- **Why it matters:** The MAWB connects the physical aircraft manifest to the cargo. CBP's Air Cargo Advance Screening (ACAS) program requires MAWB-level data submission before departure. The MAWB-to-HAWB relationship is critical for consolidated shipments -- it tells customs "this aircraft carried these consolidated loads, each of which contains these individual consignments." For de-consolidation at destination, the MAWB tells the ground handler which HAWBs to expect and where to route them.

#### Bill of Lading Number (Ocean B/L)

- **What it is:** The ocean freight equivalent of the air waybill. Serves three functions: receipt of goods by the carrier, evidence of the contract of carriage, and document of title to the goods.
- **Format:** Carrier-specific, typically alphanumeric, 10-20 characters.
- **Who creates it:** The ocean carrier (for a master B/L) or the NVOCC/freight forwarder (for a house B/L).
- **When created:** When the container is loaded and the vessel departs, or at booking confirmation.
- **Mutable:** No, though amendments (corrections) can be issued.
- **Why it matters:** The B/L is a legal document. It determines who has title to the goods and who can take delivery. Banks use it in letters of credit. Customs requires it for ISF filing and entry processing. The B/L number links the customs entry to the physical container on the vessel. An error in the B/L number can prevent customs release and container pickup.

#### Pro Number (Truck)

- **What it is:** The freight carrier's tracking number for a truck shipment (LTL or truckload). "Pro" stands for "progressive" -- a sequential number assigned to each shipment.
- **Format:** Carrier-specific, typically 8-12 digits.
- **Who creates it:** The trucking carrier.
- **When created:** At pickup or booking.
- **Mutable:** No.
- **Why it matters:** For cross-border truck freight (US-Mexico, US-Canada), the pro number is the reference used at the border crossing. CBP requires it for in-bond and entry filing for truck-mode shipments.

---

### 1.2 Service and Transport

#### Service Type

- **What it is:** The level of service contracted for the shipment. Determines transit time commitment, handling priority, and pricing.
- **Values (express):** International Priority (IP), International Economy (IE), International First, International Priority Freight (IPF), International Economy Freight (IEF).
- **Values (freight):** Full Container Load (FCL), Less-than-Container Load (LCL), Full Truckload (FTL), Less-than-Truckload (LTL), Air Freight, Sea-Air, Rail.
- **Who creates it:** Selected by the shipper at booking.
- **When created:** At booking / order creation.
- **Mutable:** Can sometimes be upgraded or downgraded before pickup, but not after goods are in transit.
- **Why it matters for clearance:** Service type determines the clearance processing window. Express shipments (IP, International First) have hours, not days, for clearance -- they must be pre-cleared or cleared within the hub sort window to maintain transit time. Economy services have more buffer. Freight shipments (FCL/LCL) have days to weeks. The service type drives the priority and urgency of clearance operations. It also affects which customs programs apply -- some expedited clearance programs (C-TPAT, AEO) provide benefits only for certain service types.

#### Transport Mode

- **What it is:** How the goods physically move across the border.
- **Values:** Air, Ocean, Truck, Rail, Multimodal, Pipeline, Mail/Postal.
- **Who creates it:** Determined by the service type and routing.
- **When created:** At booking.
- **Mutable:** Rarely; changing mode mid-transit is exceptional.
- **Why it matters for clearance:** Transport mode determines: which advance filing requirements apply (ACAS for air, ISF 10+2 for ocean, ACE eManifest for truck); the port of entry; the applicable fees (Harbor Maintenance Fee applies only to ocean); the timing of customs processing; and the physical exam logistics (tailgate exam at an airport facility versus container exam at a seaport).

#### Carrier SCAC Code

- **What it is:** Standard Carrier Alpha Code -- a 2-4 character code identifying the carrier.
- **Format:** 2-4 uppercase letters (e.g., FEDX, UPSN, MAEU for Maersk).
- **Who creates it:** Assigned by the National Motor Freight Traffic Association (NMFTA).
- **When created:** At carrier registration; static per carrier.
- **Mutable:** No.
- **Why it matters:** CBP uses the SCAC code to identify the carrier on customs filings. The carrier's compliance history, C-TPAT status, and risk profile are all indexed by SCAC. A carrier with a poor compliance record (frequent violations, data quality issues) may see its shipments targeted at higher rates.

---

### 1.3 Incoterms

#### Incoterm Code

- **What it is:** An International Chamber of Commerce (ICC) standardized three-letter code that defines the allocation of costs, risks, and responsibilities between buyer and seller in an international transaction.
- **Who creates it:** Agreed between buyer and seller at the time of the commercial transaction (purchase order, sales contract).
- **When created:** At the time of sale/contract.
- **Mutable:** Only by mutual agreement of the parties; cannot be changed unilaterally after shipment.
- **Why it matters for clearance:** The Incoterm determines WHO is responsible for customs clearance, duty payment, and insurance -- and critically, it determines WHAT costs are included in the customs value (the value on which duties are assessed).

**Incoterms Explained (2020 edition, the current standard):**

| Incoterm | Full Name | Seller's Responsibility Ends | Who Clears Export | Who Clears Import | Who Pays Duty | Insurance | Effect on Customs Value |
|----------|-----------|------------------------------|-------------------|-------------------|---------------|-----------|------------------------|
| **EXW** | Ex Works | At seller's premises | Buyer | Buyer | Buyer | Buyer's risk | Customs value = purchase price only. Freight, insurance NOT included. CBP requires adding freight to the port of entry for US valuation. |
| **FCA** | Free Carrier | Goods delivered to carrier at named place | Seller | Buyer | Buyer | Buyer's risk | Value includes costs to the carrier pickup point. Freight from there to destination is buyer's responsibility. |
| **FAS** | Free Alongside Ship | Goods alongside vessel at port of shipment | Seller | Buyer | Buyer | Buyer's risk | Ocean only. Value includes delivery to port. |
| **FOB** | Free On Board | Goods loaded on vessel at port of shipment | Seller | Buyer | Buyer | Buyer's risk | Ocean only. Value includes loading onto ship. This is the most common Incoterm for ocean freight. For US customs valuation, FOB is the base -- CBP assesses duty on the FOB value (transaction value at the port of export). |
| **CFR** | Cost and Freight | Seller pays freight to destination port | Seller | Buyer | Buyer | Buyer's risk | Ocean only. Value includes freight to destination port. For US customs, the freight portion must be DEDUCTED to arrive at FOB value for duty calculation. |
| **CIF** | Cost, Insurance, Freight | Seller pays freight and insurance to destination port | Seller | Buyer | Buyer | Seller provides minimum insurance | Ocean only. Value includes freight and insurance to destination. For US customs, both must be deducted to get FOB. For EU customs, CIF IS the customs value -- the EU assesses duty on CIF value, not FOB. This is a critical distinction. |
| **CPT** | Carriage Paid To | Seller pays freight to named destination | Seller | Buyer | Buyer | Buyer's risk | Any mode. Similar to CFR but for non-ocean transport. |
| **CIP** | Carriage and Insurance Paid To | Seller pays freight and insurance to named destination | Seller | Buyer | Buyer | Seller provides all-risk insurance | Any mode. Similar to CIF for non-ocean. |
| **DAP** | Delivered at Place | Goods arrive at named destination, ready for unloading | Seller | Buyer | Buyer | Seller's risk until delivery | Seller bears all risk and cost to destination, but buyer handles import clearance and duty. |
| **DPU** | Delivered at Place Unloaded | Goods unloaded at named destination | Seller | Buyer | Buyer | Seller's risk until unloading | Seller bears cost of unloading. Buyer clears customs. |
| **DDP** | Delivered Duty Paid | Goods delivered to buyer, all costs and duties paid | Seller | Seller | Seller | Seller's risk throughout | The seller handles everything, including import clearance and duty payment. The buyer has zero customs involvement. For e-commerce, DDP is increasingly demanded by consumers because it eliminates surprise charges. For the seller/platform, DDP requires accurate duty prediction at the point of sale. |

**Critical Clearance Impact:** The Incoterm determines the customs value basis. The US assesses duties on the FOB (transaction) value -- meaning international freight and insurance are NOT included. The EU assesses duties on the CIF value -- meaning freight and insurance ARE included. A $10,000 CIF shipment to the EU with $2,000 in freight and $200 in insurance has a customs value of $10,000 in the EU but only $7,800 in the US. This difference directly affects the duty amount. Getting the Incoterm wrong means getting the customs value wrong, which means getting the duty wrong.

---

### 1.4 Dates

#### Ship Date

- **What it is:** The date the shipment physically departs the origin (pickup from shipper or acceptance at carrier facility).
- **Who creates it:** The carrier, at the time of pickup/acceptance.
- **When created:** At pickup.
- **Mutable:** No (historical fact once it occurs).
- **Why it matters:** Ship date starts the clock for advance filing requirements (ACAS must be submitted before departure for air cargo). It also determines which tariff rates apply -- if a tariff change takes effect on a specific date, the ship date (or more precisely, the date of export) determines whether the old or new rate applies.

#### Estimated Delivery Date (EDD)

- **What it is:** The carrier's projected date of delivery to the consignee.
- **Who creates it:** The carrier's routing/planning system, based on service type, origin-destination, and transit time standards.
- **When created:** At booking or label creation.
- **Mutable:** Yes -- updated as the shipment moves through the network, and especially when customs events (holds, exams) occur.
- **Why it matters:** EDD is the customer's expectation. When customs clearance takes longer than planned, the EDD must be updated. The gap between the original EDD and the actual delivery date is the most visible measure of clearance performance to the shipper and consignee.

#### Entry Date

- **What it is:** The date the customs entry is filed with the customs authority (e.g., CBP in the US).
- **Who creates it:** The customs broker or carrier filing the entry.
- **When created:** When the entry is electronically transmitted to ACE (or equivalent system).
- **Mutable:** No.
- **Why it matters:** Entry date starts the liquidation clock. In the US, CBP has approximately 314 days from the entry date to liquidate (finalize) the entry. The entry date also determines the applicable exchange rate for duty calculation (CBP uses the exchange rate in effect on the date of export, which is typically close to the entry date for express shipments).

#### Arrival Date

- **What it is:** The date the goods physically arrive at the port of entry in the destination country.
- **Who creates it:** The carrier, based on the transport schedule and actual arrival.
- **When created:** At actual arrival (or estimated before arrival).
- **Mutable:** Estimated arrival is mutable; actual arrival is not.
- **Why it matters:** For ocean shipments, the arrival date triggers the ISF filing deadline (ISF must be filed 24 hours before loading at foreign port for ocean, but in practice customs processing begins when the vessel arrives). For express, arrival triggers the clearance processing window. Goods that arrive without a filed entry go into "general order" after 15 days.

#### Export Date

- **What it is:** The date goods are exported from the country of origin (loaded on the departing vessel/aircraft).
- **Who creates it:** Determined by the actual departure of the conveyance.
- **When created:** At departure.
- **Mutable:** No.
- **Why it matters:** The export date determines which tariff rates and exchange rates apply. If a tariff increase takes effect on March 1 and goods are exported on February 28, the lower rate applies. This is why tariff announcements often trigger "rush to ship" behaviors. The export date is a legally significant date that can be audited.

---

### 1.5 Weights and Dimensions

#### Gross Weight

- **What it is:** The total weight of the shipment including all packaging, pallets, crating, and the goods themselves. Measured in kilograms (international standard) or pounds.
- **Who creates it:** The shipper at origin (declared) and/or the carrier (measured at acceptance).
- **When created:** At pickup/acceptance.
- **Mutable:** Can be corrected if the shipper's declared weight differs from the carrier's measured weight.
- **Why it matters:** Gross weight is required on customs declarations, bills of lading, and air waybills. CBP uses gross weight as a risk indicator (weight anomalies relative to product type can flag suspicion). For specific-rate duties (duties calculated per kilogram rather than per dollar of value), gross or net weight directly determines the duty amount.

#### Net Weight

- **What it is:** The weight of the goods only, excluding all packaging and containers.
- **Who creates it:** The shipper, typically declared on the commercial invoice and packing list.
- **When created:** At order fulfillment or shipment preparation.
- **Mutable:** Only if the original declaration was in error.
- **Why it matters:** Net weight is used for duty calculation on specific-rate tariff lines (e.g., "$0.42 per kilogram"). It is also used for statistical reporting to the Census Bureau (the statistical unit of measure for many HS codes is kilograms). Discrepancy between net weight and gross weight should be plausible given the product type -- a 100 kg gross shipment with 5 kg net weight would be unusual for most products and could trigger scrutiny.

#### Tare Weight

- **What it is:** The weight of packaging, pallets, and containers only (gross weight minus net weight).
- **Who creates it:** Calculated from gross and net weight, or declared separately.
- **When created:** At shipment preparation.
- **Mutable:** Only if component weights change.
- **Why it matters:** Tare weight matters for container shipping (the tare weight of the container itself is stamped on the container and must be accounted for in weight calculations) and for products where packaging weight is significant relative to product weight.

#### Chargeable Weight

- **What it is:** The weight used by the carrier for pricing. It is the GREATER of the actual gross weight or the dimensional (volumetric) weight.
- **Formula:** Dimensional weight = (Length x Width x Height) / dimensional factor. The dimensional factor varies by carrier and mode (e.g., 5000 for air freight in cm/kg, 6000 for some carriers).
- **Who creates it:** The carrier, calculated from dimensions and actual weight.
- **When created:** At acceptance or measurement.
- **Mutable:** Can be recalculated if dimensions or weight are remeasured.
- **Why it matters:** Chargeable weight does not directly affect customs clearance (customs uses actual weight, not dimensional weight), but it is critical for freight cost, which in turn affects landed cost calculations under certain Incoterms. For CIF/CIP shipments, the freight cost (based on chargeable weight) is part of the customs value in EU/many jurisdictions.

#### Dimensional Weight

- **What it is:** A calculated weight based on the package dimensions, reflecting the space the package occupies rather than how much it weighs. Low-density goods (large, light packages) will have a dimensional weight much higher than actual weight.
- **Who creates it:** Carrier systems, from measured dimensions.
- **When created:** At acceptance.
- **Mutable:** Can be recalculated on remeasurement.
- **Why it matters:** Primarily a commercial/pricing attribute, but relevant to clearance when freight cost affects customs value.

---

### 1.6 Value and Currency

#### Total Declared Value

- **What it is:** The value of the goods as declared by the shipper for customs purposes. This is typically the transaction value -- the price actually paid or payable for the goods.
- **Who creates it:** The shipper/exporter, on the commercial invoice.
- **When created:** At invoicing, before shipment.
- **Mutable:** Should not be changed after filing, except through formal amendment. CBP can adjust the value during entry review or post-entry audit.
- **Why it matters:** This is the base on which ad valorem duties are calculated. If the declared value is $10,000 and the duty rate is 25%, the duty is $2,500. Understating the value reduces duty owed (but is fraud -- penalties up to 4x the lost revenue). Overstating the value increases duty unnecessarily. For related-party transactions (buyer and seller are corporate affiliates), CBP scrutinizes whether the transaction value reflects an arm's length price. Valuation disputes are among the most common and financially significant customs issues.

#### Transaction Currency

- **What it is:** The currency in which the commercial transaction was conducted (the currency on the commercial invoice).
- **Format:** ISO 4217 three-letter code (USD, EUR, GBP, CNY, JPY, etc.).
- **Who creates it:** Determined by the commercial agreement between buyer and seller.
- **When created:** At the time of sale.
- **Mutable:** No (fixed by the invoice).
- **Why it matters:** If the transaction currency is not the local currency of the importing country, the value must be converted for duty calculation. CBP publishes official exchange rates (sourced from the Federal Reserve/Treasury) that must be used for conversion. The applicable rate is the rate in effect on the date of export, NOT the rate on the filing date or the market rate. Using the wrong exchange rate is a compliance error.

#### Insurance Value

- **What it is:** The value for which the goods are insured during transport.
- **Who creates it:** The shipper or freight forwarder, based on the insurance policy.
- **When created:** At booking or insurance procurement.
- **Mutable:** Can be adjusted before shipment.
- **Why it matters:** For CIF/CIP Incoterms, the insurance cost is part of the customs value in jurisdictions that use CIF valuation (EU, many Asian countries). In the US (FOB valuation), insurance cost is NOT included in customs value. Understanding whether insurance is part of the value is essential for accurate duty calculation.

#### Freight Charges

- **What it is:** The cost of transporting the goods from origin to destination (or to the port of import).
- **Who creates it:** The carrier or freight forwarder.
- **When created:** At booking (quoted) and finalized after delivery.
- **Mutable:** Estimated at booking; finalized after shipment.
- **Why it matters:** Same as insurance -- in CIF-valuation jurisdictions, freight is part of the customs value. In the US, international freight from the port of export to the port of entry is NOT included in customs value, but domestic freight from the port of entry to the final destination IS excluded. Understanding the freight component and its relationship to the Incoterm and the jurisdiction's valuation method is essential.

---

### 1.7 Package Information

#### Number of Pieces / Packages

- **What it is:** The total count of individual packages (cartons, pallets, crates, drums, etc.) in the shipment.
- **Who creates it:** The shipper, declared on the shipping documents.
- **When created:** At shipment preparation.
- **Mutable:** Only if an error is discovered.
- **Why it matters:** Piece count is a fundamental customs data element. Discrepancies between declared piece count and actual count at the border trigger holds. For ACAS filing, piece count is one of the minimum data elements required before aircraft departure. Piece count affects exam logistics -- a 1-piece express parcel is trivial to examine; a 500-piece palletized freight shipment requires significant staging.

#### Package Type

- **What it is:** The type of outer packaging (carton, pallet, drum, crate, envelope, tube, etc.).
- **Format:** UN/CEFACT packaging type codes or carrier-specific codes.
- **Who creates it:** The shipper.
- **When created:** At shipment preparation.
- **Mutable:** No.
- **Why it matters:** Package type can be a risk indicator. Certain package types are associated with higher-risk goods (drums with chemicals, crates with machinery). It also affects handling requirements and exam feasibility.

---

### 1.8 Consolidation Indicators

#### Consolidation Flag

- **What it is:** Indicates whether the shipment is part of a consolidation (multiple shippers' goods grouped together for transport under a single MAWB or master B/L).
- **Values:** Yes/No, or a consolidation reference number.
- **Who creates it:** The freight forwarder or carrier.
- **When created:** At consolidation planning.
- **Mutable:** No.
- **Why it matters:** Consolidated shipments require de-consolidation at destination before individual consignments can be cleared. Each HAWB within a consolidation may require its own customs entry. Consolidation adds a processing step and complicates exam logistics -- if one consignment in a consolidation is selected for exam, the entire consolidated unit may need to be broken down.

---

### 1.9 Special Handling Codes

#### Hazmat / Dangerous Goods Classification

- **What it is:** Indicates the shipment contains dangerous goods as defined by IATA (air), IMDG (ocean), or DOT (ground) regulations. Includes the UN number, proper shipping name, hazard class, and packing group.
- **Format:** UN Number (4-digit, e.g., UN3481 for lithium-ion batteries), Hazard Class (1-9), Packing Group (I, II, III).
- **Who creates it:** The shipper, who must properly classify dangerous goods.
- **When created:** At shipment preparation. The shipper must complete a Shipper's Declaration for Dangerous Goods.
- **Mutable:** No -- once tendered, the DG classification is fixed.
- **Why it matters for clearance:** Dangerous goods have additional regulatory requirements -- EPA, DOT, and sometimes DEA involvement. PGA holds for hazmat shipments are common. Incorrect DG declaration can result in carrier penalties, aircraft groundings, and criminal liability. CBP and PGAs use DG indicators for targeting.

#### Perishable Indicator

- **What it is:** Flags the shipment as containing time-sensitive, perishable goods (food, flowers, pharmaceuticals, biological samples).
- **Who creates it:** The shipper or carrier at booking.
- **When created:** At booking.
- **Mutable:** No.
- **Why it matters for clearance:** Perishable shipments have priority processing requirements -- they deteriorate in cage/hold situations. FDA prior notice requirements apply to food shipments (must be filed 2-24 hours before arrival). USDA/APHIS may require phytosanitary certificates. Clearance delay for perishables can result in total loss of the goods, creating significant financial and waste implications.

#### High-Value Indicator

- **What it is:** Flags shipments exceeding a carrier-defined value threshold (varies, but typically above $50,000-$100,000 for express).
- **Who creates it:** The carrier's system, based on declared value.
- **When created:** At acceptance or data entry.
- **Mutable:** If value is corrected.
- **Why it matters for clearance:** High-value shipments receive enhanced scrutiny from customs (valuation review, potential for higher bond requirements). They may also require enhanced security handling and insurance coverage. CBP may request supporting documentation (purchase contracts, wire transfer records) for high-value entries.

#### Temperature-Controlled Indicator

- **What it is:** Indicates the shipment requires temperature maintenance during transit (cold chain, frozen, controlled ambient).
- **Who creates it:** The shipper at booking.
- **When created:** At booking.
- **Mutable:** No.
- **Why it matters for clearance:** Temperature excursion during a customs hold can render goods unusable (pharmaceuticals, vaccines, food). This creates urgency for customs clearance and limits how long goods can remain in cage. Facilities must have appropriate temperature-controlled storage for held shipments.

#### Fragile Indicator

- **What it is:** Indicates the shipment contains fragile goods requiring careful handling.
- **Who creates it:** The shipper.
- **When created:** At shipment preparation.
- **Mutable:** No.
- **Why it matters for clearance:** Primarily affects physical handling during exam. A VACIS/X-ray exam is preferred over an intensive physical exam for fragile goods, but this is at CBP's discretion.

---

## 2. Party Records

Every international shipment involves multiple parties, each with distinct roles and regulatory obligations. Accurate identification of each party is essential for customs filing, compliance screening (denied party screening), duty assessment, and communication.

---

### 2.1 Shipper / Exporter

#### Name and Address (Structured)

- **Fields:** Legal entity name, Street address (line 1, line 2), City, State/Province, Postal code, Country (ISO 3166-1 alpha-2).
- **Who creates it:** The shipper provides this information at booking/label creation.
- **When created:** At booking.
- **Mutable:** Can be corrected before filing; after filing, corrections require amendment.
- **Why it matters:** The shipper/exporter is one of the parties screened against restricted party lists (OFAC SDN, BIS Entity List, EU sanctions lists). If the shipper matches a restricted party, the shipment cannot proceed without a license or exemption. The shipper's address determines the country of export (which may differ from the country of origin of the goods). The shipper's identity is a key data element for advance filing (ACAS, ISF).

#### Known Shipper Status

- **What it is:** Indicates whether the shipper has been vetted and approved under a Known Shipper Program (TSA's Known Shipper Management System in the US, similar programs in other jurisdictions).
- **Values:** Known Shipper / Unknown Shipper.
- **Who creates it:** The carrier or indirect air carrier, based on TSA vetting.
- **When created:** During the shipper's initial engagement with the carrier; maintained ongoing.
- **Mutable:** Yes -- status can be revoked if the shipper fails to meet requirements.
- **Why it matters:** Unknown shippers' air cargo is subject to additional physical screening before being loaded on passenger aircraft. This is a security requirement, not a customs requirement, but it affects processing time and cost for air shipments. Known shipper status reduces screening burden and processing time.

---

### 2.2 Consignee / Importer

#### Name and Address (Structured)

- **Fields:** Same structured format as shipper (legal name, street, city, state/province, postal code, country).
- **Who creates it:** The shipper at booking, based on the commercial transaction.
- **When created:** At booking.
- **Mutable:** Address corrections are common (incorrect zip codes, apartment numbers), but changing the consignee entity after filing requires formal amendment.
- **Why it matters:** The consignee is typically the Importer of Record (IOR) and is the party legally responsible for customs compliance, duty payment, and all regulatory obligations. The consignee is screened against denied party lists. The consignee's address determines the final delivery point and may affect which port of entry is used.

#### Tax ID / EIN / VAT Number

- **What it is:** The consignee/importer's tax identification number in the destination country.
  - **US:** Employer Identification Number (EIN/IRS number) or Social Security Number (SSN) for individual importers.
  - **EU:** Value Added Tax (VAT) registration number (format: country code + 8-12 alphanumeric characters).
  - **UK:** VAT number or EORI number.
  - **Other:** Country-specific tax identification numbers.
- **Who creates it:** Issued by the destination country's tax authority. The importer provides it to the broker/carrier.
- **When created:** At business registration.
- **Mutable:** No (fixed by the tax authority).
- **Why it matters for clearance:** The IRS number / EIN is REQUIRED on every US customs entry. It is the key that identifies the Importer of Record in ACE and links all of that importer's entries for history, risk assessment, and audit purposes. CBP's risk targeting algorithms use the IOR's tax ID to assess the importer's compliance history. An importer with a history of violations, unpaid duties, or UFLPA detentions will see elevated targeting on all future entries. Without a valid tax ID, the entry cannot be filed.

#### EORI Number (EU/UK)

- **What it is:** Economic Operators Registration and Identification number. A unique identifier for businesses engaged in customs activities in the EU or UK.
- **Format:** Country code + up to 15 characters (e.g., DE123456789012345 for Germany).
- **Who creates it:** Assigned by the customs authority of the EU member state where the business is established.
- **When created:** At registration.
- **Mutable:** No.
- **Why it matters:** EORI is mandatory for any entity filing customs declarations in the EU/UK. It is the equivalent of the US IRS number for customs purposes. Without an EORI, goods cannot be cleared in the EU. The EORI links to the entity's AEO (Authorized Economic Operator) status, customs history, and regulatory profile.

#### Importer of Record (IOR) Number

- **What it is:** The CBP-assigned number that identifies the Importer of Record. In the US, this is typically the IRS EIN or, for entities without an EIN, a CBP-assigned number.
- **Who creates it:** CBP or the IRS.
- **When created:** At registration.
- **Mutable:** No.
- **Why it matters:** The IOR bears ultimate legal responsibility for the accuracy of the customs entry, payment of duties, and compliance with all import laws. The IOR number is the anchor for CBP's account-based processing vision (21CCF) -- all of an importer's entries, compliance history, and risk profile are indexed by this number.

---

### 2.3 Notify Party

#### Name and Contact Information

- **What it is:** The party that should be notified upon arrival of the goods at the destination port. Often used in ocean freight where the consignee may not be at the port.
- **Who creates it:** The shipper, specified on the bill of lading.
- **When created:** At booking.
- **Mutable:** Can be amended.
- **Why it matters:** For ocean freight, the notify party receives the arrival notice and arranges for customs clearance and cargo pickup. If the notify party information is wrong, no one responds to the arrival notice, and the goods sit at the port accumulating demurrage and storage charges. After 15 days without entry, goods go to general order. The notify party is also screened against denied party lists.

---

### 2.4 Customs Broker

#### ABI Filer Code

- **What it is:** Automated Broker Interface (ABI) filer code -- a three-character alphanumeric code assigned by CBP to each licensed customs broker or self-filing importer authorized to file entries electronically through ACE.
- **Format:** 3 characters (letter + 2 digits, e.g., A01).
- **Who creates it:** Assigned by CBP.
- **When created:** At broker registration for ABI access.
- **Mutable:** No.
- **Why it matters:** The ABI filer code identifies who filed the entry. CBP tracks filer performance metrics (accuracy, timeliness, error rates) by filer code. A filer with poor metrics may face increased scrutiny on all entries. The filer code is also used for communication -- CBP sends entry status messages (release, hold, exam orders, requests for information) to the ABI filer code.

#### Customs Broker License Number

- **What it is:** The license number issued by CBP authorizing an individual or entity to transact customs business on behalf of importers.
- **Who creates it:** CBP, upon passing the Customs Broker License Examination and meeting other requirements.
- **When created:** At license issuance.
- **Mutable:** No (though licenses can be suspended or revoked).
- **Why it matters:** Only licensed customs brokers can file entries on behalf of importers in the US. The broker accepts legal responsibility for entries filed under their license. CBP can audit, fine, suspend, or revoke broker licenses for compliance failures.

#### Power of Attorney (POA) Reference

- **What it is:** A legal document in which the importer authorizes a specific customs broker to act on their behalf for customs purposes.
- **Who creates it:** Executed by the importer and accepted by the broker.
- **When created:** Before the first entry is filed. Can be a transaction-specific POA or a blanket POA covering all entries.
- **Mutable:** Can be revoked by the importer.
- **Why it matters:** CBP requires that the broker have a valid POA from the importer before filing an entry. Filing without a POA is a violation of customs regulations and can result in penalties against the broker. The POA also defines the scope of the broker's authority -- some POAs limit the broker to filing entries, while others authorize the broker to make binding commitments (FTA claims, drawback elections) on the importer's behalf.

---

### 2.5 Bond Information

#### Bond Type

- **What it is:** The type of customs bond securing the importer's obligation to pay duties, taxes, and fees.
- **Values:**
  - **Continuous Bond (Activity Code 1):** Covers all entries filed by the importer during the bond period (typically one year, renewed annually). Required for importers who file more than occasional entries.
  - **Single-Entry Bond (Activity Code 1 -- single transaction):** Covers a single entry only. Used for occasional importers.
  - **Other Activity Codes:** 2 (custodian of bonded merchandise), 3 (international carrier), 3a (instrument of international traffic), 4 (foreign trade zone operator), etc.
- **Who creates it:** Issued by a surety company, applied for by the importer through their broker.
- **When created:** Before the first entry is filed.
- **Mutable:** Bond amount can be increased or decreased. Bond can be terminated.
- **Why it matters:** A valid customs bond is REQUIRED before any formal entry can be filed in the US. The bond guarantees that the importer will pay all duties, taxes, fees, and penalties. If the importer defaults, CBP collects from the surety. Bond sufficiency is a live issue -- if an importer's duty obligations increase dramatically (due to tariff stacking), their existing bond may be insufficient, and CBP can refuse to release goods until the bond is increased. This has been a significant operational issue since IEEPA tariffs increased effective duty rates to 100%+ on many products.

#### Surety Company

- **What it is:** The insurance company that issues the customs bond and guarantees the importer's obligations.
- **Format:** 3-digit surety code assigned by Treasury.
- **Who creates it:** Treasury Department maintains the list of acceptable sureties.
- **When created:** At bond issuance.
- **Mutable:** Importer can change sureties by terminating one bond and obtaining another.
- **Why it matters:** CBP tracks surety reliability. If a surety has high claim rates, CBP may increase scrutiny on entries covered by that surety's bonds.

#### Bond Amount

- **What it is:** The dollar amount of the bond, representing the maximum amount the surety will pay to CBP if the importer defaults.
- **Minimum:** For continuous bonds, the minimum is $50,000 or 10% of the total duties, taxes, and fees paid in the prior year, whichever is greater.
- **Who creates it:** Determined by the surety based on the importer's volume and risk profile.
- **When created:** At bond issuance; reviewed and adjusted periodically.
- **Mutable:** Yes -- can be increased or decreased.
- **Why it matters:** Bond sufficiency is actively monitored by CBP. If an importer's annual duty payments exceed the bond amount, CBP can issue a bond insufficiency notice and refuse to release goods until the bond is increased. With tariff rates now exceeding 100% on some products, importers who previously needed a $50,000 bond may now need $500,000 or more.

---

### 2.6 Carrier

#### Carrier Name and SCAC

- **See Section 1.2** (Carrier SCAC Code above).

#### Conveyance Name / Flight Number / Vessel Name

- **What it is:** The specific aircraft (by flight number), vessel (by name/IMO number), or truck (by plate number) carrying the goods.
- **Who creates it:** The carrier.
- **When created:** At dispatch/departure.
- **Mutable:** Can change due to rebooking, vessel substitution, etc.
- **Why it matters:** CBP's manifest system tracks goods by conveyance. The entry is linked to a specific conveyance and must match the manifest data. Discrepancies between entry data and manifest data trigger holds.

---

### 2.7 Freight Forwarder

#### Forwarder Name and FMC Number

- **What it is:** The freight forwarder's identity and their Federal Maritime Commission (FMC) license or bond number (for ocean freight) or their forwarding agent information.
- **Who creates it:** FMC (for ocean); the forwarder provides their information.
- **When created:** At forwarder registration.
- **Mutable:** No.
- **Why it matters:** For ISF 10+2 filing, the identity of the party who stuffed the container and the consolidator are required data elements. The forwarder often fills these roles. For export, the forwarding agent is identified on the EEI (Electronic Export Information) filing.

---

## 3. Line Item / Commodity Detail

The line item is where the regulatory rubber meets the road. Each distinct commodity in a shipment must be separately classified, valued, and assessed for duty and compliance requirements. A single shipment can have dozens of line items, each with its own HS code, duty rate, country of origin, and PGA requirements.

---

### 3.1 Product Description

#### Commodity Description (Free-Text)

- **What it is:** A written description of the goods sufficient for customs classification.
- **Who creates it:** The shipper (on the commercial invoice) and/or the customs broker (on the entry).
- **When created:** At invoicing (shipper) or entry preparation (broker).
- **Mutable:** Can be corrected before filing; after filing, requires amendment.
- **Why it matters:** THIS IS THE MOST IMPORTANT FREE-TEXT FIELD IN THE ENTIRE PROCESS. Customs authorities use the description to verify that the declared HS code is correct. A vague description ("electronics," "clothing," "parts") provides no basis for classification verification and will trigger a request for information or a hold. A specific description ("100% cotton, men's woven dress shirts, long sleeve, collar, button-front, style #4452") enables accurate classification and reduces scrutiny. CBP has issued guidance that descriptions must include: what the article is, what it is made of, and its intended use. For FDA-regulated products, the description must include the FDA product code and common or usual name. The quality of the commodity description is the single strongest predictor of whether a shipment will clear without exception.

---

### 3.2 Classification

#### HS Code (Harmonized System Code)

- **What it is:** The internationally standardized numeric code that classifies traded products. Maintained by the World Customs Organization (WCO).
- **Structure:**
  - **Digits 1-2:** Chapter (e.g., 87 = Vehicles other than railway)
  - **Digits 1-4:** Heading (e.g., 8708 = Parts and accessories of motor vehicles)
  - **Digits 1-6:** Subheading (e.g., 8708.30 = Brakes and servo-brakes and parts thereof). The first 6 digits are internationally harmonized -- every WCO member uses the same 6-digit codes.
  - **Digits 7-8:** Country-specific subdivision (e.g., 8708.30.50 in the US). These vary by country. The US 8-digit code differs from the EU 8-digit code, even for the same product.
  - **Digits 9-10 (US only):** Statistical suffix (e.g., 8708.30.5060). Used for statistical reporting to the Census Bureau. Does not affect duty rate but is required on US entries.
- **Who creates it:** The customs broker or shipper, based on the product description and General Rules of Interpretation (GRI).
- **When created:** At entry preparation (ideally before shipment for pre-clearance).
- **Mutable:** Can be corrected by amendment. CBP can reclassify at any time during or after the entry process.
- **Why it matters:** The HS code is the SINGLE MOST CONSEQUENTIAL DATA ELEMENT in customs clearance. It determines:
  - The duty rate (MFN rate, ranging from 0% to 30%+).
  - Whether special tariff programs apply (Section 301 lists are defined by HS code; Section 232 scope is defined by HS code).
  - Which PGA requirements apply (FDA product codes map from HS codes; CPSC, EPA, FCC jurisdiction is determined by HS code).
  - Whether FTA preferences are available (rules of origin are defined at the HS heading/subheading level).
  - Whether AD/CVD orders apply (AD/CVD scope is defined by product description and HS code).
  - Statistical reporting categories.

  A classification error cascades through every downstream process. If the HS code is wrong, the duty rate is wrong, the PGA screening is wrong, the FTA eligibility check is wrong, and the AD/CVD check is wrong. CBP estimates that up to 20% of entries contain classification errors. With tariff stacking, the financial impact of a misclassification can be enormous -- a product classified to an HS code covered by Section 301 + IEEPA could face 170%+ duty, while the same product classified one digit differently might face 2.5%.

---

### 3.3 Origin

#### Country of Origin

- **What it is:** The country where the goods were manufactured, produced, or grown. For goods that undergo processing in multiple countries, the country of origin is where the goods underwent their last "substantial transformation" -- a change in the character, name, or use of the article.
- **Format:** ISO 3166-1 alpha-2 code (e.g., CN, DE, MX, JP).
- **Who creates it:** Declared by the shipper/exporter, verified by the customs broker.
- **When created:** At invoicing.
- **Mutable:** Can be challenged and changed by CBP based on evidence.
- **Why it matters:** Country of origin determines:
  - Whether country-specific tariff programs apply (Section 301 applies to CN-origin goods; IEEPA rates are country-specific; AD/CVD orders are country-specific).
  - The applicable duty rate column (MFN rates apply to "normal trade relations" countries; Column 2 rates -- dramatically higher -- apply to a few non-NTR countries like Cuba, North Korea).
  - FTA eligibility (goods must originate in an FTA partner country).
  - Marking requirements (goods must be marked with their country of origin for the ultimate purchaser to see).
  - UFLPA risk (goods originating in or with components from Xinjiang are subject to the forced labor rebuttable presumption).

  Origin is one of the most disputed areas of customs law. "Substantial transformation" is a fact-specific legal test, not a formula. A product assembled in Vietnam from Chinese components may be considered of Vietnamese origin (if the assembly constitutes substantial transformation) or Chinese origin (if it does not). The stakes of this determination have never been higher given the differential between China-specific tariff rates and rates for other countries.

#### Country of Export

- **What it is:** The country from which the goods were physically shipped to the destination country. This may differ from the country of origin if goods were transshipped or warehoused in a third country.
- **Format:** ISO 3166-1 alpha-2.
- **Who creates it:** The shipper/carrier, based on the actual shipment routing.
- **When created:** At shipment.
- **Mutable:** No (factual).
- **Why it matters:** Country of export is required on customs declarations and is used for risk assessment. A discrepancy between country of origin (CN) and country of export (VN) raises questions -- are the goods genuinely of Vietnamese origin, or were Chinese goods transshipped through Vietnam to evade China-specific tariffs? CBP's EAPA (Enforce and Protect Act) investigations specifically target transshipment to evade AD/CVD duties.

#### Country of Manufacture

- **What it is:** The country where the goods were actually manufactured or produced. Often the same as country of origin, but can differ when substantial transformation rules produce a different origin determination.
- **Who creates it:** Declared by the shipper.
- **When created:** At invoicing.
- **Mutable:** Can be challenged.
- **Why it matters:** For Section 232 (steel and aluminum), the relevant origin is "country of melt and pour" (steel) or "country of smelt and cast" (aluminum), which may differ from the country of manufacture of the finished product. A steel product manufactured in Mexico from Chinese steel retains Chinese origin for Section 232 purposes.

---

### 3.4 Quantity and Value per Line Item

#### Quantity

- **What it is:** The number of units of the commodity in the line item.
- **Who creates it:** The shipper, declared on the commercial invoice.
- **When created:** At invoicing.
- **Mutable:** Can be corrected.
- **Why it matters:** Quantity is used for duty calculation on specific-rate tariff lines, for statistical reporting, and for reasonableness checks (does the declared value per unit make sense given the product?).

#### Unit of Measure (Commercial vs. Statistical)

- **What it is:** The unit in which quantity is measured. There are two distinct units that may differ:
  - **Commercial unit:** The unit used in the transaction (pieces, sets, pairs, cartons, etc.).
  - **Statistical unit:** The unit required by customs for statistical reporting, defined in the HTSUS for each tariff line (typically kilograms, liters, dozens, square meters, etc.).
- **Who creates it:** Commercial unit by the shipper; statistical unit determined by the HTSUS.
- **When created:** At invoicing (commercial) and entry preparation (statistical).
- **Mutable:** Can be corrected.
- **Why it matters:** If the HTSUS requires quantity in kilograms but the shipper invoices in pieces, the broker must convert. For specific-rate duties ("$0.42 per kg"), the statistical quantity in the correct unit is essential for accurate duty calculation.

#### Unit Value and Total Line Value

- **What it is:** The price per unit and total value (unit value x quantity) for the line item.
- **Who creates it:** The shipper, on the commercial invoice.
- **When created:** At invoicing.
- **Mutable:** Can be corrected.
- **Why it matters:** Line item value is the basis for ad valorem duty calculation at the line level. If a shipment has multiple line items with different HS codes and duty rates, each line item's value determines how much duty accrues to that tariff line.

---

### 3.5 Material and Manufacturing

#### Material Composition

- **What it is:** The breakdown of materials that make up the product, typically expressed as percentages by weight.
- **Example:** "97% cotton, 3% spandex" or "Steel body, rubber gasket, plastic handle."
- **Who creates it:** The manufacturer or shipper.
- **When created:** At product specification; declared on the invoice or packing list.
- **Mutable:** Not for a given product.
- **Why it matters:** Material composition is critical for classification. A shirt that is 51% cotton is classified differently from one that is 51% polyester. Chapter 11 (milling products) vs. Chapter 19 (food preparations) hinges on processing level. Material composition also determines Section 232 applicability (is the product steel or aluminum, or does it merely contain steel/aluminum components?).

#### Manufacturer Identification (MID) Code

- **What it is:** A coded identifier for the manufacturer of the goods, required on all US customs entries since 2009.
- **Format:** Constructed from: country of origin ISO code (2 chars) + first 3 characters of manufacturer name + first 3 characters of city + first 4 characters of street address. Example: CNDONYU1234 for a manufacturer named "Yuxing" in Dongguan at "1234 Industrial Blvd."
- **Who creates it:** Constructed by the customs broker from manufacturer information provided by the shipper.
- **When created:** At entry preparation.
- **Mutable:** Should not change for a given manufacturer, but errors in construction are common.
- **Why it matters:** CBP uses the MID to track which manufacturers' goods are entering the US. Manufacturer-level risk profiles are maintained -- a manufacturer associated with forced labor, counterfeiting, or AD/CVD evasion will trigger enhanced scrutiny on all future shipments. The MID is also used for UFLPA enforcement -- CBP can identify all US imports from a specific manufacturer and assess whether that manufacturer has Xinjiang supply chain exposure. The MID format is imprecise (collisions are common for manufacturers with similar names in the same city), but it is the best manufacturer-level identifier available on customs entries.

---

### 3.6 Trade Remedy and Special Program Indicators

#### Anti-Dumping / Countervailing Duty (AD/CVD) Case Number

- **What it is:** The Commerce Department case number identifying the specific AD/CVD order that applies to the goods.
- **Format:** Typically in the form "A-xxx-xxx" (antidumping) or "C-xxx-xxx" (countervailing duty), where xxx-xxx is a country and case identifier.
- **Who creates it:** The Department of Commerce issues the order; the broker identifies the applicable case number.
- **When created:** At entry preparation.
- **Mutable:** Can be corrected.
- **Why it matters:** AD/CVD duties can be enormous -- rates exceeding 200% are not uncommon for specific manufacturers. The case number links the entry to the specific order, which determines the applicable rate. AD/CVD rates are company-specific: a manufacturer with a separately calculated rate may pay 5%, while the "all others" rate for the same product from the same country may be 150%. Correctly identifying the case number AND the manufacturer-specific rate is essential. Errors here are a major source of underpayment penalties and EAPA investigations.

#### Section 301 / 232 / IEEPA Indicators

- **What it is:** Flags indicating that the line item is subject to special tariff programs.
- **Values:** True/False, with the specific list number or program identifier (e.g., "Section 301, List 3" or "Section 232 -- Steel").
- **Who creates it:** Determined by the classification engine or broker based on the HS code and country of origin.
- **When created:** At entry preparation or tariff calculation.
- **Mutable:** Changes when tariff programs are modified (new lists, exclusions, pauses).
- **Why it matters:** These indicators determine whether additional tariff layers are stacked on top of the MFN rate. Missing a Section 301 indicator means underpaying duty (fraud risk). Incorrectly applying a Section 301 indicator means overpaying duty (financial loss). With IEEPA rates at 145% for China, the financial impact of getting this wrong is massive.

#### Trade Agreement Eligibility Flags

- **What it is:** Indicates whether the line item qualifies for preferential duty treatment under a Free Trade Agreement (USMCA, KORUS, US-Australia, US-Singapore, etc.).
- **Values:** Eligible / Not Eligible / Claimed / Not Claimed.
- **Who creates it:** The importer or broker, based on rules of origin analysis.
- **When created:** At entry preparation.
- **Mutable:** Yes -- an importer can claim an FTA preference on a post-entry basis (within timeframes allowed by each FTA).
- **Why it matters:** An eligible-but-unclaimed FTA preference means the importer is paying duty they do not owe. Industry estimates suggest 15-30% of FTA-eligible trade goes unclaimed, representing billions of dollars in unnecessary duty payments annually. Conversely, claiming an FTA preference without proper documentation (certificate of origin, rules of origin analysis) exposes the importer to penalties if audited.

#### Related Party Transaction Indicator

- **What it is:** A flag indicating whether the buyer and seller are related parties (parent-subsidiary, corporate affiliates, etc.) as defined by customs valuation law.
- **Values:** Yes/No.
- **Who creates it:** The importer, declared on the customs entry.
- **When created:** At entry preparation.
- **Mutable:** No (factual).
- **Why it matters:** Related-party transactions receive heightened valuation scrutiny from CBP. When the buyer and seller are related, CBP questions whether the transaction value (price paid) reflects a true arm's-length price or has been manipulated (typically understated) to reduce duties. The importer must demonstrate that the relationship did not influence the price, using test methods defined in the WTO Valuation Agreement. Approximately 40% of global trade is between related parties.

---

### 3.7 PGA-Specific Codes

#### FDA Product Code

- **What it is:** A 7-character code identifying the product for FDA regulatory purposes. More granular than the HS code for FDA-regulated products.
- **Format:** 2 characters (product area) + 2 characters (product) + 3 characters (product code). Example: "40MRI01" for "chocolate bars."
- **Who creates it:** The broker, based on the product description and FDA product code database.
- **When created:** At entry preparation.
- **Mutable:** Can be corrected.
- **Why it matters:** FDA uses the product code (not the HS code) to determine which FDA requirements apply. The product code drives: prior notice requirements, import alert screening, FSVP requirements, and labeling requirements. A wrong FDA product code can result in the product being flagged against an import alert that does not actually apply (false positive) or missing an alert that does apply (false negative, leading to consumer safety risk).

#### USDA Commodity Code

- **What it is:** APHIS (Animal and Plant Health Inspection Service) commodity codes identifying agricultural products for pest and disease risk assessment.
- **Who creates it:** The broker.
- **When created:** At entry preparation.
- **Mutable:** Can be corrected.
- **Why it matters:** USDA commodity codes determine whether a phytosanitary certificate is required, which pests are of concern, and what inspection or treatment protocols apply. Incorrect coding can result in holds, re-export, or destruction of agricultural products.

#### EPA Codes (TSCA, FIFRA)

- **What it is:** Codes or certifications required for products regulated by the Environmental Protection Agency under TSCA (Toxic Substances Control Act) or FIFRA (Federal Insecticide, Fungicide, and Rodenticide Act).
- **Who creates it:** The importer (positive or negative TSCA certification) or the broker.
- **When created:** At entry preparation.
- **Mutable:** Can be corrected.
- **Why it matters:** Chemical products (broadly defined -- includes plastics, adhesives, coatings, etc.) require a TSCA certification statement: either a positive certification ("all chemical substances in this shipment are listed on the TSCA inventory or exempt") or a negative certification ("this shipment contains chemicals not listed on the TSCA inventory and is imported under [specific exemption]"). Missing TSCA certification results in holds.

---

### 3.8 Marks and Numbers

#### Marks and Numbers

- **What it is:** The identifying marks, numbers, or labels on the outside of the packages/containers. These are the physical markings that allow a customs officer to match a declaration to a specific package.
- **Who creates it:** The shipper, applied to the outer packaging.
- **When created:** At packing.
- **Mutable:** No (applied physically).
- **Why it matters:** During a physical exam, CBP verifies that the marks and numbers on the packages match the declaration. Discrepancies trigger further investigation. Country of origin marking is legally required on all imported goods (19 USC 1304) -- goods must be marked with their country of origin in a manner that is conspicuous and legible to the ultimate purchaser. Failure to mark is a violation that can result in marking duties (10% of appraised value) or re-export.

---

## 4. Customs Declaration / Entry

The customs entry is the formal declaration to the customs authority, containing all the information needed to assess duties, verify compliance, and release the goods.

---

### 4.1 Entry Type

#### Entry Type Code

- **What it is:** A 2-digit code identifying the type of customs entry being filed.
- **Key values:**
  - **01 -- Consumption Entry:** The standard formal entry for goods entering US commerce. Required for all goods valued over $2,500 and, since de minimis suspension, effectively all imported goods. Full data required: 10-digit HTSUS, value, origin, IOR, bond.
  - **03 -- Consumption Entry -- Antidumping/Countervailing Duty:** Formal entry where AD/CVD duties apply. Triggers additional data requirements (case number, manufacturer-specific rate, cash deposit amount).
  - **06 -- Consumption Entry -- Foreign Trade Zone:** Entry of goods from an FTZ into US commerce. The importer may choose to pay duty on the goods in their FTZ-admitted condition or in their condition when withdrawn from the FTZ (whichever results in lower duty).
  - **11 -- Informal Entry:** Simplified entry for goods valued under $2,500 (where formal entry is not required). Reduced data requirements. Duty payable at time of entry.
  - **21 -- Warehouse Entry:** Goods placed in a bonded warehouse. Duties deferred until the goods are withdrawn for consumption.
  - **22 -- Re-Export Entry:** Goods withdrawn from a bonded warehouse for re-export. No duties owed.
  - **23 -- Temporary Importation under Bond (TIB):** Goods imported temporarily (for exhibition, testing, repair) that will be re-exported within a specified time. Duty deferred, secured by bond.
  - **26 -- Warehouse Entry -- FTZ:** Goods admitted to an FTZ and subsequently entered into a bonded warehouse.
  - **86 -- Entry Type 86 (DEPRECATED):** Previously used for Section 321 de minimis shipments with enhanced data. Effectively deprecated since the suspension of de minimis (August 29, 2025).
- **Who creates it:** The customs broker, based on the nature of the import.
- **When created:** At entry preparation.
- **Mutable:** Entry type can be changed by amendment in some circumstances (e.g., converting a TIB to a consumption entry if goods are not re-exported).
- **Why it matters:** Entry type determines: the data requirements, the duty payment timing, the bond requirements, the post-entry obligations, and the liquidation process. Choosing the wrong entry type can result in duty overpayment (using Type 01 when Type 06 FTZ would provide a lower duty) or compliance violations (using Type 11 when Type 01 is required).

---

### 4.2 Entry Identification

#### Entry Number

- **What it is:** A unique 11-character number identifying the customs entry.
- **Format:** 3-digit filer code + 7-digit entry number + 1-digit check digit.
- **Who creates it:** Generated by the broker's ABI system using CBP-provided entry number ranges.
- **When created:** At entry filing.
- **Mutable:** No.
- **Why it matters:** The entry number is the primary key for all CBP processing. It links to the duty payment (the 7501 entry summary), the release decision, any exam orders, protest filings, liquidation notices, and drawback claims. All post-entry activity (amendments, reconciliation, audit) references the entry number.

#### Port of Entry Code

- **What it is:** A 4-digit code identifying the CBP port where the goods arrive and the entry is processed.
- **Format:** 4 digits (e.g., 1001 for JFK, 2704 for Memphis, 2809 for Los Angeles Airport).
- **Who creates it:** Determined by the shipment routing; declared by the broker.
- **When created:** At entry preparation.
- **Mutable:** Can change if the shipment is rerouted.
- **Why it matters:** The port of entry determines which CBP officers and Center of Excellence and Expertise (CEE) process the entry. Different ports have different specializations (the Electronics CEE in Los Angeles, the Automotive CEE in Detroit, the Petroleum CEE in Houston). Port of entry also affects exam logistics and timing.

---

### 4.3 Duty Assessment

#### Estimated Duties, Taxes, and Fees

- **What it is:** The estimated amount of duties, taxes, and fees owed on the entry, calculated at the time of filing.
- **Who creates it:** Calculated by the broker's system based on the tariff schedule, applicable programs, and declared value.
- **When created:** At entry filing.
- **Mutable:** Yes -- this is an estimate. The final amount is determined at liquidation.
- **Why it matters:** The estimated duty amount is deposited with CBP (or charged against the bond) at the time of entry. The importer must pay this amount to obtain release of the goods. If the liquidated amount differs, the importer pays the difference or receives a refund. Accurate estimation is financially important -- overestimation ties up working capital; underestimation creates a liability.

#### Liquidated Duties

- **What it is:** The final, official duty amount determined by CBP at liquidation (approximately 314 days after entry date).
- **Who creates it:** CBP, at liquidation.
- **When created:** At liquidation (10 months + 14 days from entry date, approximately).
- **Mutable:** Can be protested within 180 days of liquidation. CBP can reliquidate within 90 days.
- **Why it matters:** Liquidation is the final word on what the importer owes. If the liquidated amount exceeds the deposited amount, the importer must pay the difference plus interest. If the liquidated amount is less, the importer receives a refund. Importers and brokers must monitor liquidation bulletins (published weekly) for each entry and contest adverse liquidations within the 180-day protest window. Failure to protest waives the right to challenge.

#### Merchandise Processing Fee (MPF)

- **What it is:** A fee assessed on all formal entries, calculated as a percentage of the customs value.
- **Rate:** 0.3464% of the entered value.
- **Minimum:** $31.67 per entry (FY2026).
- **Maximum:** $614.35 per entry (FY2026).
- **Who creates it:** Calculated by the broker/system at entry filing.
- **When created:** At entry filing.
- **Mutable:** Adjusted at liquidation if the value changes.
- **Why it matters:** MPF applies to every formal entry. The minimum and maximum create a floor and ceiling. For very high-value entries, the maximum cap means MPF is negligible relative to the value. For low-value formal entries (now common post-de minimis), the minimum fee can be significant relative to the goods' value ($31.67 on a $50 shipment is over 60%).

#### Harbor Maintenance Fee (HMF)

- **What it is:** A fee assessed on goods arriving by vessel (ocean shipments only).
- **Rate:** 0.125% of the customs value.
- **No minimum or maximum.
- **Who creates it:** Calculated at entry filing.
- **When created:** At entry filing.
- **Mutable:** Adjusted at liquidation.
- **Why it matters:** HMF applies ONLY to ocean shipments. Air freight and truck shipments are exempt. This creates a mode-of-transport cost difference that can influence routing decisions.

#### Duty Preference Program Code (SPI)

- **What it is:** The Special Program Indicator code identifying the preferential duty program being claimed.
- **Key values:**
  - **S** or **S+** -- USMCA (US-Mexico-Canada Agreement)
  - **KR** -- KORUS (Korea FTA)
  - **AU** -- US-Australia FTA
  - **SG** -- US-Singapore FTA
  - **Various others** for other FTAs, GSP (suspended), CBERA, ATPA, etc.
- **Who creates it:** The broker, when claiming a preference.
- **When created:** At entry filing.
- **Mutable:** Can be claimed post-entry within FTA-specific timeframes.
- **Why it matters:** The SPI tells CBP that the importer is claiming a preferential (reduced or zero) duty rate under a trade agreement. CBP audits FTA claims -- the importer must have supporting documentation (certificate of origin, rules of origin analysis) and be able to produce it on demand. A false FTA claim is a serious violation with penalties.

---

### 4.4 FTZ and Warehouse

#### Foreign Trade Zone Admission Number

- **What it is:** The number assigned when goods are admitted into a Foreign Trade Zone.
- **Who creates it:** The FTZ operator.
- **When created:** At FTZ admission.
- **Mutable:** No.
- **Why it matters:** Links the FTZ admission to the eventual consumption entry (Type 06). Enables tracking of goods within the FTZ and calculation of duty at the appropriate rate (either on the imported material or the finished product, whichever is lower -- "inverted tariff" benefit).

#### Reconciliation Flag

- **What it is:** Indicates that the entry is flagged for reconciliation -- meaning certain data elements (typically value, classification, or FTA eligibility) are not final at the time of filing and will be reconciled later.
- **Values:** Yes/No, with an indication of which elements are to be reconciled.
- **Who creates it:** The broker, at entry filing.
- **When created:** At entry filing.
- **Mutable:** The flag is set at filing; the reconciliation entry is filed later (within 21 months of the earliest entry in the reconciliation).
- **Why it matters:** Reconciliation allows importers to file entries with the best available information and correct it later -- essential for transfer pricing adjustments, retroactive FTA qualification, and first-sale valuation claims. Without reconciliation, importers would need to amend every individual entry, which is operationally prohibitive at scale.

---

## 5. Documents

Each document in international trade serves a specific regulatory or commercial purpose. This section describes what data each document contains and why it exists.

---

### 5.1 Commercial Invoice

- **What it is:** The seller's bill for the goods. The primary document for customs valuation.
- **Key data elements:** Seller/buyer names and addresses, invoice date, invoice number, description of goods, quantity, unit price, total price, currency, Incoterm, payment terms, country of origin, marks and numbers.
- **Who creates it:** The seller/exporter.
- **When created:** At the time of sale or shipment.
- **Why it matters for clearance:** The commercial invoice is the foundation of customs valuation. The declared value on the entry must match (or be derivable from) the commercial invoice. CBP can request the original invoice to verify value. Discrepancies between the invoice and the entry trigger valuation reviews.

### 5.2 Packing List

- **What it is:** A detailed listing of the contents of each package in the shipment.
- **Key data elements:** Package numbers, contents of each package, net and gross weight per package, dimensions per package, marks and numbers.
- **Who creates it:** The shipper.
- **When created:** At packing.
- **Why it matters for clearance:** The packing list enables CBP to identify which package contains which goods during a physical exam. It supports piece count verification and weight verification. For multi-line-item shipments, the packing list shows how line items are distributed across packages.

### 5.3 Bill of Lading / Air Waybill

- **See Section 1.1** for HAWB, MAWB, and Ocean B/L.
- **Key data elements:** Shipper, consignee, notify party, description of goods, piece count, gross weight, dimensions, routing, freight charges (prepaid or collect), declared value for carriage.
- **Why it matters for clearance:** The transport document is required for customs filing. It is the link between the commercial transaction (invoice) and the physical movement (carrier manifest). For ocean, the B/L is a document of title -- the party holding the original B/L has the right to claim the goods.

### 5.4 Certificate of Origin

#### Preferential Certificate of Origin

- **What it is:** A document certifying that the goods qualify for preferential duty treatment under a specific FTA.
- **Key data elements:** Exporter, producer, and importer identification; product description and HS code; origin criterion (tariff shift, RVC percentage, net cost); blanket period (if applicable, typically 12 months for USMCA).
- **Format:** USMCA uses a certification of origin (not a form -- can be included on an invoice or as a standalone document). Other FTAs may have specific forms.
- **Who creates it:** The exporter, producer, or importer (USMCA allows any of these parties to certify).
- **When created:** Before or at the time of importation; can cover a blanket period.
- **Mutable:** No -- a certificate covers a specific product and period.
- **Why it matters:** This is the evidentiary basis for an FTA duty preference claim. Without a valid certificate of origin, the preference claim is unsupported and vulnerable to audit. CBP can request the certificate at any time -- at filing, at random audit, or as part of a verification with the exporting country's customs authority.

#### Non-Preferential Certificate of Origin

- **What it is:** A document certifying the country of origin of the goods, NOT for duty preference purposes, but for other requirements (marking, origin-based trade restrictions, letters of credit).
- **Who creates it:** Often issued by the Chamber of Commerce in the exporting country.
- **When created:** Before shipment.
- **Why it matters:** Required by some importing countries as part of the customs declaration. Also required by banks for letter-of-credit transactions (to verify origin matches the L/C terms).

### 5.5 Importer Security Filing (ISF 10+2)

- **What it is:** Also known as "10+2," this is a US customs requirement for ocean cargo. The importer (or their agent) must file 10 data elements with CBP at least 24 hours before the cargo is loaded onto the vessel at the foreign port. The carrier provides 2 additional data elements.
- **The 10 Importer Data Elements:**
  1. **Manufacturer (or supplier) name and address** -- Who made or supplied the goods.
  2. **Seller name and address** -- Who sold the goods.
  3. **Buyer name and address** -- Who purchased the goods.
  4. **Ship-to name and address** -- Where the goods are being shipped.
  5. **Container stuffing location** -- Where the container was loaded.
  6. **Consolidator name and address** -- The party that consolidated the cargo.
  7. **Importer of Record number** -- IOR's IRS/EIN number.
  8. **Consignee number** -- Consignee's IRS/EIN number.
  9. **Country of origin** -- Where the goods were manufactured.
  10. **HS code (6-digit minimum)** -- Commodity classification.
- **The 2 Carrier Data Elements:**
  11. **Vessel stow plan** -- Where containers are located on the vessel.
  12. **Container status messages** -- Container movement events.
- **Who creates it:** The importer or their agent (broker/forwarder) files the 10 elements; the carrier files the 2 elements.
- **When created:** Must be filed at least 24 hours before vessel loading at the foreign port.
- **Why it matters:** ISF is a security-focused filing that gives CBP advance data about ocean cargo. Late or inaccurate ISF filing triggers penalties ($5,000 per violation). ISF data feeds CBP's targeting system -- cargo identified as high-risk based on ISF data may be held for exam upon arrival. ISF is SEPARATE from the customs entry -- a shipment can have an ISF filed but no entry filed yet. ISF compliance is a significant operational burden for ocean importers and their brokers.

### 5.6 AES / EEI (Export Filing)

- **What it is:** The Automated Export System (AES) Electronic Export Information (EEI) filing, formerly the Shipper's Export Declaration (SED). Required for most US exports valued over $2,500 per Schedule B number or any export requiring an export license.
- **Key data elements:** USPPI (US Principal Party in Interest), ultimate consignee (foreign buyer), intermediate consignee, forwarding agent, country of ultimate destination, Schedule B number (US export classification code), quantity, value, export control classification number (ECCN), license information.
- **Who creates it:** The USPPI (exporter) or their forwarding agent.
- **When created:** Before export.
- **Why it matters:** EEI is mandatory for export compliance. The Internal Transaction Number (ITN) returned by AES must be provided to the carrier before export. Failure to file is a violation with penalties up to $10,000 per shipment. EEI data is used for export control enforcement (preventing diversion of controlled technology) and trade statistics.

### 5.7 FDA Prior Notice

- **What it is:** An advance notification to the FDA for all food (including animal feed) imports into the US, required under the Bioterrorism Act of 2002 and the FDA Food Safety Modernization Act (FSMA).
- **Key data elements:** FDA product code, common or usual name, estimated quantity, manufacturer, grower (if produce), shipper, country of origin, anticipated arrival information.
- **Filing deadline:** 2-24 hours before arrival (varies by transport mode: 2 hours for truck, 4 hours for rail, 8 hours for air, 15 days/24 hours for ocean depending on timing).
- **Who creates it:** The importer, customs broker, or foreign supplier's agent.
- **When created:** Before shipment arrival.
- **Why it matters:** Food imports without proper prior notice are REFUSED admission -- the goods cannot clear customs regardless of whether all other requirements are met. Prior notice feeds FDA's PREDICT system (Predictive Risk-based Evaluation for Dynamic Import Compliance Targeting), which uses AI/ML to score each food import line for risk.

### 5.8 Arrival Notice

- **What it is:** A notification from the carrier or freight forwarder to the consignee/notify party that goods have arrived (or will arrive) at the destination port.
- **Key data elements:** B/L or AWB number, vessel/flight information, arrival date, cargo description, free time period, demurrage/detention charges schedule, customs status.
- **Who creates it:** The carrier or their agent.
- **When created:** Upon or before vessel/flight arrival.
- **Why it matters:** The arrival notice triggers the consignee's obligation to arrange customs clearance and cargo pickup. Failure to respond results in demurrage charges (for use of the container) and storage charges (for use of terminal/warehouse space). After free time expires, charges accumulate rapidly.

### 5.9 Delivery Order

- **What it is:** An authorization from the carrier or terminal to release the cargo to the consignee or their agent.
- **Key data elements:** B/L reference, consignee authorization, container/cargo identification, release conditions (e.g., all charges paid, customs released).
- **Who creates it:** The carrier or terminal operator.
- **When created:** After freight charges are paid and customs clearance is obtained.
- **Why it matters:** The delivery order is the "key" that releases the physical goods. Without it, the terminal will not release the container/cargo. It represents the convergence of commercial clearance (freight paid) and regulatory clearance (customs released).

### 5.10 Proof of Delivery (POD)

- **What it is:** Confirmation that the goods were delivered to the consignee at the final destination, including the date/time, recipient name, and signature.
- **Key data elements:** Delivery date/time, recipient name, signature (electronic or physical), delivery address, any exceptions noted (damage, shortage).
- **Who creates it:** The delivering carrier (driver or delivery agent).
- **When created:** At the moment of delivery.
- **Mutable:** No.
- **Why it matters:** POD is the final event in the shipment lifecycle. It provides legal proof of delivery for insurance claims, payment triggers (some contracts require POD before payment), and dispute resolution. For DDP shipments, POD confirms that the seller's obligation has been fulfilled.

---

## 6. Events / Milestones

Shipment events are timestamped records of each significant occurrence in the shipment lifecycle. They provide visibility, trigger downstream actions, and create an audit trail.

---

### 6.1 Standard Tracking Events

| Event Code | Event Name | Who Generates | What Data It Carries | Operational Significance |
|-----------|------------|---------------|---------------------|-------------------------|
| PU | Pickup | Origin driver/station | Timestamp, location, piece count, weight | Shipment has entered the carrier network. Clock starts for transit time. |
| AR | Arrived at Facility | Hub/station | Timestamp, facility code, scan data | Package is at a carrier processing facility. Used for network tracking. |
| DP | Departed Facility | Hub/station | Timestamp, facility code, next destination | Package has left a facility. In-network transit. |
| CC | Customs Clearance | Clearance operations | Timestamp, port of entry, clearance status | CRITICAL EVENT: Indicates the shipment has entered customs processing. Everything described in Sections 4-5 happens between this event and the release event. |
| CR | Customs Released | CBP/clearance ops | Timestamp, release type (pre-clearance, post-arrival, conditional) | THE MOST IMPORTANT EVENT for clearance operations. Goods are authorized for delivery. |
| OH | On Hold | CBP/PGA | Timestamp, hold type, reason code, agency | Indicates a regulatory hold. Triggers exception management workflow. See Section 6.2. |
| OD | Out for Delivery | Delivery driver | Timestamp, vehicle, route | Package is on the delivery vehicle. Final mile. |
| DL | Delivered | Delivery driver | Timestamp, recipient name, signature, location | Delivery complete. POD generated. Shipment lifecycle ends (for tracking purposes). |
| IT | In Transit | Network systems | Timestamp, location (hub, country) | Generic in-transit event between major milestones. |
| AF | Arrived at Foreign Hub | International hub | Timestamp, hub code, country | Package has arrived at the destination country's primary gateway facility. |

---

### 6.2 Customs-Specific Events

| Event Code | Event Name | Who Generates | What Data It Carries | Operational Significance |
|-----------|------------|---------------|---------------------|-------------------------|
| EF | Entry Filed | Broker/carrier filing system | Timestamp, entry number, entry type, port code | The customs entry has been electronically transmitted to CBP/ACE. The clock starts for CBP processing. |
| UR | Under Review | CBP/ACE | Timestamp, review type (automated, manual) | CBP is actively reviewing the entry. May be automated selectivity or manual specialist review. |
| RFI | Request for Information | CBP | Timestamp, document type requested, deadline, CBP contact | CBP needs additional information before making a release decision. Triggers communication to importer/broker. Common requests: commercial invoice, certificate of origin, product photos, lab reports. |
| HLD | Hold - CBP | CBP | Timestamp, hold reason code, hold type (document, exam, enforcement), releasing officer | Goods are held pending resolution of a customs issue. Triggers cage operations (physical movement of goods to secure bonded area). |
| HLP | Hold - PGA | PGA (FDA, USDA, CPSC, EPA, etc.) | Timestamp, agency code, hold reason, PGA reference number | A Partner Government Agency has placed a hold. Independent of CBP hold status -- goods can be CBP-released but PGA-held, or vice versa. |
| EXM | Exam Ordered | CBP | Timestamp, exam type (document/tailgate/intensive/VACIS), exam site, exam deadline | CBP has ordered a physical or document examination. Triggers staging of shipment for exam. |
| EXC | Exam Complete | CBP | Timestamp, exam findings (clear, discrepancy, seizure), examiner ID | The exam is done. Findings determine next action (release, continued hold, seizure). |
| REL | Released | CBP | Timestamp, release type, conditions (if any) | GOODS ARE RELEASED. This is the "green light" for delivery. If conditional, there may be post-release obligations. |
| DTN | Detained (UFLPA) | CBP/FLETF | Timestamp, detention basis, UFLPA entity reference, evidence deadline | Goods are detained under UFLPA. Importer has 30 days to provide "clear and convincing evidence" to overcome the forced labor presumption. This is the most operationally disruptive customs event -- goods can be detained for months. |
| SEZ | Seized | CBP | Timestamp, seizure basis (fraud, contraband, UFLPA, IPR), case number | Goods are seized by CBP. Importer loses possession. May be able to petition for relief but the goods are now in CBP's custody. |
| LIQ | Liquidated | CBP | Timestamp, liquidation date, final duty amount, adjustment amount | Entry has been liquidated (finalized). If the final amount differs from the deposit, a refund or additional bill is generated. Liquidation typically occurs 10-11 months after entry date. |
| RCN | Reconciled | CBP | Timestamp, reconciliation entry number, adjusted values | A reconciliation entry has been processed, adjusting prior entry data (value, classification, FTA). |
| PTF | Protest Filed | Importer/broker | Timestamp, protest number, contested issue (classification, value, rate) | Importer has formally protested CBP's liquidation decision. Must be filed within 180 days of liquidation. |

---

### 6.3 Exception Events

| Event Code | Event Name | Who Generates | What Data It Carries | Operational Significance |
|-----------|------------|---------------|---------------------|-------------------------|
| DLY | Delay | Various | Timestamp, delay reason (weather, customs, mechanical, capacity), estimated new delivery date | A delay has occurred. EDD must be updated. Customer notification triggered. |
| DMG | Damage | Carrier/handler | Timestamp, damage description, damage location, claim number | Package has been damaged. May affect clearance if goods are inspected during exam. |
| REF | Refused | Consignee | Timestamp, refusal reason (unexpected charges, wrong goods, no order), return instructions | Consignee has refused delivery. Common for DDU shipments where the consignee is surprised by duty charges. Triggers return-to-origin or disposal process. |
| ADC | Address Correction | Carrier/driver | Timestamp, original address, corrected address | Delivery address was incorrect and has been corrected. May delay delivery. |
| MIS | Misrouted | Network systems | Timestamp, current location, intended destination | Package was sent to the wrong facility/destination. Requires rerouting. |
| MSC | Missing / Short | Warehouse/hub | Timestamp, expected piece count, actual count, investigation reference | Expected pieces are missing. Triggers inventory investigation. For customs, a piece count discrepancy between the manifest and physical count triggers a hold. |
| ABN | Abandoned | Importer/system | Timestamp, abandonment type (voluntary, involuntary after 15 days without entry) | Importer has abandoned the goods or failed to file entry within 15 days of arrival. Goods go to general order warehouse. After 6 months in general order, goods are sold or destroyed. |
| GO | General Order | CBP/carrier | Timestamp, GO warehouse, GO date, destruction/sale date | Goods have been transferred to general order. Carrier custody ends. GO warehouse assumes custody. |

---

### 6.4 Event Data Structure

Each event carries a standard set of metadata:

- **Event ID:** Unique identifier for the event.
- **Shipment ID / Tracking Number:** Links the event to the shipment.
- **Entry Number:** Links the event to the customs entry (when applicable).
- **Timestamp:** Date and time of the event (UTC preferred for consistency across time zones).
- **Location:** Where the event occurred (facility code, city, country).
- **Actor:** Who or what generated the event (system ID, officer ID, driver ID).
- **Event Type Code:** Standardized code (see tables above).
- **Event Description:** Human-readable description.
- **Supporting Data:** Event-type-specific data fields (hold reason, exam type, duty amount, etc.) as a structured payload.

---

## 7. Financial Records

Customs clearance generates a complex set of financial obligations and transactions. Accurate financial tracking is essential for cost management, compliance, and duty recovery.

---

### 7.1 Duty and Tax Line Items

Each duty or tax line item on an entry contains:

#### Duty Type

- **Values:** MFN Duty, Section 301, Section 232, IEEPA Reciprocal, Antidumping Duty, Countervailing Duty, FTA Preferential Rate, Marking Duty, etc.
- **Why it matters:** Each duty type has a different legal basis, rate, and potential for challenge. Duty types stack -- a single line item can have MFN + Section 301 + IEEPA simultaneously.

#### Duty Rate

- **Types:**
  - **Ad valorem:** A percentage of the customs value (e.g., 25%).
  - **Specific:** A fixed amount per unit of quantity (e.g., "$0.42 per kg").
  - **Compound:** A combination of ad valorem and specific (e.g., "5% plus $0.10 per kg").
  - **Mixed/alternative:** The higher or lower of two rates (e.g., "8% or $0.60 per kg, whichever is higher").
- **Why it matters:** The rate type determines the calculation methodology. Ad valorem rates require accurate value; specific rates require accurate quantity in the correct unit of measure; compound and mixed rates require both.

#### Duty Amount

- **What it is:** The calculated duty for this duty type on this line item.
- **Formula:** For ad valorem: customs value x rate. For specific: quantity x rate per unit.
- **Who creates it:** Calculated by the broker's system.
- **When created:** At entry filing (estimated) and at liquidation (final).
- **Mutable:** Estimated at filing; finalized at liquidation.

#### Preference Claimed

- **What it is:** Whether a preferential rate was claimed and the program code.
- **Why it matters:** If a preference is claimed, the duty may be zero or reduced. The preference claim must be supported by documentation. If the preference is denied at audit, the importer owes the full MFN rate plus interest and potentially penalties.

---

### 7.2 Broker Fees

#### Brokerage Fee

- **What it is:** The customs broker's fee for preparing and filing the customs entry.
- **Typical range:** $25-$150 per entry for express/e-commerce; $150-$500+ for complex formal entries; hourly rates for consulting/advisory work.
- **Who creates it:** The broker, per their fee schedule or contract.
- **When created:** At entry filing or at the end of a billing period.
- **Why it matters:** Brokerage fees are a significant component of import cost, especially for low-value shipments where the fee can exceed the duty amount.

#### Additional Service Fees

- **What it is:** Fees for services beyond basic entry filing: classification research, protest preparation, FTA analysis, exam coordination, storage management, prior notice filing, etc.
- **Why it matters:** These fees can add significantly to the clearance cost, especially for entries with exceptions (holds, exams, RFIs).

---

### 7.3 Freight Charges

#### Origin Charges

- **What it is:** Costs incurred at the origin: pickup, local transport to the airport/port, export customs clearance, origin terminal handling.
- **Who creates it:** The carrier or origin agent.
- **When created:** After shipment.
- **Why it matters:** For certain Incoterms (EXW, FCA), origin charges may not be included in the customs value. For CIF/CIP, they are.

#### International Freight

- **What it is:** The cost of transporting goods from the port of export to the port of import.
- **Who creates it:** The carrier.
- **When created:** At booking (quoted) and after delivery (finalized).
- **Why it matters:** For US customs valuation (FOB basis), international freight is NOT included in the customs value. For EU customs valuation (CIF basis), it IS included. This distinction directly affects the duty base.

#### Destination Charges

- **What it is:** Costs incurred at the destination: terminal handling, local delivery, customs processing fees, container drayage.
- **Who creates it:** The carrier or destination agent.
- **When created:** After delivery.
- **Why it matters:** Destination charges are generally NOT included in the customs value in any jurisdiction (they occur after importation).

#### Accessorial Charges

- **What it is:** Additional charges for non-standard services: residential delivery, liftgate, inside delivery, limited access, appointment delivery, re-delivery attempts.
- **Who creates it:** The carrier.
- **When created:** After the service is performed.
- **Why it matters:** Not customs-relevant but part of the total landed cost calculation.

---

### 7.4 Insurance

- **What it is:** Premium paid for cargo insurance covering loss or damage during transit.
- **Who creates it:** The shipper or their insurance broker.
- **When created:** At booking or policy inception.
- **Why it matters:** Under CIF/CIP Incoterms, insurance cost is part of the customs value in CIF-valuation jurisdictions (EU). Insurance also determines financial recovery in case of loss -- if goods are seized by customs, cargo insurance typically does NOT cover the loss (seizure is an excluded peril).

---

### 7.5 Storage and Demurrage

#### Demurrage

- **What it is:** Charges imposed by the ocean carrier for retaining the carrier's container beyond the allotted "free time" (typically 3-7 days after vessel discharge).
- **Who creates it:** The ocean carrier.
- **When created:** Accrues daily after free time expires.
- **Why it matters:** Customs holds directly cause demurrage. A container held for exam or UFLPA detention continues accumulating demurrage charges. These charges can reach $200-500+ per container per day and escalate over time. For lengthy holds, demurrage can exceed the value of the goods.

#### Detention

- **What it is:** Charges for retaining the carrier's container equipment (the chassis and container itself) outside the terminal beyond free time.
- **Who creates it:** The ocean carrier or chassis provider.
- **When created:** Accrues daily.
- **Why it matters:** Detention charges combine with demurrage to create significant financial pressure to resolve customs holds quickly.

#### Storage

- **What it is:** Charges imposed by the terminal, warehouse, or carrier facility for storing goods.
- **Who creates it:** The terminal operator, bonded warehouse, or carrier.
- **When created:** After free time expires.
- **Why it matters:** Cargo sitting in cage (bonded area) at a carrier facility or in a general order warehouse accumulates storage charges. For UFLPA detentions lasting months, storage charges can be substantial.

---

### 7.6 Payment Terms and Methods

#### Duty Payment Method

- **Values:**
  - **Periodic Monthly Statement (PMS):** Importers with continuous bonds can pay duties on a monthly consolidated statement rather than per-entry. Most large importers use PMS.
  - **ACH Debit:** Electronic funds transfer from the importer's bank account.
  - **Check / Cashier's Check:** For individual entries.
  - **Customs Bond Draw:** Duties charged against the bond (if bond covers duty).
- **Who creates it:** The importer/broker selects the payment method.
- **When created:** At account setup.
- **Why it matters:** Payment method affects cash flow. PMS gives importers approximately 10 extra days to pay duties (statements are due on the 15th working day of the month following the statement). For importers with high duty volumes (millions per month), this cash flow benefit is significant.

---

### 7.7 Refund and Drawback

#### Duty Drawback

- **What it is:** A refund of up to 99% of duties paid on imported goods that are subsequently exported, either in their imported form (unused merchandise drawback) or as part of a manufactured product (manufacturing drawback).
- **Key data elements:** Import entry number, export entry/EEI reference, product linkage (how the imported material relates to the exported product), duty paid, drawback claimed.
- **Who creates it:** The importer or drawback specialist, filed as a drawback claim with CBP.
- **When created:** After export of the goods.
- **Processing time:** Typically 1-3 years for CBP to process a drawback claim.
- **Why it matters:** Drawback is one of the most under-utilized customs programs. Companies that import materials and export finished products (or re-export goods) are often entitled to duty refunds they never claim. With duty rates now exceeding 100% on some products, the drawback opportunity can be enormous. The administrative burden of linking imports to exports and documenting the relationship deters many companies from pursuing drawback.

#### Protest Refund

- **What it is:** A refund resulting from a successful protest of CBP's liquidation decision.
- **Key data elements:** Entry number, protest number, contested issue, CBP decision, refund amount.
- **Who creates it:** The importer/broker files the protest; CBP processes the refund if the protest is approved.
- **When created:** After CBP approves the protest.
- **Why it matters:** Protests are the mechanism for challenging incorrect duty assessments after liquidation. The 180-day filing window from the date of liquidation is absolute -- missing the deadline forfeits the right to protest forever.

---

## 8. Reference Data Entities

These are not per-shipment records but the master data tables that drive classification, duty calculation, compliance screening, and regulatory processing.

---

### 8.1 Tariff Schedule (HTSUS / Combined Nomenclature)

- **Fields:** HS code, description, MFN duty rate, special rates (FTA preferences), unit of quantity, chapter notes, section notes, General Rules of Interpretation.
- **Over 17,000 tariff lines at the 10-digit level in the US.
- **Updated:** Annually with interim amendments. Tariff program changes (301, 232, IEEPA) can modify effective rates within days.
- **Why it matters:** This is the master table that maps every product to its duty rate. Accuracy and currency of the tariff schedule is the foundation of duty calculation.

### 8.2 Restricted Party Lists

- **Lists:** OFAC SDN (Specially Designated Nationals), BIS Entity List, BIS Denied Persons List, UFLPA Entity List, EU Consolidated Sanctions List, UN Security Council Consolidated List.
- **Fields:** Entity name, aliases, addresses, identification numbers, list source, programs, effective date.
- **Updated:** Continuously (OFAC updates can occur any business day).
- **Why it matters:** Every party on a shipment (shipper, consignee, manufacturer, notify party, end user) must be screened against these lists before the shipment can proceed. A match triggers a hold and requires license or exemption review. Failure to screen is a strict-liability violation with severe penalties (civil fines up to $307,922 per violation for OFAC; criminal penalties including imprisonment for willful violations).

### 8.3 Exchange Rate Tables

- **Fields:** Currency code, rate to USD, effective date, source (CBP/Treasury/Federal Reserve).
- **Updated:** CBP publishes certified exchange rates quarterly for most currencies.
- **Why it matters:** All US customs entries must be valued in USD. If the transaction is in a foreign currency, the conversion MUST use the CBP-certified rate in effect on the date of export. Using market rates, bank rates, or other rates is non-compliant.

### 8.4 Fee Schedules

- **Fields:** Fee type (MPF, HMF), rate, minimum, maximum, fiscal year, entry type applicability.
- **Updated:** Annually (CBP publishes updated fee amounts in the Federal Register).
- **Why it matters:** Fee schedules define the per-entry processing fees that apply to every formal entry. The MPF minimum and maximum are adjusted periodically for inflation.

---

## Cross-Cutting Observations

### Data Quality is the Universal Determinant

Across every entity and attribute described in this document, data quality is the dominant factor in whether a shipment clears smoothly or creates an exception. The most impactful data quality issues are:

1. **Vague or missing commodity descriptions** -- causes classification errors, PGA misscreening, and CBP holds.
2. **Incorrect or missing HS codes** -- cascades into wrong duty rates, wrong PGA requirements, wrong FTA eligibility, wrong AD/CVD assessment.
3. **Wrong country of origin** -- triggers incorrect tariff programs (or misses applicable programs), misses marking requirements, potentially constitutes fraud.
4. **Understated value** -- causes duty underpayment, triggers valuation reviews, creates penalty exposure.
5. **Missing party identification** -- prevents entry filing (no IRS number = no entry), blocks denied party screening, creates IOR ambiguity.

### Regulatory Velocity Affects Every Attribute

Many of the attributes described here have rates, thresholds, or applicability rules that change frequently. IEEPA rates have been modified multiple times in months. Section 301 lists expand and contract. De minimis thresholds are being eliminated globally. Exchange rates fluctuate. PGA requirements evolve. Any system that treats these attributes as static reference data will produce incorrect results within months of deployment.

### Immutability vs. Mutability Matters for System Design

This document carefully distinguishes between immutable attributes (tracking number, entry number, entry date) and mutable attributes (EDD, estimated duties, clearance status). Immutable attributes are safe to cache, index, and use as keys. Mutable attributes require versioning, event-driven updates, and careful handling of race conditions (e.g., a duty estimate changing while a payment is being processed).

---

*This document is part of the FedEx Global Clearance Knowledge Base. For related context, see:*
- *[01: Regulatory Landscape](01_regulatory_landscape.md)*
- *[02: Pain Points Exhaustive](02_pain_points_exhaustive.md)*
- *[04: Clearance Process Today](04_clearance_process_today.md)*
- *[06: Supply Chain Traceability from Clearance Data](06_supply_chain_traceability_from_clearance_data.md)*
