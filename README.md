# Prescription Performance Dashboard (Power BI)

## 📊 Project Overview

**Goal:**  
Create a dynamic Power BI dashboard to analyze prescription performance by **doctor**, **brand**, **region**, and **medical representative**, and to understand **doctor conversion** and **brand competition** trends.

### 🎯 Objectives
- Clean & transform raw RCPA data.  
- Build a structured data model with relationships.  
- Generate insightful visuals using DAX and Power BI visuals.  
- Help business users track brand performance and doctor behavior.  

---

## 🧾 Dataset Summary

The project uses data from **three main tables**:
- **RCPA Reporting Form**  
- **Product Master**  
- **Brand Targets**

---

## ⚙️ Step 1: Data Preparation & ETL (Power Query)

### 🔹 Import Data Sources
1. Load all required files: `RCPA_Reporting_Form`, `Product_Master`, and `Brand_Targets`.  
2. In Power BI:  
   `Home → Get Data → Excel/CSV/SQL` → Select your files.  
3. Choose **Transform Data** to open **Power Query Editor**.  

### 🧹 Cleaning & Transformation Tasks

1. **Unpivot Columns**  
   - Columns containing *Med Rep* and *Doctor*  
   - Columns containing *Region*, *First, Second, and Third Chemist*  
   - Columns containing *Focus Products*  

2. **Split Columns** using delimiters  
   - Use **custom delimiters** (line feed) to separate Region and Chemist.  
   - Use **commas** and **colons** to separate Focus Product and Rx Qty/Week.  

3. **Remove Duplicates** — retain each chemist with:
   - 9 products in `Focus_RCPA_Data`  
   - 6 products in `Competitor_RCPA_Data`  

4. **Brand Targets Table:** remove first row → use *first row as header*.  
5. **Product Master Table:** remove top 3 rows → use *first row as header*.  
6. Create two new tables:  
   - `Focus_RCPA_Data`  
   - `Competitor_RCPA_Data`  
7. When done, click **Close & Apply** to commit ETL steps.

---

## 🧩 Step 2: Creating Relationships

Your Power BI model should now include:

1. `Product_Master`  
2. `Focus_RCPA_Data`  
3. `Competitor_RCPA_Data`  
4. `Brand_Targets`

### ➕ Additional Dimension Tables
- **Doctor Table** – from doctor names in both RCPA tables.  
- **Region_Dim** – from region fields.

### 🔗 Relationships (Star Schema)

| From | To | Type | Cardinality |
|------|----|------|--------------|
| Product_Master[SR NO] | Focus_RCPA_Data[Unique_ID] | 1 → * | Single |
| Product_Master[SR NO] | Competitor_RCPA_Data[Unique_ID] | 1 → * | Single |
| Product_Master[SR NO] | Brand_Targets[Product_Code] | 1 → * | Single |
| Doctor[Doctor] | Focus_RCPA_Data[Doctor] | 1 → * | Single |
| Doctor[Doctor] | Competitor_RCPA_Data[Doctor] | 1 → * | Single |
| Region_Dim[Region] | Focus_RCPA_Data[Region] | 1 → * | Single |
| Region_Dim[Region] | Competitor_RCPA_Data[Region] | 1 → * | Single |

All relationships are tested and configured for correct cross-filtering.

---

## 📈 Step 3: Building Visualizations

### 1. **Doctor Prescription Performance**

**Objective:**  
Visualize doctor prescription (Rx) performance for each brand compared to targets, broken down by **Medical Rep** and **Region**.

#### Key Questions
- How is each brand performing vs its target?  
- Which doctors or reps drive Rx volume?  
- How does performance vary by region?  

#### Tables Used
`Product_Master`, `Focus_RCPA_Data`, and `Brand_Targets`

#### DAX Measures
```DAX
Total Rx = SUM(Focus_RCPA_Data[Rx_Qty])

Total Target = SUM(Brand_Targets[Target_Qty])

Achievement % = DIVIDE([Total Rx], [Total Target], 0)
```

#### Matrix Visual Setup
| Rows | Columns | Values |
|-------|----------|---------|
| Brand (Product_Name) | Medical Rep / Region / Month | Total Rx, Total Target, Achievement % |

<img width="949" height="1117" alt="Doctor Rx Performance" src="https://github.com/user-attachments/assets/898c0a17-7064-4d9a-b03c-e21d80c7da98" />


---

### 2. **Doctor Conversion Status**

**Objective:**  
Understand whether each doctor has switched or adopted the **focus brand** instead of competitor brands.

| Conversion Status | Meaning |
|--------------------|----------|
| Converted | Doctor prescribes your Focus Brand only |
| Partially Converted | Doctor prescribes both Focus and Competitor brands |
| Not Converted | Doctor prescribes only competitor brands |

