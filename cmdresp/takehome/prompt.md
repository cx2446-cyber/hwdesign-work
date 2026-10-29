# Programmable FIR Filter

Build a Vitis HLS kernel in this folder following the Waveflow stream_inband
example (https://sdrangan.github.io/waveflow/docs/examples/stream_inband/) and
the command-response pattern. Where they differ, follow the pattern.

## Operation

Compute the causal FIR:
y[n] = sum(h[k] * x[n-k] for k = 0,...,L-1).

h[0] multiplies the newest sample. Produce one output per input sample;
do not append a zero-padded filter tail. Initially, all input history is zero.
Preserve history across PROCESS commands within one activation. Each valid
CONFIG replaces all coefficients and clears the history. Nothing persists
between activations.

## Requirements

Samples and coefficients are signed 16-bit two's-complement integers.
Accumulate exactly and return signed 64-bit integer outputs, with no
rounding, saturation or wraparound.

Support 1 <= L <= 8 taps and 1 <= N <= 256 samples per PROCESS.
For L=8 and N=256 with no input stalls or output backpressure, target at
most 4096 clock cycles from the first input data transfer to the last
output data transfer, measured in RTL co-simulation.
The estimated clock target is under 10 ns; a clock-uncertainty warning
alone is not a failure.

Report missed targets as FAIL rather than redesigning to meet them.
Do not build a Python timing model.

## Commands and parameters

CONFIG: header carries tx_id and L, followed by L coefficients in order
h[0],...,h[L-1] in one data burst. After validating the complete burst,
install the coefficients, clear history and return a header-only acknowledgement.

PROCESS: header carries tx_id and N, followed by N samples in time order
in one data burst. Return a response header and N output samples in order.
A valid CONFIG is required before PROCESS.

END: header only; return without a response. A valid CONFIG is required
before END, so each activation begins with CONFIG.

Check header framing, parameter ranges and configuration state before
reading payload data. Buffer each payload until its complete framing is
validated; send no response for an invalid command or payload.
Each response echoes tx_id. Choose command codes, header layouts, word
packing and file formats consistently across model, kernel and testbench.
Ignore unused input lanes in the final packed word and zero unused output
lanes. TLAST belongs on the final required word of each burst.

## Errors

Errors go in AXI-Lite status registers (halted, error code, tx_id). On an
error the kernel closes any output burst it started with TLAST and returns
without reading more input. Nothing persists between activations: the status
registers are cleared at the start of each one.

Use distinct error codes for malformed header framing, unknown command,
invalid L or N, missing configuration, early payload TLAST, and missing
payload TLAST. Check in that priority order. Header framing errors include
early TLAST and TLAST absent at the expected header end.

Missing payload TLAST means the final expected word arrives with TLAST=0.
Do not read another word or drain the stream to diagnose it. No timeout
is required for a host that stops sending data.

On any error, send no response for the offending command; earlier completed
responses remain intact. Report its tx_id, or zero if tx_id was not decoded.
The host resets the stream path before a fresh activation.

## Verification

Independently check the Python model against these hand cases:
- h=[1,2], x=[3,4,5] gives y=[3,10,13].
- Splitting that input into PROCESS [3] and PROCESS [4,5] gives the same result.
- After reconfiguring to h=[-1], x=[2,-3] gives y=[-2,3].

Also compare against NumPy convolve using int64 operands, taking only the
first len(x) outputs. Concatenate PROCESS inputs between CONFIG commands
for this independent reference.

Cover L=1 and 8, N=1 and 256, zeros, impulses, all-negative values, signed
extremes, and results exceeding signed 32-bit range. Test multiple PROCESS
commands, reconfiguration from long to short filters and back within one
activation, and CONFIG followed immediately by END.

Test every error, including L=0 and 9, N=0 and 257, PROCESS and END before
CONFIG, header framing errors, and early/missing TLAST on both coefficient
and sample payloads. Include combined faults to check error priority and an
error after a successful PROCESS to check preservation of earlier responses.
Follow each error activation with a fresh valid activation and check that
status and filter history were reset.

Compare the model, C simulation and co-simulation bit for bit. Run
co-simulation once: one testbench run that walks through every scenario in
order, calling the kernel once per activation with fresh streams, including
each error activation and the fresh activation after it.

## results.md

Write results.md with these sections:

- Interface: the command and response layouts, how values pack into words,
  the status registers and the error codes.
- Design: the loop structure, where state is kept, and how the throughput
  target is reached.
- Targets: each target above with the required value, the measured value,
  PASS/FAIL and the file it came from (co-simulation for timing; synthesis
  for clock, II and resources). Always report the latency and throughput
  from co-simulation and the estimated clock and resources from synthesis,
  writing "no target" where this prompt sets none.
- Verification: the model's check against the hand-computed examples and the
  reference; then one row per scenario saying whether the model, C simulation
  and co-simulation agree, and for each error scenario the status it set and
  whether the fresh activation after it passed. Name the source file of each.
- Assumptions: every place this prompt was silent or ambiguous and what you
  chose, apart from interface details already given under Interface.
- Open Items: questions you could not settle, and anything that failed or was
  not done, or "None".
- Run History: one row per run, with the prompt version, what was built or
  changed, and what passed, so that a later run can continue from here.

Every number and PASS/FAIL in Targets and Verification names the file it came
from (a log, a comparison output or a synthesis report), and that file stays in
this folder to be committed: a reviewer opens them to check results.md.

## Revision history

| Version | Change | Why |
| --- | --- | --- |
| 1 | Initial FIR specification. | Defines arithmetic, limits, history, reconfiguration, errors and independent checks before implementation. |
