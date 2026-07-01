# Asynchronous FIFO Design and Verification

This project implements an **Asynchronous FIFO** using Verilog HDL.  
The FIFO supports independent write and read clocks, making it suitable for **Clock Domain Crossing (CDC)** applications.

The design was verified using a self-checking Verilog testbench and waveform analysis in GTKWave.

---

## Project Overview

An asynchronous FIFO is used when data must be transferred safely between two different clock domains.

In this project:

- Write operation happens in the `w_clk` domain.
- Read operation happens in the `r_clk` domain.
- Read and write clocks are independent.
- Binary pointers are used locally.
- Gray-coded pointers are used for clock domain crossing.
- Two-flop synchronizers are used to safely transfer Gray-coded pointers across clock domains.
- Full and empty conditions are generated using synchronized Gray pointers.

---

## FIFO Features

- 8-bit data width
- 8-depth FIFO memory
- Separate read and write clocks
- Separate read and write enable signals
- Full flag generation
- Empty flag generation
- Binary to Gray pointer conversion
- Two-stage pointer synchronizers
- Protection against overflow
- Protection against underflow
- Pointer wrap-around support

---

## Design Files

The FIFO design is divided into the following modules:

| Module | Description |
|---|---|
| `write` | Handles write pointer logic and FIFO full generation |
| `read` | Handles read pointer logic and FIFO empty generation |
| `sync` | Synchronizes Gray-coded pointers between clock domains |
| `memo` | FIFO memory block |
| `connect` | Top-level module connecting all submodules |

---

## Block Diagram

```text
                 Write Clock Domain                     Read Clock Domain
              -----------------------                 ----------------------
 data_in ---> | FIFO Memory Write |                   | FIFO Memory Read | ---> data_out
              -----------------------                 ----------------------
                       |                                      |
                       v                                      v
                 Binary Write Ptr                       Binary Read Ptr
                       |                                      |
                       v                                      v
                  Gray Write Ptr                        Gray Read Ptr
                       |                                      |
                       |                                      |
                       v                                      v
              2-Flop Synchronizer                  2-Flop Synchronizer
              to Read Clock Domain                 to Write Clock Domain
                       |                                      |
                       v                                      v
                 Empty Logic                          Full Logic
```

---

## Write Pointer and Full Logic

The write module works in the write clock domain.

The binary write pointer is converted into Gray code before crossing into the read clock domain.

```verilog
assign g_wptr = (b_wptr ^ (b_wptr >> 1));
assign b_wptr_next = b_wptr + write_allow;
assign g_wptr_next = (b_wptr_next ^ (b_wptr_next >> 1));
```

The FIFO full condition is detected by comparing the next Gray-coded write pointer with the synchronized read pointer.

```verilog
if(g_wptr_next == {~g_rptr_sync[3:2], g_rptr_sync[1:0]})
    fifofull <= 1'b1;
else
    fifofull <= 1'b0;
```

This condition checks whether the FIFO is full by comparing the next write pointer with the read pointer after inverting the upper two bits.

---

## Read Pointer and Empty Logic

The read module works in the read clock domain.

The binary read pointer is converted into Gray code before crossing into the write clock domain.

```verilog
assign g_rptr = (b_rptr ^ (b_rptr >> 1));
assign b_rptr_next = b_rptr + read_allow;
assign g_rptr_next = (b_rptr_next ^ (b_rptr_next >> 1));
```

The FIFO empty condition is detected by comparing the next Gray-coded read pointer with the synchronized write pointer.

```verilog
assign empty = (g_rptr_next == g_wptr_sync);
```

If both pointers are equal, the FIFO is empty.

---

## CDC Handling

Clock Domain Crossing is handled using Gray-coded pointers and two-flop synchronizers.

Only Gray-coded pointers are transferred between clock domains:

- `g_wptr` crosses from write clock domain to read clock domain.
- `g_rptr` crosses from read clock domain to write clock domain.

Binary pointers are not directly transferred between clock domains.

---

## Synchronizer Logic

The synchronizer module uses two flip-flops for each CDC path.

Write pointer synchronized into read clock domain:

```verilog
always @(posedge r_clk)
begin
    if(reset)
    begin
        FFR1 <= 4'd0;
        FFR2 <= 4'd0;
    end
    else
    begin
        FFR1 <= g_wptr;
        FFR2 <= FFR1;
    end
end
```

Read pointer synchronized into write clock domain:

```verilog
always @(posedge w_clk)
begin
    if(reset)
    begin
        FFW1 <= 4'd0;
        FFW2 <= 4'd0;
    end
    else
    begin
        FFW1 <= g_rptr;
        FFW2 <= FFW1;
    end
end
```

This reduces metastability risk while passing pointer information between asynchronous clock domains.

---

## FIFO Memory

The memory stores 8-bit data with 8 locations.

