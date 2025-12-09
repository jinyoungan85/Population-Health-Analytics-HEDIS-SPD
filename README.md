# Population Health Analytics: HEDIS SPD Care Gap Analysis

## 📌 Project Overview
**Objective:** To automate the identification of care gaps for **Statin Therapy for Patients with Diabetes (SPD)**, a key HEDIS quality measure, while addressing real-world data challenges such as alert fatigue and unstructured clinical documentation.

**Business Problem:**
Insurance providers and health systems frequently audit charts for diabetic patients aged 40-75 who are not on statin therapy. Relying solely on basic queries leads to high **false-positive rates** because valid clinical reasons for discontinuation (e.g., intolerance, myalgia) are often documented in free-text notes rather than structured diagnosis codes.

**My Solution (PharmD & Data Analyst Approach):**
I developed a SQL logic that not only identifies therapy gaps but also:
1.  **Classifies Statin Intensity** (High vs. Moderate) based on ACC/AHA guidelines.
2.  **Filters out False Positives** by checking for contraindications (e.g., Rhabdomyolysis, Myalgia).
3.  **Scans Unstructured Data** (Clinical Notes) to catch un-coded intolerance history.

---

## 🧠 Clinical Logic & Workflow
This project utilizes my **Pharm.D. background** to define precise inclusion/exclusion criteria.



* **Inclusion Criteria (The Denominator):**
    * Age: 40 - 75 years
    * Diagnosis: Type 2 Diabetes (ICD-10 `E11.%`)
* **Statin Intensity Classification:**
    * **High Intensity:** Atorvastatin $\ge$ 40mg, Rosuvastatin $\ge$ 20mg
    * **Moderate Intensity:** Simvastatin, Pravastatin, lower doses of above.
* **Exclusion Criteria (The Safety Valve):**
    * Structured: ICD-10 codes for Myalgia (`M79.1`), Rhabdomyolysis (`M62.82`).
    * Unstructured: Keyword search in provider notes (e.g., "muscle pain", "cramps").

---

## 💻 SQL Logic
*Note: This query simulates a production environment analysis.*

```sql
/*
Project: HEDIS SPD Gap Analysis & Intensity Review
Author: Jin-Young An, PharmD
Logic: Identify diabetic patients needing statin therapy while excluding those with intolerance.
*/

WITH Target_Population AS (
    -- 1. Identify Denominator: Patients 40-75 with T2DM
    SELECT
        p.patient_id,
        TIMESTAMPDIFF(YEAR, p.birth_date, CURRENT_DATE) AS age
    FROM patients p
    JOIN diagnoses d ON p.patient_id = d.patient_id
    WHERE d.icd_code LIKE 'E11%' -- ICD-10 Family for Type 2 Diabetes
      AND TIMESTAMPDIFF(YEAR, p.birth_date, CURRENT_DATE) BETWEEN 40 AND 75
),

Exclusions AS (
    -- 2. Identify Exclusions: Reducing False Positives
    -- A. Structured Data Check
    SELECT DISTINCT patient_id, 'ICD-10' as source
    FROM diagnoses
    WHERE icd_code IN ('M79.1', 'M62.82', 'G72.0') -- Myalgia, Myopathy, Rhabdo

    UNION

    -- B. Unstructured Data Check (Text Mining)
    -- Catches cases where providers noted intolerance in free text but didn't assign an ICD code.
    SELECT DISTINCT patient_id, 'Clinical Note' as source
    FROM clinical_notes
    WHERE LOWER(note_text) LIKE '%statin intolerance%'
       OR LOWER(note_text) LIKE '%muscle pain%'
       OR LOWER(note_text) LIKE '%stopped%due to%cramp%'
),

Medication_Check AS (
    -- 3. Check Pharmacy Claims & Determine Intensity
    SELECT
        tp.patient_id,
        m.med_name,
        m.dosage_mg,
        CASE
            WHEN m.med_name IN ('Atorvastatin', 'Rosuvastatin') AND m.dosage_mg >= 40 THEN 'High'
            WHEN m.med_name IS NOT NULL THEN 'Moderate/Low'
            ELSE 'None'
        END AS intensity
    FROM Target_Population tp
    LEFT JOIN medications m
        ON tp.patient_id = m.patient_id
        AND m.med_category = 'Statin'
        AND m.status = 'Active'
)

-- 4. Final Actionable Report
SELECT
    mc.patient_id,
    COALESCE(mc.med_name, 'None') AS current_med,
    CASE
        WHEN ex.patient_id IS NOT NULL THEN 'Excluded (Intolerance)'
        WHEN mc.intensity = 'None' THEN 'GAP: Needs Therapy 🚨'
        WHEN mc.intensity = 'Moderate/Low' THEN 'Review: Optimization Oppty ⚠️'
        ELSE 'Compliant ✅'
    END AS clinical_status,
    ex.source AS exclusion_reason
FROM Medication_Check mc
LEFT JOIN Exclusions ex ON mc.patient_id = ex.patient_id
ORDER BY clinical_status;
