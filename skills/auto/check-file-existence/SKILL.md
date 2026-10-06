---
name: check-file-existence
description: Use when you need to ensure that required files are present before executing data processing tasks.
---
1. Identify all required input files for the task.
2. For each required file, check if it exists in the specified directory.
3. If any file is missing, log an error message indicating which file is not found.
4. Stop further execution of the task until all required files are available.
5. If all files are present, proceed to the next step of the task.
6. Optionally, provide a summary of the files checked and their statuses.
