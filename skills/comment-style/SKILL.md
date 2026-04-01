---
name: comment-style
description: "Write, add, or revise code comments to match the user's preferred style: brief, structured, language-appropriate documentation comments for functions plus high-level block comments that explain what a section of code does. Use when editing, reviewing, or adding comments in C, C++, Python, R, or similar languages, especially when the user wants Doxygen-style or standard doc comments, concise parameter/return descriptions, file headers, logical subsections inside functions, and top-of-block comments instead of dense inline commentary, including for code that currently has little or no commenting."
---

# Comment Style

Write comments in the user's preferred style. This includes both revising existing comments and adding new comments to code that currently has none.

## Core Style

- Keep comments brief and functional.
- Prefer comments above a line or block, not trailing comments on the right.
- Use trailing comments only when defining what a variable or constant represents.
- Comment groups of lines at a high level so a reader can skim the comments and understand the flow.
- Avoid narrating obvious syntax or repeating what the code already says clearly.
- Add a little more explanation only when the reasoning behind an implementation choice is not obvious.

## Function Comments

Document every function with the standard comment style for that language.

- In C/C++, prefer Doxygen-style block comments, with at least @brief, @param and @return.
- In Python, prefer concise docstrings in the style already used by the file or project. Do not stop at a one-line summary when parameters or return values should be documented too.
- In R, prefer the project's normal roxygen-style or script-level comment convention when appropriate.

Keep function comments short and practical. Include:

- what the function does
- parameters
- return value when relevant

When exceptions, side effects, or important assumptions matter, mention them briefly.

## File Headers

When adding or revising a file header, keep it short. Include these when they fit the language and file type:

- what the file/script does
- author
- date

Do not write long prose headers.

## Section Comments And Subsections

Subsection functions aggressively enough that a reader can skim them.

When writing or revising a function:

- break the body into logical groups of related lines
- insert blank lines between those groups when needed
- put a short comment directly above each meaningful group
- keep section comments focused on intent, not mechanics
- let the comment describe what the next contiguous block does as a whole
- do not place a comment high above several unrelated guard clauses or decisions

Do not turn every two lines into a subsection. A subsection should reflect a real stage of the function.

Short guard clauses are an exception when they represent meaningful logic, especially early returns that skip work for different reasons. In those cases, add a short comment above each distinct guard block.

Good section comments:

- describe the overall purpose of the next group of lines
- help a reader follow the flow of the function
- stay at the level of intent, not micro-operations

Avoid stacking too many comments in a row. A section comment should earn its space.

## Style Pattern To Follow

Use patterns like these.

### C/C++ function comment

```cpp
/**
 * @brief Pack status, speeds, initial pause, and battery into 32 bits.
 * @param header Output array for the packed controller header.
 */
void encodeControllerHeader(uint8_t (&header)[4]) {
  // keep the status code in the low 3 bits
  uint32_t status = 4;
  if((serialFrame.statusCode >= 0) && (serialFrame.statusCode <= 7)) {
    status = static_cast<uint8_t>(serialFrame.statusCode);
  }

  // clamp fields to their packed bit ranges
  uint32_t configuredSpeed = static_cast<uint32_t>(constrain(serialFrame.configuredSpeed, 0L, SPEED_COMMAND_MAX));
  uint32_t actualSpeed = static_cast<uint32_t>(constrain(serialFrame.actualSpeed, 0L, SPEED_COMMAND_MAX));
  uint32_t initialPause = static_cast<uint32_t>(constrain(serialFrame.initialPause, PAUSE_COMMAND_MIN, PAUSE_COMMAND_MAX));
  uint32_t batteryMv = static_cast<uint32_t>(constrain(lroundf(battery * 1000.0f), 2500L, 4425L));
  uint32_t batteryBits = static_cast<uint32_t>(lroundf(((batteryMv - 2500.0f) / (4425.0f - 2500.0f)) * 31.0f));
  uint32_t packed = status & 0x07;

  // pack the full 32-bit controller header contiguously
  packed |= (configuredSpeed & 0x01FF) << 3;
  packed |= (actualSpeed & 0x01FF) << 12;
  packed |= (initialPause & 0x3F) << 21;
  packed |= (batteryBits & 0x1F) << 27;

  // split the packed header into 4 little-endian bytes
  header[0] = static_cast<uint8_t>(packed & 0xFF);
  header[1] = static_cast<uint8_t>((packed >> 8) & 0xFF);
  header[2] = static_cast<uint8_t>((packed >> 16) & 0xFF);
  header[3] = static_cast<uint8_t>((packed >> 24) & 0xFF);
}
```

This is a strong example of the preferred style:

- brief Doxygen header
- logical blank-line-separated subsections
- one short comment per subsection
- comments explain what each stage does overall

### Another good subsectioning pattern

