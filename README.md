# 📈-ABC-Analysis-in-Power-BI-Power-Query-DAX-
This repository demonstrates an **end‑to‑end**, **real‑world ABC (Pareto) analysis** using **Power BI**, starting from **messy CSV data**, cleaning it with **Power Query (M)**, and building an **advanced DAX-driven analytical model** with dynamic thresholds, field parameters, and executive‑grade visuals.
<br>
The goal of this project is not just to build visuals, but to **teach the reasoning behind every step**, so readers can **learn**, **reuse**, and **extend** the patterns.
## 🧠 Business Problem
Delivery delays are hurting operations. We need to know which Products, Suppliers, or Regions contribute most to the delay so we can prioritize action.
Instead of looking at raw totals, we apply **ABC classification (Pareto principle)**:
  - **A** → Small number of items causing most of the delay
  - **B** → Moderate contributors
  - **C** → Long tail with minimal impact
## ♻ Power Query (M) – From Messy to Clean Data
- Real‑world data is rarely clean. This section explains each transformation step applied in Power Query.
- 🔹 Step 1: Load CSV
  ```m
  = Csv.Document(
    File.Contents("C:\Users\user\Desktop\Messy Data.csv"), 
    [Delimiter = ",", Encoding = 1252, QuoteStyle = QuoteStyle.None]
  )
  ```
  - Why?
    - Explicit encoding avoids character corruption
    - Raw import without assumptions
- 🔹 Step 2: Promote Headers
  ```m
  = Table.PromoteHeaders(Source, [PromoteAllScalars = true])
  ```
  - Why?
    - CSV files often treat headers as first data row
    - Required for semantic modeling in Power BI
- 🔹 Step 3: Make a Column List dynamically
  ```m
  let
    Source = Csv.Document(File.Contents("C:\Users\user\Desktop\Messy Data.csv"),[Delimiter=",", Encoding=1252, QuoteStyle=QuoteStyle.None]),
    #"Promoted Headers" = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    Custom1 = Table.ColumnNames(#"Promoted Headers")
  in
    Custom1
  ```
  - Why?
    - For transforming a single function to all columns
- 🔹 Step 4: Trim All Text Columns
  ```m
  = Table.TransformColumns(
    #"Promoted Headers", 
    List.Transform(ColList, each {_, Text.Trim, type any})
  )
  ```
  - Why this matters:
    - Removes hidden spaces causing duplicate keys
    - Prevents relationship & filter issues later
- 🔹 Step 5: Standardize Business Columns
  ```m
  = Table.TransformColumns(
    #"Trim All Col", 
    {
      {"Supplier Name", each Text.Upper(Text.Start(_, 3)) & " " & Text.End(_, 1), type any}, 
      {"Product Code", each Text.Upper(Text.Start(_, 3)) & " " & Text.End(_, 3), type any}, 
      {
        "Delivery Delay (days)", 
        each 
          if _ = null then null
          else if Value.Is(_, type number) then _
          else if Text.Upper(_) = "ZERO" then 0
          else Text.BeforeDelimiter(Text.Trim(_), " "), 
        type any
      }
    }
  )
  ```
  - What this solves:
    - Inconsistent naming conventions
    - Mixed numeric & text delay values ("12 days", "ZERO")
  - This mimics **real operational data issues**.
- 🔹 Step 6: Enforce Data Types
  ```m
  = Table.TransformColumnTypes(
    Standardize, 
    {
      {"Order ID", Int64.Type}, 
      {"Supplier Name", type text}, 
      {"Product Code", type text}, 
      {"Delivery Delay (days)", Int64.Type}, 
      {"Order Date", type date}, 
      {"Region", type text}, 
      {"Remarks", type text}
    }
  )
  ```
  - Why?
    - Required for accurate aggregations
    - Prevents silent DAX errors
- 🔹 Step 7: Filter Invalid Records
  ```m
  // Filter invalid delays (negative, zero, text)
  = Table.SelectRows(
    #"Changed Type", 
    each ([#"Delivery Delay (days)"] <> null and [#"Delivery Delay (days)"] > 0)
  )
  ```
  - Business logic:
    - Zero or negative delays have no operational meaning