```verilog
reg [7:0] mem [7:0];
```

Write operation:

```verilog
if(w_en && !fifofull)
    mem[b_wptr[2:0]] <= data_in;
```

Read operation:

```verilog
if(r_en && !fifoempty)
    data_out <= mem[b_rptr[2:0]];
```

The FIFO does not write new data when `fifofull` is high.  
The FIFO does not read invalid data when `fifoempty` is high.

---

## Testbench Verification

A self-checking Verilog testbench was written to verify the FIFO operation.

The testbench checks:

- Reset condition
- Empty flag after reset
- Read from empty FIFO
- Simple write and read operation
- FIFO full condition
- Write attempt when FIFO is full
- Correct read order
- Pointer wrap-around
- Independent read and write clocks

---

## Testbench Clock Setup

The testbench uses two different clocks.

```verilog
always #5 w_clk = ~w_clk;   // Write clock = 10 ns
always #7 r_clk = ~r_clk;   // Read clock = 14 ns
```

Since the write and read clocks have different time periods, the FIFO is tested under asynchronous clock conditions.

---

## Important Test Case: Write When FIFO Is Full

In this test, the FIFO is filled completely first.  
After that, `data_in = 8'hEE` is applied while the FIFO is full.

```verilog
data_in = 8'hEE;
w_en = 1'b1;
```

Since the FIFO is full, this data must not be written into memory.

The write condition prevents the data from being stored:

```verilog
if(w_en && !fifofull)
    mem[b_wptr[2:0]] <= data_in;
```

During waveform analysis, `EE` appears on `data_in`, but it does not appear on `data_out`.  
This confirms that overflow protection is working correctly.

---

## Simulation Commands

The design was compiled and simulated using Icarus Verilog.

```bash
iverilog -o async_fifo_tb async_fifo.v tb_async_fifo.v
vvp async_fifo_tb
gtkwave async_fifo_tb.vcd
```

---

## Simulation Result

The testbench displays pass or fail messages for each test case.

Expected final output:

```text
ALL TESTS PASSED
```

This confirms that the FIFO passed all functional test cases.

---

## Screenshot 1: Initial FIFO Operation

Paste your first GTKWave screenshot here.

This screenshot should show:

- Reset behavior
- Write and read clocks
- Write enable and read enable
- Data written into FIFO
- Data read from FIFO
- Empty flag behavior
- Full flag behavior
- Write attempt with `EE` during full condition

```markdown
![GTKWave Screenshot 1](images/gtkwave_part1.png)
```

---

## Screenshot 2: Pointer Wrap-Around Verification

Paste your second GTKWave screenshot here.

This screenshot should show:

- Continuous write and read operation
- Pointer incrementing
- Pointer wrap-around
- Correct data order from `data_in` to `data_out`
- Empty flag changing correctly

```markdown
![GTKWave Screenshot 2](images/gtkwave_part2.png)
```

---

## Screenshot 3: Terminal Output

Paste your terminal output screenshot here.

This screenshot should show that all tests passed.

```markdown
![Terminal Output](images/simulation_passed.png)
```

---

## GTKWave Signals Observed

The following signals were observed in GTKWave:

| Signal | Purpose |
|---|---|
| `w_clk` | Write clock |
| `r_clk` | Read clock |
| `w_en` | Write enable |
| `r_en` | Read enable |
| `data_in[7:0]` | Input data |
| `data_out[7:0]` | Output data |
| `fifoempty` | FIFO empty flag |
| `fifofull` | FIFO full flag |
| `b__wpt[3:0]` | Binary write pointer |
| `b__rpt[3:0]` | Binary read pointer |

---

## CDC Verification Explanation

The design uses a standard CDC-safe asynchronous FIFO structure.

CDC safety is handled by:

- Using Gray-coded pointers for crossing clock domains
- Passing Gray pointers through two-flop synchronizers
- Keeping binary pointers local to their own clock domains
- Generating full and empty flags only after pointer synchronization

Functional simulation verifies that the FIFO works correctly with independent clocks.

For complete industrial CDC signoff, a static CDC verification tool can also be used.  
However, structurally, this design follows the standard asynchronous FIFO CDC method.

---

## Final Result

The asynchronous FIFO was successfully designed and verified.

Verified cases:

- FIFO reset
- Empty condition
- Full condition
- Correct write operation
- Correct read operation
- Overflow protection
- Underflow protection
- Data order preservation
- Pointer wrap-around
- Asynchronous read and write clocks

The simulation completed successfully with:

```text
ALL TESTS PASSED
```

---

## Conclusion

This project demonstrates the design and verification of an asynchronous FIFO using Verilog.

The FIFO safely transfers data between two independent clock domains by using Gray-coded pointers and two-stage synchronizers.  
The self-checking testbench and GTKWave waveform analysis confirm correct FIFO functionality.
