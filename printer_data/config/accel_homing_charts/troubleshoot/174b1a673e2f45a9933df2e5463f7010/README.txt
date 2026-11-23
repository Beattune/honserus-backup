ACCEL_HOMING_TROUBLESHOOT evidence

Share the entire folder or ZIP. Read manifest.json first.
home-*.csv.gz: every retained raw decoded XYZ sample before graph reduction, in g including gravity/bias.
Optional filtered_* columns: causal notch after resting-offset subtraction; blank before arm. Coefficients/rate/start identity are in manifest.json.
packets.jsonl.gz: bounded callback timing, pending/filtered packet context.
timing.jsonl.gz: MCU host receive, driver/query/decode, detector/observer, reactor and GC timing. See manifest timing hook availability and omissions.
delivery.json / delivery.csv: per-home source-batch tail, aggregate delivery/driver, adapter/detector and host request timing; clock bases and missing evidence are explicit.
MODE=COMPARE also provides comparison.json/.csv, tester-summary.txt, noise-margin.json and comparison.png: observed intervention, paired deltas, margins, omissions and restoration.
manifest.json: home/return/next-home linkage, detector/reference/quality metadata, errors, config, installed source SHA256, bounds.
debug/: bounded host logs and runtime/configuration context, with missing/truncated-source details in debug/context.json.
Each home records capture/core drops and recorder omissions. Missing is not zero.
No physical-position sensor is present; travel is commanded step accounting.
Stop markers distinguish sensor print_time from host dispatch. Hardware receipt is unknown.
PNG failures do not remove raw evidence. Discovery/rejections/outliers remain.