```cpp
uint32_t computeAutonomousSleepMs(uint32_t now) {
  uint32_t nextWakeMs = UINT32_MAX;
  mustReadControllerNext = false;

  if(!serialFrame.valid || (serialFrame.lastPacketMs == 0U)) return 0U;

  // dont schedule a sleep if tasks are pending
  if(memoryTxPending) return 0U;

  if(gpsWindowPending) {
    int32_t untilGpsRead = static_cast<int32_t>((gpsWindowStartMs + rcd.getGpsWindow() * 1000UL) - now);
    if(untilGpsRead <= 0) return 0U;
    nextWakeMs = min(nextWakeMs, static_cast<uint32_t>(untilGpsRead));
  }

  // check if a command is pending verification
  if(pendingCommand.active) {
    int32_t remaining = static_cast<int32_t>(pendingCommand.deadlineMs - now);
    if(remaining <= 0) return 0U;
    nextWakeMs = min(nextWakeMs, static_cast<uint32_t>(remaining));
  }

  // calculate time until next preemptive wake
  int32_t untilExpectedPacket = static_cast<int32_t>(
    (serialFrame.lastPacketMs + CONTROLLER_PACKET_PERIOD) - CONTROLLER_WAKE_MARGIN - now
  );

  // if we already missed the window, force a read ASAP
  if(untilExpectedPacket <= 0) {
    mustReadControllerNext = true;
    return 0U;
  }

  // schedule the next preemptive wake
  uint32_t packetWakeMs = static_cast<uint32_t>(untilExpectedPacket);
  if(packetWakeMs < nextWakeMs) {
    nextWakeMs = packetWakeMs;
    mustReadControllerNext = true;
  }

  return nextWakeMs;
}
```

This is the pattern to favor when writing comments for control-flow-heavy code.

### Operations should be commented where they happen

```python
def update_running_state(state: ControllerState, now: float) -> None:
    """Advance the controller state after the initial pause expires.

    Args:
        state: Controller values to update.
        now: Current monotonic timestamp.
    """

    # Skip all work if the controller is not running.
    if not state.running:
        return

    # Skip if the controller is not in initial pause mode (0) or there is no deadline to wait for.
    if state.status_code != 0 or state.pause_deadline is None:
        return

    # Move from status 0 to status 1 once the pause deadline has passed.
    if now >= state.pause_deadline:
        state.status_code = 1
        state.pause_deadline = None
        logging.info("initial pause finished, status changed to 1")
```

In this style, each comment should sit directly above the exact branch or block it describes. If two nearby `if` statements serve different purposes, give them separate comments instead of one comment for the whole area.

### Python docstrings should still document parameters

```python
def encode_location(latitude, longitude):
    """Encode latitude and longitude into compact fields.

    Args:
        latitude: Latitude in decimal degrees.
        longitude: Longitude in decimal degrees.

    Returns:
        Encoded latitude and longitude values.
    """
    encoded_lat = int((latitude + 90.0) / (180.0 / (2 ** 24)))
    encoded_lon = int((longitude + 180.0) / (360.0 / (2 ** 24)))

    return encoded_lat, encoded_lon
```

Do not leave Python docstrings at only a summary line when the function has meaningful inputs or outputs. Keep them brief, but still document arguments and returns.

### More detailed explanation only when justified

```cpp
char *endPtr = nullptr;
long parsedStatus = strtol(tokens[2], &endPtr, 10);
// strtol leaves endPtr at the first character it did not consume.
// If endPtr still equals the input pointer, no digits were parsed at all.
// If *endPtr is not '\0', the token had extra non-numeric characters.
if((endPtr != tokens[2]) && (*endPtr == '\0')) {
  serialFrame.statusCode = static_cast<int>(parsedStatus);
}
else {
  serialFrame.statusCode = -1;
}
```

Use this longer style only when the reasoning is not obvious from the code itself.

## Adding Comments To Bare Code

When code has few comments or no comments at all:

- add function documentation first
- create logical subsections with blank lines if they do not already exist
- add the high-level section comments needed to make the flow easy to skim
- do not try to comment every line
- prefer a small number of well-placed comments over dense annotation
- comment the intent of a block, not the mechanics of each statement

When inserting comments into an uncommented function, part of the job is to improve structure:

- regroup related lines with spacing if the function is visually dense
- place the comment directly above the block it describes
- keep the function easy to scan from comment to comment

## Out Of Scope

This skill must not change code behavior. It only improves commenting style and code readability through comments and whitespace grouping.

Do not:

- change logic
- reorder lines in a way that changes behavior
- alter conditions, expressions, or data flow
- rename symbols as part of comment work
- make functional refactors under the excuse of improving comments

It is acceptable to insert/remove blank lines to create logical subsections, but only when that does not change behavior.

## Editing Rules

When revising comments:

- preserve existing useful comments that already match the style
- tighten long comments instead of expanding them
- remove comments that only restate obvious code
- normalize mixed comment styles toward one consistent style within the file
- keep terminology consistent with the codebase

When unsure, choose the shorter comment, but do not omit the subsection comments that help a reader follow the function.
