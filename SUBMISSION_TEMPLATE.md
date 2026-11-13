# Submission: [Your Name]

## Time Spent

Total time: **3h 50 minutes** (approximate)

## Ticket Triage

### Tickets I Addressed

List the ticket numbers you worked on, in the order you addressed them:

1. **CFG-142**: Fixed an issue where the displayed price remained at $0.00 despite changes to the product configuration. The root cause was incorrect timestamp comparison in the price calculation hook. I replaced it with request IDs to ensure that only the latest price calculation result is applied.
2. **CFG-148**: Fixed dependent add-on cleanup by updating the selected add-ons immutably when their required product options change. This prevents invalid add-on selections from causing the configurator to crash.
3. **CFG-147**: Updated configuration encoding and decoding to support Unicode characters, including Polish diacritics and emojis, preventing errors when generating share links.
4. **Additional fix — Product preview image URL**: Fixed incorrect query parameter construction in generatePreview() using the native URL API. This prevents configuration parameters from appearing as text inside the product preview image.
5. **CFG-152**: Replaced custom color swatches with native radio inputs, improved focus indicators for quantity controls, and implemented Escape handling with focus restoration for modal dialogs. Verified that native dropdowns already support arrow-key navigation.
6. **CFG-145**: Added a keyboard shortcut for the existing Quick Add functionality, prioritizing TechStyle's explicit demo requirement over the conflicting request to remove the feature in CFG-144.
7. **CFG-143**: Added cleanup for the window resize event listener to prevent it from accumulating across component mounts. Further performance testing is needed to determine whether this resolves the reported long-session slowdown

### Tickets I Deprioritized

List tickets you intentionally skipped and why:

| Ticket  | Reason                                                                                                                                                                                                                       |
| ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CFG-144 | Conflicts with CFG-145. I prioritized TechStyle's explicit requirement to demonstrate Quick Add with a keyboard shortcut. Removing the feature should be reconsidered after clarification with Product and Customer Success. |
| CFG-146 | The incorrect draft timestamp has a lower impact on the upcoming demo than pricing, crashes, and accessibility issues.                                                                                                       |
| CFG-149 | The configurator already provides a basic loading state during price calculation. I prioritized functional reliability over further visual improvements.                                                                     |
| CFG-150 | Mobile color picker alignment was less critical than keyboard accessibility and demo-blocking issues.                                                                                                                        |
| CFG-151 | User-friendly error messages would improve UX, but critical functionality and contractual accessibility requirements took priority.                                                                                          |
| CFG-153 | Comparing saved configurations is a substantial new feature, estimated at 2–3 weeks, making it unrealistic within the four-hour time limit.                                                                                  |
| CFG-155 | Dark mode would improve visual integration with TechStyle's website but was not essential for the demo's core functionality.                                                                                                 |
| CFG-156 | React list key warnings were lower priority than user-facing functional issues and did not justify a separate ticket within the remaining time.                                                                              |
| CFG-157 | Unsaved changes confirmation requires additional UX decisions and implementation effort, so I prioritized existing core features.                                                                                            |
| CFG-158 | The suggested memoization fix was not sufficiently justified. currentConfig was already memoized, and further optimization should be based on React Profiler measurements rather than assumptions.                           |

### Tickets That Need Clarification

List any tickets where you couldn't proceed due to ambiguity:

| Ticket  | Question                                                                                                                                                                                                                                                                                                                                  |
| ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CFG-154 | What is the correct quantity discount threshold and percentage? CONTEXT.md specifies 10% for 50+ units, while CFG-154 references public documentation promising 15% for 50+ units. An internal Slack message also suggests that the discount may intentionally start at 51 units. Which pricing rules should be treated as authoritative? |
| CFG-144 | Should Quick Add be permanently removed or retained for enterprise customers such as TechStyle? CFG-144 requests its removal, while CFG-145 states that the keyboard shortcut is mandatory for the upcoming demo. I implemented CFG-145 for the demo, but the long-term product direction requires confirmation.                          |

### Tickets With a Misleading Priority Label

