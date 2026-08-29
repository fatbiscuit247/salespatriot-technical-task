# SalesPatriot Technical Task — Firm Data Ingestion

Ingests three supplier data files and produces three lists
of firms (orphan manufacturers, orphan vendors, and linked vendor–manufacturer
groups) ready to load into a `firms` table.

## Files
- `firm_ingestion.ipynb` — the full pipeline
- `outputs/linked_vendor_manufacturers.csv`
- `outputs/orphan_manufacturers.csv`
- `outputs/orphan_vendors.csv`

## How to run
Place the three provided source files in the project root:
`cages_ANONYMIZED.csv`, `Cage Maintenance_ANONYMIZED.csv`,
`Sales Patriot Project Vendor File_ANONYMIZED.csv`. Run the notebook top to
bottom. The three output CSVs are written to the working directory. 

or 

The notebook can also be opened in Google Colab


## Data model
- **Cages**:  manufacturer identity directory (`id` = cage code, `company` = name)
- **Maintenance** — relationships (`CageCode` → manufacturer, `VendorCode` → parent vendor)
- **Vendors**:  vendor identity directory (`VendorId` = id, `VendorName` = name, plus contact fields)

Manufacturer names come from `cages` (join `CageCode → id`); vendor names come
from `vendors` (match `VendorCode → VendorId`). 

## While creating each list, here are some of my notes/key decisions I found/made along the way: 
- null values appear throughout: through Vendors, Cages, and Manufacturers, and it's important to recognize the different kind of meanings this could have: orphan unresolved key, or placeholder, and how to handle each case. 
- **Cage key is `id`, not `cao`.** `cao` had only 17 distinct values across
  ~4M rows (a category field); `id` is unique per firm and matches `CageCode`
  in ~98% of cases, confirming it as the true cage code.
- **Vendor link required normalization.** `VendorCode` arrived as floats
  (`2538.0`) and unpadded, while `VendorId` is 6-digit zero-padded. Stripping
  the trailing `.0` and zero-padding to 6 digits raised the match rate from
  ~39% to ~99.9%.
- **Nameless vendors treated as non-links.** 26 vendor records have a null
  `VendorName`. The 3,491 manufacturers pointing at these placeholder vendors
  are reclassified as orphans, since a link to a nameless vendor is not a real
  relationship.
- **Linked wins over orphan.** 5 manufacturers appeared both linked (via one
  maintenance row) and orphaned (via a separate null row). These are kept in
  the linked list and dropped from orphans.
- **linkID = the vendor's id.** Each linked group shares its parent vendor's
  id as `linkID`, tying parent and children in the flat output. Orphan lists
  have no `linkID`. For linked manufacturers I derive it from VendorCode, which points to its corresponding vendor. 

## Data-quality notes
- ~388 manufacturers reference a cage not present in the cages directory;
  they remain in the output with a null name.
- The cages file has no contact columns, so manufacturers carry null
  email/phone/contact; vendors carry full contact info.

## Validation
Every firm is classified exactly once:
- Manufacturers: 8,492 linked + 11,889 orphan = 20,381 (= 20,386 maintenance
  rows − 5 deduped duplicates)
- Vendors: 5,091 parents + 13,227 orphan = 18,318 (= total vendors)