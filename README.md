# zip-flattening

## Parallelism & Cluster Sizing Guidance
This notebook performs ZIP discovery, extraction, and file flattening using shell-based processing (%sh) on the Databricks driver node. 

The parallelism parameter controls how many ZIP files are processed concurrently via xargs -P.

### Recommended parallelism settings

| Driver CPU Cores   | Reccommended Parallelism |
| ------------------ | ------------------------ |
| 4 cores            | 2-4                      |
| 8 cores            | 4-8                      |  
| 16 cores           | 8-16                     |

General rule of thumb:
Set parallelism approximately equal to the number of driver CPU cores.

Guidance by data shape:
	•	Many small ZIPs (1–3 files each):
parallelism = 4–8
	•	Fewer ZIPs with many files inside (high fan-out):
parallelism = 2–4
	•	Large ZIPs (few but big):
parallelism = 2–4
	•	Mixed ERP drops (common case):
parallelism = 4 (safe default)

Increasing parallelism beyond this often yields diminishing returns due to storage metadata contention.

⸻

Cluster Sizing

This workload does not benefit from Spark workers.

Recommended baseline:
	•	Driver: 8 vCPU / 32GB RAM
	•	Workers: 0–1
	•	Parallelism: 4

For larger or more complex drops, prefer increasing driver CPU before increasing parallelism. 


## Tested Scenarios


### Single-file ZIPs (1:1)

Scenario
	•	Each ZIP contained exactly 1 TXT file

Expected Behaviour
	•	Each TXT file is extracted
	•	All files land directly in the target root directory

Outcome: **Pass**

All TXT files were successfully unpacked and written to the base output folder. 



### Multi-file ZIPs (1:N expansion)

Scenario
	•	Multiple Zip Files
	•	Each ZIP File contains > 1 TXT files

Expected Behaviour
	•	All TXT files extracted
	•	Flattened into the base output directory

Outcome: **Pass**

Each ZIP expanded correctly, resulting in 15 TXT files written to the root folder.  



### Filename collisions across ZIPs

Scenario
	•	2 ZIP files
	•	Both ZIPs contained a TXT file with the same filename

Expected Behaviour
	•	No overwrite
	•	Files uniquely named using a deterministic naming strategy

Naming Strategy

`<ZipName>__<Hash(source_path)>__<FileName>`

Outcome: **Pass**

Both files were successfully extracted and written without collision using the naming convention.  



 ### Unexpected file types inside ZIP

Scenario
	•	1 ZIP containing non-TXT files (e.g. image, JSON)

Expected Behaviour
	•	Files extracted successfully **(content type not restricted at unzip stage yet)**

Outcome: **Pass**
Unexpected file types were unpacked without issue.  



### Corrupt ZIP handling

Scenario
	•	intentionally corrupted ZIP file

Expected Behaviour
	•	ZIP should fail validation
	•	Process should continue for other ZIPs
	•	Corrupt file should not block the run

Outcome: **Expected Failure / Handled Gracefully**

The corrupt ZIP failed integrity checks and was skipped. All other ZIPs processed successfully.  


### Deeply nested ZIP location

Scenario
• ZIP file located in a deeply nested directory structure
• ZIP path example:

`deep_nesting/.../DEEP_NEST_ZIP_LOCATION__drop001.zip`

• ZIP contained multiple TXT files

Expected Behaviour
• ZIP should be discovered regardless of directory depth
• All TXT files extracted successfully
• Extracted files flattened into the target root directory (no subfolders preserved)

Outcome: **Pass**

The ZIP was correctly discovered despite deep nesting, all TXT files were unpacked, and the extracted files were written directly to the base output folder.  



### Internal folder structure inside ZIP (colliding basenames)

Scenario
• 1 ZIP file containing an internal directory hierarchy
• ZIP members stored in nested paths (e.g. folderA/, folderB/sub/)
• Multiple TXT files shared the same basename but existed in different internal folders

Expected Behaviour
• All files should be extracted regardless of internal ZIP folder structure
• Internal directory paths should be ignored during extraction
• Files flattened into the target root directory
• Filename collisions handled using deterministic renaming

Outcome: **Pass**

All TXT files were successfully extracted from nested paths within the ZIP. Despite colliding basenames, files were flattened into the base output directory without overwrites using the configured naming strategy.  



### Zero-byte files inside ZIP

Scenario
• ZIP containing one or more zero-byte TXT files

Expected Behaviour
• Zero-byte files should be extracted successfully
• Files should not be skipped or dropped
• Zero-byte files should be written to the target root directory

Outcome: **Pass**

Zero-byte TXT files were successfully extracted and written to the base output folder with no errors or omissions.  



### High-file-count ZIP (large fan-out)

Scenario
• ZIP containing a high number of member files
• Example: MANY_SMALL_FILES__1000_members.zip

Expected Behaviour
• All member files should be extracted successfully
• Files should be flattened into the target root directory
• Temporary extraction space used to safely handle large fan-out

Outcome: **Pass**

All files were successfully extracted into a temporary directory, flattened, and written to the base output folder.  