Our board labels aren't always right. List tickets where you disagreed with the label (e.g. a Low that's actually High, a High that's actually Low, a Medium that's actually Unclear) and explain why. If you didn't disagree with any, say so.

| Ticket  | Label on board | Your read   | Why                                                                                                                                                                                                                              |
| ------- | -------------- | ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CFG-152 | ⚪ Low         | 🔴 Critical | Keyboard accessibility is part of TechStyle's contractual requirements (WCAG 2.1 AA). Shipping a configurator that cannot be fully operated using a keyboard creates compliance risks and could jeopardize the upcoming demo     |
| CFG-145 | 🟠 High        | 🔴 Critical | TechStyle explicitly confirmed that the Quick Add keyboard shortcut is mandatory for the upcoming demo. This makes it a demo requirement rather than a general feature improvement.                                              |
| CFG-158 | 🟠 High        | 🟡 Medium   | The suggested fix is not supported by the current implementation because currentConfig is already memoized. Performance optimization should be guided by profiling results rather than assumptions about unnecessary re-renders. |

### Tickets Where You Stopped to Ask Instead of Implementing

If a ticket was ambiguous enough that the right move was to escalate / ask PM / gather more context rather than ship a fix, list it here. We value this signal — picking one interpretation blindly and shipping it can cost more than stopping to ask.

| Ticket  | Who would you ask     | What you'd ask                                                                                                                                                                                                                                                                                             |
| ------- | --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CFG-154 | PM / Finance          | Should the discount start at 50 or 51 items? And should it be 10% or 15%? The information in CONTEXT.md doesn't match what the ticket says about the public pricing documentation. I didn't change the code because I wasn't sure which rule was correct, and a wrong change could affect customer prices. |
| CFG-144 | PM / Customer Success | Should Quick Add be removed as originally planned, or retained for TechStyle and other enterprise customers? I implemented the shortcut required for the upcoming demo but deferred the removal decision until the conflicting product requirements can be resolved.                                       |

---

## Technical Write-Up

### Critical Issues Found

Describe the most important bugs you identified:

#### Issue 1: Price stuck at $0.00

**Ticket(s):** CFG-142

**What was the bug?**

The price stayed at $0.00 even when the user changed the configuration. The problem was in usePriceCalculation. The hook compared the response timestamp from the API with Date.now(), even though the mocked API used a request counter instead of a real timestamp.

Because of this, valid price responses were ignored.

**How did you find it?**

I checked the price calculation hook and the mocked API to understand why the price wasn't updating. I noticed that the values being compared were completely different types of numbers, which explained why the price never changed.

**How did you fix it?**

I replaced the timestamp comparison with local request IDs. Each new price calculation gets an ID, and only the latest request is allowed to update the price.

I also made sure older requests could not incorrectly change the loading state.

**Why this approach?**

I wanted to fix the problem without changing the mocked API. Using request IDs also helps prevent older responses from overwriting newer prices when users change options quickly.

---

#### Issue 2: Crash when disabling packaging

**Ticket(s):** CFG-148

**What was the bug?**

The application crashed when the user disabled packaging while Gift Wrap was selected.

Gift Wrap depends on packaging, but the code responsible for removing dependent add-ons only handled select options, not toggles. It also modified the selected add-ons array directly.

This could leave an unavailable add-on in the selected configuration, causing the summary to access a missing add-on and crash.

**How did you find it?**

I followed the reproduction steps from the ticket and checked how the configurator handled dependencies between options and add-ons.

I found that changing a toggle didn't correctly remove add-ons that depended on it.

**How did you fix it?**

I updated handleOptionChange to check add-on dependencies whenever an option changes, including toggles.

I also replaced the direct array modification with an immutable update using filter().

**Why this approach?**

This fixes the dependency handling instead of just preventing the crash in the summary. It also follows React's recommended approach of updating state without mutating existing arrays.

---

#### Issue 3: Shared links failing with special characters

**Ticket(s):** CFG-147

**What was the bug?**

Sharing a configuration could fail when its name contained Polish characters or emojis.

The encoding function passed the JSON string directly to btoa(), which cannot handle Unicode characters outside its supported byte range.

