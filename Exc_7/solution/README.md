## Why does the checker fail in section 1?
The example job does not remember anything. Every run starts reading the
Kafka topic logs from the beginning and adds everything to the lake.
After two runs, every record is in the lake twice. The checker finds these
duplicates, so it fails. The job needs a way to remember where it stopped.

## My decision: architecture B ("the lake is the truth")
I chose B. The job finds out where it stopped by looking into the lake
itself. Each run first writes its files into a staging folder (nobody reads
it) and then moves the finished folder into the lake with one rename. Why B:

- The position is stored together with the data. There is no second
  place (like a state file) that could get out of sync with the lake.
- Rename is one single step. A run folder is either completely in the
  lake or not there at all. Readers never see half-written data.
- A crash is harmless. If a run stops while writing, only the staging
  folder is affected. The lake is unchanged, so the next run starts from the
  same position and does the work again.

## My result

def run_once():  
    start = lake_position()        - ask the lake where the last run stopped  
    new = read_new(start)          - read only the new records  
    run = write_files(new, STAGING) - write them to the staging folder  
    publish(run)                   - move the folder into the lake (one rename)  

There is nothing to save at the end, because the lake already contains the
position. The job passes all five situations of `check_all`, including the
runs that stop halfway: no record is missing and no record is copied twice.
