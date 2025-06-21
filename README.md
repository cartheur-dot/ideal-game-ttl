## ideal-game-ttl

Playing around is more than half of the fun.

# Adding a Guess Counter to the "Guess the Hidden Number" Game

To enhance the "Guess the Hidden Number" game with a **guess counter** that tracks the number of guesses (LOAD button presses) a player makes, we’ll add logic to the existing circuit (2114 RAM, LM555, 74LS73, 74LS00, 74LS244, 74LS85, SPST switches, LEDs). The counter displays the number of guesses on LEDs and resets upon a correct guess or manual reset. The goal is to minimize additional ICs while reinforcing the reinforcement learning (RL) analogy.

## Counter Design Overview
- **Purpose**: Count LOAD button presses (guesses), display the count, and reset when the player wins (correct guess) or starts a new game.
- **Display**: Use a 4-bit binary counter (0–15 guesses) with 4 LEDs to keep the design simple, avoiding complex 7-segment display drivers.
- **Logic**:
  - Increment the counter each time the LOAD button is pressed (using the same pulse as RAM read and 74LS85 comparison).
  - Reset when the 74LS85 indicates a correct guess (O_A=B = 1) or via an optional manual reset switch.
- **Components**:
  - **74LS93** (4-bit binary counter) for counting.
  - 4 LEDs for binary display.
  - Reuse existing 74LS73 and 74LS00 for control signals.

## Circuit Modifications
The counter integrates with the existing circuit by using the LOAD pulse (from 74LS73) to increment and the 74LS85’s O_A=B output to reset. Here’s the implementation:

### 1. Counter IC: 74LS93 (4-bit Binary Counter)
- **Description**: The 74LS93 is a 4-bit ripple counter with a divide-by-2 (QA) and divide-by-8 (QB–QD) section, counting from 0000 to 1111 (0–15), sufficient for tracking guesses (typically ≤4 with binary search).
- **Pinout** (14-pin DIP):
  - **CKA (pin 14)**: Clock for QA (increments on falling edge).
  - **CKB (pin 1)**: Clock for QB–QD (connect to QA for 4-bit counting).
  - **R0(1), R0(2) (pins 2, 3)**: Reset inputs (active high, both high to reset).
  - **QA–QD (pins 12, 9, 8, 11)**: Outputs (QA = LSB, QD = MSB).
  - **VCC (pin 5)**: 5V.
  - **GND (pin 10)**: Ground.
- **Connections**:
  - **Clock**: Connect CKA (pin 14) to 74LS73 Q output (pin 5, LOAD pulse for \(\overline{\text{CS}}\) and \(\overline{\text{WE}}\)). Q’s falling edge (after LOAD release) triggers the counter.
  - **CKB**: Connect to QA (pin 12) for 4-bit counting.
  - **Reset**:
    - Connect R0(1) and R0(2) (pins 2, 3) to 74LS85 O_A=B (pin 5) via a 74LS00 NAND gate for reset on correct guess.
    - Example: Use a 74LS00 NAND gate (pins 1, 2 input, pin 3 output) with O_A=B and delayed LOAD pulse (via RC or second 74LS73 flip-flop) as inputs.
  - **Outputs**: Connect QA–QD (pins 12, 9, 8, 11) to 4 LEDs (L12–L15) via 330 Ω resistors:
    - L12: QA (bit 0, LSB).
    - L13: QB (bit 1).
    - L14: QC (bit 2).
    - L15: QD (bit 3, MSB).
    - LED cathodes to GND.
  - **Decoupling**: Add 0.1 µF capacitor near pin 5 (VCC).

### 2. Reset Logic
- **Automatic Reset**:
  - Reset 74LS93 when 74LS85 O_A=B = 1 (correct guess).
  - Use a 74LS00 NAND gate to combine O_A=B with LOAD pulse (74LS73 Q):
    - Inputs: O_A=B (74LS85 pin 5), Q (74LS73 pin 5).
    - Output: To R0(1) and R0(2) (74LS93 pins 2, 3).
    - Resets counter only on correct guess with LOAD.