**How did you find it?**

At first, I couldn't reproduce the issue using a normal configuration. I eventually found that entering a draft name with special characters and closing the modal without saving left the name in the configuration state.

When I opened the Share modal, the encoding function failed and the app crashed.

**How did you fix it?**

I updated the encoding function to convert the configuration data into UTF-8 bytes before encoding it with Base64.

I also updated the decoding function to correctly restore the UTF-8 text.

**Why this approach?**

I considered clearing the draft name when closing the modal, but that would only hide the issue in one specific situation.

Fixing the encoding logic directly allows configuration names containing Unicode characters to be processed correctly without removing user data.

---

### Other Changes Made

Brief description of any other modifications:

- CFG-152 - Keyboard accessibility: Replaced custom color swatches with native radio inputs, added visible focus indicators, and improved Escape handling and focus restoration for modals.

- CFG-145 - Quick Add shortcut: Added a keyboard shortcut for Quick Add, as requested by TechStyle for the demo.

- CFG-143 - Resize listener cleanup: Added removeEventListener to clean up the resize listener when the component unmounts. Further testing would be needed to confirm whether this solves the reported long-session slowdown.

- Additional fix - Product preview: Fixed the preview image URL by using the native URL API to add query parameters correctly. This was an issue I noticed during manual testing and was not part of the original backlog.

---

## Code Quality Notes

### Things I Noticed But Didn't Fix

List any issues you noticed but intentionally left:

| Issue                                       | Why I Left It                                                                                                                                                                                                                                                      |
| ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| ESLint warnings in useEffect hooks          | Some effects update React state directly, triggering the react-hooks/set-state-in-effect rule. I used targeted ESLint disable comments where needed to keep the fixes focused. With more time, I would refactor these effects instead of suppressing the warnings. |
| Further performance investigation (CFG-143) | I added cleanup for the resize event listener, but I didn't fully investigate the slowdown reported after 15–20 minutes. More testing and profiling would be needed to confirm the root cause.                                                                     |

### Potential Improvements for the Future

If you had more time, what would you improve?

1. Split ProductConfigurator into smaller components.
2. Add automated tests for critical functionality.
3. Improve the handling of asynchronous operations.

---

## Questions for the Team

Questions you would ask in a real scenario:

1. What are the correct quantity discount rules? Should the discount start at 50 or 51 items, and should it be 10% or 15%?
2. Should Quick Add be removed as requested in CFG-144, or kept because TechStyle requires it?
3. Are there any automated tests or performance benchmarks we should use to verify changes before merging them?
4. Should dependent add-ons be automatically removed when their required option is disabled, or should the user be asked to confirm?
5. Should user be able to use Quick Add when a modal is opened?

---

## Assumptions Made

List any assumptions you made to proceed:

1. I assumed that only the latest price calculation result should be displayed when users change options quickly.
2. I assumed that add-ons should be automatically removed when their required options are disabled, without asking the user for confirmation.

3. I assumed that Quick Add should remain available for the upcoming demo despite the conflicting request to remove it.

4. I assumed that improving keyboard accessibility was more important than its Low priority suggested because of TechStyle's requirements.

---

## Self-Assessment

### What went well?

I managed to fix the most critical bugs within the time limit. I also found and fixed additional issues. I also think I made good decisions regarding task prioritization.

### What was challenging?

Working with an unfamiliar project under the time pressure. Some bugs took longer to reproduce then I expected, It was also difficult what to focus on first when the conflicting requirments appeared

### What would you do differently with more time?

I would spend more time testing my changes and add unit tests as well as E2E tests for the user flow and spend more time investigating the performance issues

---

## AI Assistance Notes (optional)

I used ChatGPT and to help me search the codebase, debug issues and generate code for some simpler fixes. I reviewed the suggested code, tested it manually and made changes when needed. I didn't always agree with the suggested solutions, so I sometimes modified or refactored the generated code to better fit the project.

---

## Additional Notes

Anything else you want us to know:

More details about each PR can be found in its description on GitHub. All PR descriptions follow the provided template.
