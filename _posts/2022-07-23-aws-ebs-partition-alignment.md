# AWS EBS Partition Alignment

Within AWS EBS GP2, what is the optimal partition alignment?
Conclusion 4096 Bytes.
Testing performed with Crystal DiskMark 8.0.4 64-bit in Windows Server 2022 on a t3.medium EC2 instance in US-East-1.

## Analysis
| Test | System Default (MB/s) | 4 KiB (MB/s) | 4000 KiB (MB/s) | 16 KiB (MB/s) | Default (MB/s) | Best (MB/s) | Worst (MB/s) | Delta (%) |
| - | - | - | - | - | - | - | - | - |
| Sequential 1 MiB Read with 8 queues & 1 threads | 137.77 | 137.39 | 137.72 | 138.81 | 138.18 | 138.81 | 137.39 | 1 |
| Sequential 1 Mib Read with 1 queues & 1 threads | 137.13 | 137.37 | 136.96 | 137.15 | 137.34 | 137.37 | 136.96 | 0 |
| Random 4 KiB Read with 32 queues & 1 threads | 12.8 | 12.82 | 12.76 | 12.68 | 12.85 | 12.85 | 12.68 | 1 |
| Random 4 KiB Read with 1 queues & 1 threads | 12.63 | 12.76 | 12.5 | 12.63 | 12.8 | 12.8 | 12.5 | 2 |
| Sequential 1 MiB Write with 8 queues & 1 threads | 137.57 | 140.48 | 138.01 | 138.18 | 137.59 | 140.48 | 137.57 | 2 |
| Sequential 1 Mib Write with 1 queues & 1 threads | 137.97 | 137.78 | 137.37 | 137.14 | 137.77 | 137.97 | 137.14 | 1 |
| Random 4 KiB Write with 32 queues & 1 threads | 12.6 | 14.09 | 11.3 | 14.3 | 14.69 | 14.69 | 11.3 | 30 |
| Random 4 KiB Write with 1 queues & 1 threads | 4.1 | 7.18 | 5.89 | 6.39 | 6.88 | 7.18 | 4.1 | 75 |

## Default System Drive
- MBR
- 1048576 B Offset

## Test 1
- GPT
- 4 KiB alignment
    ```powershell
    New-Partition -DiskNumber 1 -UseMaximumSize -Alignment 4096 -AssignDriveLetter
    ```
- 16777216 B offset

## Test 2
- GPT
- 4000 KiB alignment
    ```powershell
    New-Partition -DiskNumber 1 -UseMaximumSize -Alignment 4096000 -AssignDriveLetter
    ```
- 20480000 B offset

## Test 3
- GPT
- 16 KiB alignment
    ```powershell
    New-Partition -DiskNumber 3 -UseMaximumSize -Alignment 16384 -AssignDriveLetter
    ```
- 16777216 B offset

## Test 4
- GPT
- Default alignment
    ```powershell
    New-Partition -DiskNumber 1 -UseMaximumSize -AssignDriveLetter
    ```
- 16777216 B offset