- **Manual Reset** (optional):
  - Add SPST push-button (S15) for manual reset (new game without correct guess).
  - Connect S15 between VCC and R0(1)/R0(2) with 10 kΩ pull-down resistors to GND.
  - When pressed, S15 sets R0(1) = R0(2) = 1, resetting to 0000.

### 3. Control Integration
- **LOAD Pulse**: 74LS73 Q (pin 5) triggers:
  - 2114 RAM read (\(\overline{\text{CS}} = 0\), \(\overline{\text{WE}} = 1\)).
  - 74LS85 comparison.
  - 74LS93 increment (via CKA).
- **Timing**:
  - 1 kHz clock (LM555) ensures synchronized LOAD pulse, preventing multiple counts from button bounce.
  - 74LS93’s ~20 ns ripple delay is negligible for 1 ms clock period.
- **74LS00 Usage**:
  - Uses two NAND gates for \(\overline{\text{CS}}\) and \(\overline{\text{WE}}\) (original circuit).
  - Third gate for counter reset (O_A=B and LOAD).
  - Fourth gate for manual reset (if S15 used) or unused.

### 4. Display
- **4 LEDs (L12–L15)**: Show guess count in binary (e.g., 0011 = 3 guesses).
- **Interpretation**: Players read binary (assume familiarity, common in digital logic contexts). Provide a reference chart (e.g., 0001 = 1, 0010 = 2).
- **Optional 7-Segment Display**:
  - A 74LS47 BCD-to-7-segment decoder and display add 1 IC and complexity. Binary LEDs minimize components.

## Updated Gameplay
The game remains "Guess the Hidden Number," with the counter tracking guesses to enhance the RL analogy (minimize guesses = maximize reward).

1. **Setup**:
   - Store a 4-bit number (0–15) in 2114 RAM at address 00000000 (write mode, S13 = 0, LOAD).
   - Set address switches (S1–S8) to 00000000, read/write switch (S13) to read (1).
   - Reset counter (via S15 or power-on, 74LS93 outputs 0000).

2. **Gameplay**:
   - **Step 1: Input Guess**:
     - Set data switches (S9–S12) to a 4-bit guess (e.g., 1000 = 8).
     - Input LEDs (L1–L4) show the guess.
   - **Step 2: Submit Guess**:
     - Press LOAD (S14). 74LS73 pulse triggers:
       - 2114 RAM read (outputs stored number to 74LS85).
       - 74LS85 comparison (guess vs. stored number).
       - 74LS93 increment (e.g., 0000 → 0001).
     - Feedback LEDs (L9–L11):
       - Red (L9): Too high (O_A>B).
       - Yellow (L10): Too low (O_A<B).
       - Green (L11): Correct (O_A=B).
     - Counter LEDs (L12–L15) show count (e.g., 0001 = 1 guess).
   - **Step 3: Adjust Guess**:
     - Adjust switches based on feedback (e.g., if 8 is too low, try 1100 = 12).
   - **Step 4: Repeat**:
     - Repeat until green LED (L11) lights.
     - On correct guess, O_A=B = 1 resets 74LS93 to 0000.

3. **Winning**:
   - Win when green LED (L11) lights.
   - Note final guess count on L12–L15 before reset (e.g., 0011 = 3 guesses).
   - Optionally, enable 74LS244 to show stored number on L5–L8.

4. **New Game**:
   - Store new number or reuse same.
   - Reset counter via S15 or power cycle.

5. **RL Enhancement**:
   - **Reward**: Minimize guess count (L12–L15). Each guess = negative reward; winning = high cumulative reward.
   - **Learning**: Players refine strategy (e.g., binary search) to reduce guesses, mimicking RL policy optimization.
   - Example: Binary search takes ≤4 guesses for 0–15.

