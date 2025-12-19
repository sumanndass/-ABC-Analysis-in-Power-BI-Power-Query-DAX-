# 📈 ABC Analysis in Power BI (Power Query + DAX)
This repository demonstrates an **end‑to‑end**, **real‑world ABC (Pareto) analysis** using **Power BI**, starting from **messy CSV data**, cleaning it with **Power Query (M)**, and building an **advanced DAX-driven analytical model** with dynamic thresholds, field parameters, and executive‑grade visuals.
<br>
The goal of this project is not just to build visuals, but to **teach the reasoning behind every step**, so readers can **learn**, **reuse**, and **extend** the patterns.
<br>
<img width="893" height="503" alt="image" src="https://github.com/user-attachments/assets/75b8197a-8a65-47cd-be48-902e3d67abf3" />
## 🧭 Project Overview
- This project delivers a **complete**, **production-style ABC (Pareto) analysis solution** built in **Power BI**, designed to transform **messy operational data** into **clear**, **decision-ready insights**.
- The solution walks through the **entire analytics lifecycle**:
  - **Ingest** inconsistent CSV data
  - **Clean & standardize** it using Power Query (M)
  - **Model** it with a scalable, field-parameter–driven design
  - **Analyze** it using modern DAX (WINDOW, cumulative %, dynamic thresholds)
  - **Communicate** insights with executive-grade visuals and narratives
## 🎯 Objective
- Enable stakeholders to quickly answer:
  - *Which Products, Suppliers, or Regions drive the majority of delivery delays, and where should improvement efforts be focused first?*
## 🔍 What Makes This Project Different
- **Real-world data problems** (mixed types, inconsistent text, invalid values)
- **User-controlled ABC thresholds** (A% / B%) via disconnected tables
- **Single model, multiple dimensions** using Field Parameters
- **Modern DAX patterns** (WINDOW, dynamic ranking, cumulative logic)
- **Insight-first storytelling**, not just charts
## 🧠 Key Outcomes
- Identifies **A-category drivers** responsible for the majority of delay
- Quantifies **concentration of impact** (e.g., “Top 4 suppliers cause 72% of delay”)
- Supports **prioritized, high-ROI operational actions**
## 🧠 Business Problem
Delivery delays are hurting operations. We need to know which Products, Suppliers, or Regions contribute most to the delay so we can prioritize action.
Instead of looking at raw totals, we apply **ABC classification (Pareto principle)**:
  - **A** → Small number of items causing most of the delay
  - **B** → Moderate contributors
  - **C** → Long tail with minimal impact
