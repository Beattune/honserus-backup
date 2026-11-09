ACCEL_HOMING_TROUBLESHOOT evidence

Share the entire folder or ZIP. Read manifest.json first.
home-*.csv.gz: every retained raw decoded XYZ sample before graph reduction, in g including gravity/bias.
packets.jsonl.gz: bounded callback timing, pending/filtered packet context.
manifest.json: home/return/next-home linkage, detector/reference/quality metadata, errors, config, installed source SHA256, bounds.
Each home records capture/core drops and recorder omissions. Missing is not zero.
No physical-position sensor is present; travel is commanded step accounting.
Stop markers distinguish sensor print_time from host dispatch. Hardware receipt is unknown.
PNG failures do not remove raw evidence. Discovery/rejections/outliers remain.