## Example Gameplay with Counter
- **Hidden Number**: 1010 (10).
- **Guess 1**: Set S9–S12 to 1000 (8), press LOAD.
  - 74LS85: 8 < 10 → Yellow LED (L10).
  - 74LS93: 0000 → 0001 (L12 on, 1 guess).
- **Guess 2**: Set 1100 (12), press LOAD.
  - 74LS85: 12 > 10 → Red LED (L9).
  - 74LS93: 0001 → 0010 (L13 on, 2 guesses).
- **Guess 3**: Set 1010 (10), press LOAD.
  - 74LS85: 10 = 10 → Green LED (L11).
  - 74LS93: 0010 → 0011 (L12, L13 on, 3 guesses), then resets to 0000.
- **Result**: Win in 3 guesses, counter shows 0011 briefly, then resets.

## Updated Bill of Materials (BoM)
| Component         | Quantity | Description                                      |
|-------------------|----------|---------------------------------------------|
| **2114**         | 1        | 1K x 4-bit static RAM IC                   |
| **LM555**        | 1        | 555 Timer IC for 1 kHz clock               |
| **74LS73**       | 1        | Dual JK flip-flop for debouncing           |
| **74LS00**       | 1        | Quad 2-input NAND gate for control logic   |
| **74LS00**       | 8        | Quad 2-input NAND gate for control logic (RAM)   |
| **74LS244**      | 1        | Octal buffer/line driver for output LEDs   |
| **74LS85**       | 1        | 4-bit magnitude comparator for game logic   |
| **74LS93**       | 1        | 4-bit binary counter for guess tracking     |
| **SPST Switch**  | 13       | 8 for address (S1–S8), 4 for data (S9–S12), 1 for read/write (S13) |
| **SPST Push-Button** | 2    | LOAD button (S14), Reset button (S15, optional) |
| **LED**          | 15       | 4 for input (L1–L4), 4 for output (L5–L8), 3 for feedback (L9: red, L10: yellow, L11: green), 4 for counter (L12–L15) |
| **Resistor**     | 32       | 14 pull-up (10 kΩ), 14 pull-down (10 kΩ) for switches; 15 current-limiting (330 Ω) for LEDs |
| **Capacitor**    | 7        | 1 for 555 timer (0.1 µF), 6 for decoupling (0.1 µF near each IC) |
| **Power Supply** | 1        | 5V DC regulated supply                     |

**Total ICs (#pins)**: 2114 (18), LM555 (8), 74LS73 (14), 74LS00 (14), 74LS244 (20), 74LS85 (16), 74LS93(14); 7404 (14), 74LS367 (16), 8226 (16); 500-ohm-sip (14), 1k-ohm sip (18), 0.1uF (8).

## Notes
- **IC Count**: 74LS93 adds 1 IC, the simplest 4-bit counter. Cascading flip-flops (e.g., 74LS73) would require more ICs.
- **Display Simplicity**: Binary LEDs (L12–L15) minimize components vs. a 7-segment display (74LS47 + display = +1 IC, +7 resistors).
- **Reset Options**: Automatic reset (O_A=B) suffices; S15 (manual reset) is optional for testing/restarting.
- **RL Fidelity**: Counter quantifies performance (guess count = negative reward), encouraging players to optimize strategy (e.g., binary search).
- **Scalability**: For >15 guesses, cascade two 74LS93s (8-bit), but unnecessary for typical 2–4 guess range.
- **Timing**: 74LS93’s ~20 ns ripple delay is negligible for 1 kHz clock (1 ms) and human interaction.

## Conclusion
The 74LS93 counter, with 4 LEDs and minimal logic, tracks guesses, enhancing the RL analogy by quantifying performance. It integrates seamlessly, using the LOAD pulse to increment and O_A=B to reset. Players minimize guesses, mirroring RL policy optimization.
