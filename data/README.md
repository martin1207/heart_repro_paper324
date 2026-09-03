Place here the two data files expected by the notebook:

- `clean_dataset.csv.gz`                 (frame-level trajectory / speed dataset, 30 fps)
- `clean_dataset_vru_detections.csv.gz`  (matched VRU detections)

They are stored gzip-compressed; `read.csv()` reads them directly. Use
`gunzip -k data/*.csv.gz` if you want the plain CSVs.

Both files are anonymised: `source` is an integer trip identifier. See the column
documentation in the root `README.md`.
