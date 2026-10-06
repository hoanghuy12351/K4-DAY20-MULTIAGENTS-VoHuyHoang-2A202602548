---
name: validate-data-format
description: Use when you need to ensure that the data format in input files meets the expected structure before processing.
---
1. Define the expected schema for the data, including required columns and their data types.
2. Read the input data file and check for the presence of all required columns.
3. Validate the data types of each column against the expected types.
4. If any discrepancies are found (missing columns or incorrect types), log an error message detailing the issues.
5. Stop further processing until the data format is corrected.
6. If the data format is valid, proceed to the next step of the task.