## ♻ Power Query (M) – From Messy to Clean Data
- Real‑world data is rarely clean. This section explains each transformation step applied in Power Query.
- [Messy Data](https://github.com/sumanndass/ABC-Analysis-in-Power-BI-Power-Query-DAX-/blob/main/Messy%20Data.csv)
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
- [Cleaned Data](https://github.com/sumanndass/ABC-Analysis-in-Power-BI-Power-Query-DAX-/blob/main/Cleaned%20Data.xlsx)
## 📊 Power BI - DAX & Visualization
### 📖 Data Model Design
- **Fact Table**: FactData
- **Disconnected Tables**:
  - ABC Classification (A / B / C)
  - Threshold sliders (A %, B %)
  - Field Parameter table (Col)
- Disconnected tables allow **dynamic logic without breaking filter context**.
### 💥 Core DAX Measures (Step by Step)
- 🔹 Total Delivery Delay
  ```dax
  _Total Delivery Delay = SUM ( FactData[Delivery Delay (days)] )
  ```
  - Foundation measure used everywhere.
- 🔹 Dynamic Rank (Field Parameter Aware)
  ```dax
  _Rank = 
  VAR dimord = SELECTEDVALUE('Col'[Col Order])
  RETURN
      SWITCH(
          TRUE(),
          dimord = 0 && ISINSCOPE(FactData[Product Code]),
          RANKX(ALLSELECTED(FactData[Product Code]), [_Total Delivery Delay],, DESC, Dense),
          dimord = 1 &&  ISINSCOPE(FactData[Supplier Name]),
          RANKX(ALLSELECTED(FactData[Supplier Name]), [_Total Delivery Delay],, DESC, Dense),
          dimord = 2 && ISINSCOPE(FactData[Region]),
          RANKX(ALLSELECTED(FactData[Region]), [_Total Delivery Delay],, DESC, Dense)
      )
  ```
  - Why this pattern:
    - Works for Product / Supplier / Region
    - No hard‑coded columns
- 🔹 Cumulative Delay
  ```dax
  _Cum Delay = 
  VAR dimord =  SELECTEDVALUE(Col[Col Order])
  RETURN
      SWITCH(
          TRUE(),
          dimord = 0,
          CALCULATE(
              [_Total Delivery Delay],
              WINDOW(
                  1, ABS,
                  0, REL,
                  ADDCOLUMNS(
                      ALLSELECTED(FactData[Product Code]),
                      "_delay",
                      [_Total Delivery Delay]
                  ),
                  ORDERBY([_delay], DESC)
              )
          ),
          dimord = 1,
          CALCULATE(
              [_Total Delivery Delay],
              WINDOW(
                  1, ABS,
                  0, REL,
                  ADDCOLUMNS(
                      ALLSELECTED(FactData[Supplier Name]),
                      "_delay",
                      [_Total Delivery Delay]
                  ),
                  ORDERBY([_delay], DESC)
              )
          ),
          dimord = 2,
          CALCULATE(
              [_Total Delivery Delay],
              WINDOW(
                  1, ABS,
                  0, REL,
                  ADDCOLUMNS(
                      ALLSELECTED(FactData[Region]),
                      "_delay",
                      [_Total Delivery Delay]
                  ),
                  ORDERBY([_delay], DESC)
              )
          )
      )
  ```
  - Why WINDOW():
    - Modern, efficient cumulative logic
    - Correct ordering guaranteed
- 🔹 Cumulative Percentage
  ```dax
  _Cum Delay % = 
  VAR dimord =  SELECTEDVALUE(Col[Col Order])
  RETURN
      SWITCH(
          TRUE(),
          dimord = 0,
          DIVIDE([_Cum Delay], CALCULATE([_Total Delivery Delay], ALL(FactData[Product Code]))),
          dimord = 1,
          DIVIDE([_Cum Delay], CALCULATE([_Total Delivery Delay], ALL(FactData[Supplier Name]))),
          dimord = 2,
          DIVIDE([_Cum Delay], CALCULATE([_Total Delivery Delay], ALL(FactData[Region])))
      )
  ```
  - This turns raw totals into Pareto insight.
- 🔹 ABC Classification Logic
  ```dax
  _ABC Category = 
  VAR CumPerc = [_Cum Delay %]
  VAR TotalContext =
      ISINSCOPE ( FactData[Product Code] ) || ISINSCOPE ( FactData[Supplier Name] ) || ISINSCOPE ( FactData[Region] )
  RETURN
  IF (
      NOT TotalContext,
      BLANK(),
      SWITCH (
          TRUE(),
          CumPerc <= 'A Cat %'[A Cat % Value], "A",
          CumPerc <= 'A Cat %'[A Cat % Value] + 'B Cat %'[B Cat % Value], "B",
          "C"
      )
  )
  ```
  - Key design idea:
    - Thresholds are user‑controlled
    - Classification updates instantly
- 🔹 ABC Aggregations (Count & Total Delay)
  ```dax
  _ABC Total Delay = 
  VAR ord = SELECTEDVALUE ( Col[Col Order] )
  RETURN
  IF (
      ISINSCOPE ( 'ABC Table'[Classification] ),
      SWITCH (
          TRUE(),
          ord = 0,
          SUMX (
              FILTER (
                  VALUES ( FactData[Product Code] ),
                  [_ABC Category] = SELECTEDVALUE ( 'ABC Table'[Classification] )
              ),
              [_Total Delivery Delay]
          ),
          ord = 1,
          SUMX (
              FILTER (
                  VALUES ( FactData[Supplier Name] ),
                  [_ABC Category] = SELECTEDVALUE ( 'ABC Table'[Classification] )
              ),
              [_Total Delivery Delay]
          ),
          ord = 2,
          SUMX (
              FILTER (
                  VALUES ( FactData[Region] ),
                  [_ABC Category] = SELECTEDVALUE ( 'ABC Table'[Classification] )
              ),
              [_Total Delivery Delay]
          )
      ),
      -- Grand Total (no ABC split)
      SWITCH (
          TRUE(),
          ord = 0, CALCULATE ( [_Total Delivery Delay], ALLSELECTED ( FactData[Product Code] ) ),
          ord = 1, CALCULATE ( [_Total Delivery Delay], ALLSELECTED ( FactData[Supplier Name] ) ),
          ord = 2, CALCULATE ( [_Total Delivery Delay], ALLSELECTED ( FactData[Region] ) )
      )
  )
  ```
  - Used in summary matrix.
- 🔹 Pareto Boundary Labels (Advanced Pattern)
  ```dax
  _Cum Delay Labels = 
  VAR DimSelect = SELECTEDVALUE ( 'Col'[Col Order] )
  VAR CurrentVal = [_Cum Delay %]
  VAR CurrentCat = [_ABC Category]
  
  -- 1. Get thresholds to identify B and C correctly inside the virtual table
  VAR LimitA = 'A Cat %'[A Cat % Value]
  VAR LimitB = 'A Cat %'[A Cat % Value] + 'B Cat %'[B Cat % Value]
  
  -- 2. Calculate the MIN % for the current category (Returns a Number)
  VAR MinPoint = 
      SWITCH ( DimSelect,
          0, -- Product Logic
          MINX ( 
              FILTER (
                  ADDCOLUMNS ( ALLSELECTED ( FactData[Product Code] ), "@Pct", [_Cum Delay %] ),
                  VAR RowCat = SWITCH ( TRUE(), [@Pct] <= LimitA, "A", [@Pct] <= LimitB, "B", "C" )
                  RETURN RowCat = CurrentCat
              ),
              [@Pct]
          ),
          1, -- Supplier Logic
          MINX ( 
              FILTER (
                  ADDCOLUMNS ( ALLSELECTED ( FactData[Supplier Name] ), "@Pct", [_Cum Delay %] ),
                  VAR RowCat = SWITCH ( TRUE(), [@Pct] <= LimitA, "A", [@Pct] <= LimitB, "B", "C" )
                  RETURN RowCat = CurrentCat
              ),
              [@Pct]
          ),
          2, -- Region Logic
          MINX ( 
              FILTER (
                  ADDCOLUMNS ( ALLSELECTED ( FactData[Region] ), "@Pct", [_Cum Delay %] ),
                  VAR RowCat = SWITCH ( TRUE(), [@Pct] <= LimitA, "A", [@Pct] <= LimitB, "B", "C" )
                  RETURN RowCat = CurrentCat
              ),
              [@Pct]
          )
      )
  
  -- 3. Calculate the MAX % for the current category (Returns a Number)
  VAR MaxPoint = 
      SWITCH ( DimSelect,
          0, -- Product Logic
          MAXX ( 
              FILTER (
                  ADDCOLUMNS ( ALLSELECTED ( FactData[Product Code] ), "@Pct", [_Cum Delay %] ),
                  VAR RowCat = SWITCH ( TRUE(), [@Pct] <= LimitA, "A", [@Pct] <= LimitB, "B", "C" )
                  RETURN RowCat = CurrentCat
              ),
              [@Pct]
          ),
          1, -- Supplier Logic
          MAXX ( 
              FILTER (
                  ADDCOLUMNS ( ALLSELECTED ( FactData[Supplier Name] ), "@Pct", [_Cum Delay %] ),
                  VAR RowCat = SWITCH ( TRUE(), [@Pct] <= LimitA, "A", [@Pct] <= LimitB, "B", "C" )
                  RETURN RowCat = CurrentCat
              ),
              [@Pct]
          ),
          2, -- Region Logic
          MAXX ( 
              FILTER (
                  ADDCOLUMNS ( ALLSELECTED ( FactData[Region] ), "@Pct", [_Cum Delay %] ),
                  VAR RowCat = SWITCH ( TRUE(), [@Pct] <= LimitA, "A", [@Pct] <= LimitB, "B", "C" )
                  RETURN RowCat = CurrentCat
              ),
              [@Pct]
          )
      )
  
  RETURN
      -- 4. Compare and Display
      IF ( 
          NOT ISBLANK ( CurrentCat ) && 
          ( ROUND ( CurrentVal, 4 ) = ROUND ( MinPoint, 4 ) || ROUND ( CurrentVal, 4 ) = ROUND ( MaxPoint, 4 ) ),
          CurrentVal,
          BLANK()
      )
  ```
  - Why this matters:
    - Shows Error Bar
    - Keeps Pareto curve clean
- 🔹 Dynamic Titles & Storytelling
  ```dax
  _Chart Title = 
  VAR DimSelect = SELECTEDVALUE ( 'Col'[Col Order] )
  VAR LimitA    = 'A Cat %'[A Cat % Value]
  
  VAR DimName = 
      SWITCH ( DimSelect, 0, "Products", 1, "Suppliers", 2, "Regions" )
  
  VAR ACount = 
      SWITCH ( DimSelect,
          0, COUNTROWS ( FILTER ( ALLSELECTED ( FactData[Product Code] ),  [_Cum Delay %] <= LimitA ) ),
          1, COUNTROWS ( FILTER ( ALLSELECTED ( FactData[Supplier Name] ), [_Cum Delay %] <= LimitA ) ),
          2, COUNTROWS ( FILTER ( ALLSELECTED ( FactData[Region] ),        [_Cum Delay %] <= LimitA ) )
      )
  
  VAR PctA = 
      SWITCH ( DimSelect,
          0, MAXX ( FILTER ( ALLSELECTED ( FactData[Product Code] ),  [_Cum Delay %] <= LimitA ), [_Cum Delay %] ),
          1, MAXX ( FILTER ( ALLSELECTED ( FactData[Supplier Name] ), [_Cum Delay %] <= LimitA ), [_Cum Delay %] ),
          2, MAXX ( FILTER ( ALLSELECTED ( FactData[Region] ),        [_Cum Delay %] <= LimitA ), [_Cum Delay %] )
      )
  
  RETURN
      "Top " & ACount & " " & DimName & " contribute " & FORMAT ( PctA, "0.00%" ) & " of total delivery delay"
  ```
  - Titles explain insights, not metrics.
### 📈 Final Dashboard Insight
- Top 4 suppliers contribute 72.27% of total delivery delay.
  - This leads directly to actionable decisions:
    - Focus improvement plans on A‑category suppliers
    - Avoid wasting effort on low‑impact C items
## 🏁 Key Takeaways
- Realistic messy data handling
- Field‑parameter‑native DAX
- Threshold‑driven ABC logic
- Executive‑ready storytelling
## 🎯 Decisions This Dashboard Enables
- 1️⃣ Priority-Based Intervention
  - Identifies the small subset of Products / Suppliers / Regions responsible for the majority of delivery delays
  - Enables management to focus effort where impact is highest, instead of spreading resources thin
  - Example insight:
    - Top 4 suppliers contribute over 72% of total delivery delay.
  - ➡ Action: Engage these suppliers first for process improvement, SLA reviews, or escalation.
- 2️⃣ Resource Allocation & ROI Optimization
  - Prevents low-impact work on long-tail items (C-category)
  - Helps operations teams justify why certain items are deprioritized
  - ➡ Action: Allocate improvement budgets, audits, and follow-ups primarily to A-category drivers.
- 3️⃣ Scenario & Sensitivity Analysis
  - Business users can adjust A% and B% thresholds dynamically
  - Instantly see how classifications and priorities shift
  - ➡ Action: Test different risk tolerances (e.g., aggressive vs conservative prioritization strategies).
- 4️⃣ Cross-Dimensional Insight (Without Rebuilding Reports)
  - Same model works for Products, Suppliers, and Regions
  - No duplication of visuals or measures
  - ➡ Action: Compare whether delays are driven more by supplier performance, product complexity, or regional logistics.
- 5️⃣ Clear Executive Communication
  - Dynamic titles and Pareto visuals state conclusions directly
  - Reduces interpretation time for leadership
  - ➡ Action: Executives can move straight from dashboard → decision without analyst mediation.
## 🚀 How to Extend This Project
- Add time intelligence (monthly ABC)
- Apply same logic to Sales, Defects, Revenue
- Add tooltip narration
- Convert thresholds to What‑If parameters
