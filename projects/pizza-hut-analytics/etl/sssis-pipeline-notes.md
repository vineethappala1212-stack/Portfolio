# SSIS / ETL Workflow – Pizza Hut Sales Analytics

**Objective:**  
Automate nightly extraction, transformation, and loading of transactional data from POS systems into Amazon Redshift.

## Workflow Summary
1. **Source Extraction**
   - POS data, coupon tables, and menu items pulled via ODBC connectors.
   - Incremental load logic based on `OrderDate`.

2. **Transformation**
   - Data cleansing: TRIM, NULL handling, UPPER() for promo codes.
   - Join with master tables for Store and Product.
   - Derive calculated fields: Net Sales, Tax, Delivery Fee, Discount %.

3. **Loading**
   - Use `OLE DB Destination` to stage in Redshift (`stg_orders`, `stg_customers`).
   - Control flow includes Pre-SQL cleanup (`DELETE FROM ... WHERE load_date = GETDATE()`).
   - Final load into **FactSales** and **DimStore**, **DimCustomer**, **DimProduct**.

4. **Automation & Validation**
   - Scheduled via SQL Agent nightly (2 AM CST).
   - Row-count validation script logs record counts and discrepancies.
   - Error handling via SSIS Event Handlers → sends email alert if any package fails.

5. **Performance Optimization**
   - Parallel data flow tasks for large tables.
   - Lookup cache mode = Full for Store/Product dimensions.
   - Implemented index maintenance every weekend on Redshift fact tables.

---

**Output:**  
Clean star-schema dataset ready for Tableau dashboards and repeat-rate SQL analytics.
