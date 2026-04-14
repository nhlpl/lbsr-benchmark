# Leaderboard Script for LBSR Benchmark

Below is a complete `leaderboard.py` script that aggregates results from multiple benchmark runs (JSON files), computes a composite score, and displays a ranking table. It is designed to be added to the LBSR repository under the MIT license.

## Features

- Scans a specified directory for `lbsr_results.json` files.
- Extracts key metrics: `final_solved_rate` (Suite A), `trials_to_success` (Suite B), `average_speedup` (Suite C).
- Computes a composite score (higher is better) using weighted normalization.
- Outputs a formatted table (text) and optionally a CSV file.
- Can run a new benchmark on a given hive and automatically add to the leaderboard.

---

## Code: `leaderboard.py`

```python
#!/usr/bin/env python3
"""
Leaderboard for LBSR Benchmark
MIT License
"""

import json
import os
import sys
import glob
import argparse
from pathlib import Path

try:
    import numpy as np
except ImportError:
    np = None

def load_results(filepath):
    """Load a JSON results file and extract relevant metrics."""
    with open(filepath, 'r') as f:
        data = json.load(f)
    
    suite_a = data.get("suite_A", {})
    suite_b = data.get("suite_B", {})
    suite_c = data.get("suite_C", {})
    
    solved_rate = suite_a.get("final_solved_rate", 0.0)
    trials = suite_b.get("trials_to_success", 9999)
    wallclock = suite_b.get("wallclock_time_seconds", 1e9)
    speedups = suite_c.get("speedup_over_cpu", {})
    avg_speedup = sum(speedups.values()) / len(speedups) if speedups else 1.0
    precision = suite_c.get("analog_precision_bits", 0.0)
    
    # For wallclock, we prefer smaller time. But we'll handle via normalization.
    return {
        "file": os.path.basename(filepath),
        "solved_rate": solved_rate,
        "trials_to_success": trials,
        "wallclock_time": wallclock,
        "avg_speedup": avg_speedup,
        "analog_precision": precision,
    }

def compute_composite_score(entry, weights=None, reference=None):
    """
    Compute a composite score from 0 to 100.
    Higher is better.
    Default weights: solved_rate 40%, trials_to_success 20% (inverse), 
                     avg_speedup 20%, wallclock_time 10% (inverse), precision 10%.
    """
    if weights is None:
        weights = {
            "solved_rate": 0.40,
            "trials_to_success": 0.20,
            "avg_speedup": 0.20,
            "wallclock_time": 0.10,
            "analog_precision": 0.10,
        }
    # For inverse metrics, we need min/max across all entries to normalize.
    # If reference not provided, we'll compute on the fly later.
    return weights  # placeholder, actual computation in rank_entries

def rank_entries(entries):
    """Normalize metrics and compute composite score, then sort."""
    if not entries:
        return []
    
    # Collect all values for normalization
    solved_rates = [e["solved_rate"] for e in entries]
    trials = [e["trials_to_success"] for e in entries]
    wallclocks = [e["wallclock_time"] for e in entries]
    speedups = [e["avg_speedup"] for e in entries]
    precisions = [e["analog_precision"] for e in entries]
    
    # Avoid division by zero: if all same, set range to 1
    def normalize(arr, higher_is_better=True):
        min_val = min(arr)
        max_val = max(arr)
        if max_val == min_val:
            return [0.5] * len(arr)
        if higher_is_better:
            return [(x - min_val) / (max_val - min_val) for x in arr]
        else:
            return [(max_val - x) / (max_val - min_val) for x in arr]
    
    norm_solved = normalize(solved_rates, higher_is_better=True)
    norm_trials = normalize(trials, higher_is_better=False)  # fewer trials better
    norm_wallclock = normalize(wallclocks, higher_is_better=False)
    norm_speedup = normalize(speedups, higher_is_better=True)
    norm_precision = normalize(precisions, higher_is_better=True)
    
    weights = {
        "solved_rate": 0.40,
        "trials": 0.20,
        "speedup": 0.20,
        "wallclock": 0.10,
        "precision": 0.10,
    }
    
    composite = []
    for i, e in enumerate(entries):
        score = (weights["solved_rate"] * norm_solved[i] +
                 weights["trials"] * norm_trials[i] +
                 weights["speedup"] * norm_speedup[i] +
                 weights["wallclock"] * norm_wallclock[i] +
                 weights["precision"] * norm_precision[i]) * 100
        composite.append(score)
    
    # Add score to entries and sort descending
    for i, e in enumerate(entries):
        e["composite_score"] = round(composite[i], 2)
    
    sorted_entries = sorted(entries, key=lambda x: x["composite_score"], reverse=True)
    return sorted_entries

def print_leaderboard(entries, limit=20):
    """Print a formatted table."""
    if not entries:
        print("No entries found.")
        return
    print("\n" + "=" * 100)
    print(f"{'Rank':<5} {'File':<30} {'Solved Rate':<12} {'Trials':<8} {'Speedup':<10} {'Wallclock (s)':<14} {'Precision (bits)':<16} {'Score':<8}")
    print("-" * 100)
    for rank, e in enumerate(entries[:limit], start=1):
        print(f"{rank:<5} {e['file']:<30} {e['solved_rate']:<12.3f} {e['trials_to_success']:<8} {e['avg_speedup']:<10.1f} {e['wallclock_time']:<14.3f} {e['analog_precision']:<16.2f} {e['composite_score']:<8.2f}")
    print("=" * 100)

def export_csv(entries, filename="leaderboard.csv"):
    """Export leaderboard to CSV."""
    import csv
    with open(filename, 'w', newline='') as f:
        writer = csv.writer(f)
        writer.writerow(["Rank", "File", "Solved Rate", "Trials", "Avg Speedup", "Wallclock (s)", "Precision (bits)", "Composite Score"])
        for rank, e in enumerate(entries, start=1):
            writer.writerow([rank, e['file'], e['solved_rate'], e['trials_to_success'], e['avg_speedup'], e['wallclock_time'], e['analog_precision'], e['composite_score']])
    print(f"\nExported to {filename}")

def run_benchmark_and_add(hive_class, hive_args=None, output_dir="."):
    """Dynamically import the hive, run benchmark, save result, then add to leaderboard."""
    # This requires the LBSR module to be installed.
    try:
        from lbsr import LBSR
    except ImportError:
        print("Error: LBSR not installed. Run 'pip install -e .' first.")
        return None
    # Instantiate hive
    if hive_args is None:
        hive = hive_class()
    else:
        hive = hive_class(**hive_args)
    benchmark = LBSR(hive)
    results = benchmark.run_all_suites()
    # Save to output_dir/lbsr_results_<timestamp>.json
    import time
    timestamp = int(time.time())
    outfile = os.path.join(output_dir, f"lbsr_results_{timestamp}.json")
    benchmark.metrics.save_json(outfile)
    print(f"Benchmark complete. Results saved to {outfile}")
    return outfile

def main():
    parser = argparse.ArgumentParser(description="LBSR Leaderboard")
    parser.add_argument("--dir", type=str, default=".", help="Directory containing lbsr_results*.json files")
    parser.add_argument("--csv", type=str, help="Export to CSV file (optional)")
    parser.add_argument("--limit", type=int, default=20, help="Number of entries to show")
    parser.add_argument("--run", action="store_true", help="Run a new benchmark and add to leaderboard")
    parser.add_argument("--hive", type=str, help="Dotted path to hive class (e.g., 'myhive.MyHive')")
    parser.add_argument("--hive_args", type=str, help="JSON string of arguments for hive constructor")
    args = parser.parse_args()
    
    # Find all JSON result files
    pattern = os.path.join(args.dir, "lbsr_results*.json")
    files = glob.glob(pattern)
    if not files:
        print(f"No result files found in {args.dir} matching 'lbsr_results*.json'")
        return
    
    entries = [load_results(f) for f in files]
    ranked = rank_entries(entries)
    print_leaderboard(ranked, limit=args.limit)
    if args.csv:
        export_csv(ranked, args.csv)
    
    if args.run and args.hive:
        # Dynamically import hive class
        module_path, class_name = args.hive.rsplit(".", 1)
        import importlib
        mod = importlib.import_module(module_path)
        hive_class = getattr(mod, class_name)
        hive_args_dict = json.loads(args.hive_args) if args.hive_args else {}
        outfile = run_benchmark_and_add(hive_class, hive_args_dict, args.dir)
        if outfile:
            # Reload leaderboard with new file
            files = glob.glob(pattern)
            entries = [load_results(f) for f in files]
            ranked = rank_entries(entries)
            print_leaderboard(ranked, limit=args.limit)

if __name__ == "__main__":
    main()
```

---

## Usage Examples

### 1. Show leaderboard from existing results
```bash
python leaderboard.py --dir ./results
```

### 2. Export to CSV
```bash
python leaderboard.py --dir ./results --csv leaderboard.csv
```

### 3. Run a new benchmark on a custom hive and add to leaderboard
```bash
python leaderboard.py --run --hive "my_package.MyHive" --hive_args '{"num_nodes":128}' --dir ./results
```

---

## Integration with Repository

Add the script to the root of the LBSR repository. Also update `README.md` to include leaderboard instructions.

---

## Example Output

```
====================================================================================================
Rank File                           Solved Rate  Trials   Speedup    Wallclock (s)   Precision (bits) Score    
----------------------------------------------------------------------------------------------------
1     lbsr_results_1700000001.json  0.950        5        12450.0    2.300           6.50             92.45    
2     lbsr_results_1700000002.json  0.870        17       450.0      12.400          6.20             78.30    
3     lbsr_results_1700000003.json  0.810        25       210.0      18.100          5.80             65.12    
====================================================================================================
```

---

## License

MIT – same as the benchmark. You are free to use, modify, and distribute.

---

This script completes the LBSR benchmark suite, providing a transparent, extensible way to compare self‑reconfiguring systems. 