#### DAX Measures

**Step 1: Doctor-level measures**
```DAX
Focus Rx Qty = SUM(Focus_RCPA_Data[Rx_Qty])
Competitor Rx Qty = SUM(Competitor_RCPA_Data[Rx_Qty])
```

**Step 2: Conversion Status**
```DAX
Doctor Conversion Status =
VAR FocusQty = [Focus Rx Qty]
VAR CompQty = [Competitor Rx Qty]
RETURN
    SWITCH(
        TRUE(),
        FocusQty > 0 && CompQty = 0, "Converted",
        FocusQty > 0 && CompQty > 0, "Partially Converted",
        FocusQty = 0 && CompQty > 0, "Not Converted",
        "No Data"
    )
```

**Step 3: Doctor Dimension Table**
```DAX
Doctors =
DISTINCT(
    UNION(
        SELECTCOLUMNS(Focus_RCPA_Data, "Doctor_Name", Focus_RCPA_Data[Doctor_Name]),
        SELECTCOLUMNS(Competitor_RCPA_Data, "Doctor_Name", Competitor_RCPA_Data[Doctor_Name])
    )
)
```

#### Matrix Visual
| Column | Source / Measure |
|---------|------------------|
| Doctor_Name | Doctors[Doctor_Name] |
| Rep_Name | Focus_RCPA_Data[Rep_Name] |
| Region | Focus_RCPA_Data[Region] |
| Focus Rx | [Focus Rx Qty] |
| Competitor Rx | [Competitor Rx Qty] |
| Conversion Status | [Doctor Conversion Status] |

<img width="811" height="771" alt="Doctor conversion status" src="https://github.com/user-attachments/assets/80d4a5f9-1af7-4129-912e-6817d4b722c1" />

---

### 3. **Brand Competition Analysis**

**Objective:**  
Analyze how each brand (focus vs competitor) performs by **region**.

#### Key Questions
- Which brands dominate each region?  
- How do competitors compare to our focus brand?  

#### Step 1: Create Region Dimension Table
```DAX
Region_Dim =
DISTINCT(
    UNION(
        SELECTCOLUMNS(Focus_RCPA_Data, "Region", Focus_RCPA_Data[Region]),
        SELECTCOLUMNS(Competitor_RCPA_Data, "Region", Competitor_RCPA_Data[Region])
    )
)
```

#### Step 2: Define Relationships
| From | To | Type |
|------|----|------|
| Region_Dim[Region] | Focus_RCPA_Data[Region] | 1 → * |
| Region_Dim[Region] | Competitor_RCPA_Data[Region] | 1 → * |

#### Step 3: Create DAX Measures
```DAX
Focus Brand Rx Qty = SUM(Focus_RCPA_Data[Rx_Qty])
Competitor Rx Qty = SUM(Competitor_RCPA_Data[Rx_Qty])
Total Rx Qty = [Focus Brand Rx Qty] + [Competitor Rx Qty]

Focus Brand Share % = DIVIDE([Focus Brand Rx Qty], [Total Rx Qty], 0)
Competitor Share % = DIVIDE([Competitor Rx Qty], [Total Rx Qty], 0)
```

#### Step 4: Visualization Setup
| Axis | Legend | Values |
|-------|---------|--------|
| Region | Brand (from Product_Master or Competitor table) | Rx Qty (Focus & Competitor) |

#### Interpretation
- **Focus Brand Share % > 60%** → Strong dominance  
- **Competitor Share % > Focus Share %** → Target for improvement  
- Combine with **Brand Targets** to align achievement with regional competition.  

<img width="1313" height="738" alt="Brand Competition per Region" src="https://github.com/user-attachments/assets/1eb1461a-220a-4275-b6f0-8e414b05e6a6" />

---

## 🧠 Insights & Takeaways
- Identify **top-performing doctors** and **sales reps** by Rx volume.  
- Detect **conversion trends** among doctors.  
- Monitor **brand share** and **regional dominance**.  
- Support **data-driven marketing** and **sales planning** decisions.

---

## 🧩 Tools & Technologies
- **Power BI Desktop**  
- **Power Query**  
- **DAX (Data Analysis Expressions)**  
- **Excel / CSV Data Sources**

Our Dashboard 
<img width="1313" height="738" alt="Brand Competition per Region" src="https://github.com/user-attachments/assets/989c3a96-c32b-4cfd-84ef-98874207af51" />

---

## 👨‍💻 Author
**Felix Amenya**  
Power BI Developer | Data Analyst  
📧 Kenyansafelix"gmail.com  
📍 Kenya  
