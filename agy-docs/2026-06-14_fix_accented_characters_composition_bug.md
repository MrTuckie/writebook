# Summary of Changes: Fixing Accented Characters (Dead Keys) Bug

## The Bug
When using international keyboard layouts (like US-International or Portuguese ABNT2), accented characters are produced using "dead key" sequences (e.g., pressing `'` then `a` to produce `á`). 
In WriteBook's Markdown editor ("House"), the editor incorrectly interrupted this native browser feature. Instead of replacing the dead key sequence with the final accented character, it appended both characters, producing artifacts like `'á`.

## Root Cause
The editor (bundled in `vendor/javascript/house.min.js`) processes the `beforeinput` event and aggressively calls `t.preventDefault()`. This cancels the browser's native composition process. It then immediately updates its internal state and re-renders the editor's `innerHTML`. This brute-force DOM recreation destroyed the browser's native composition text nodes, causing it to lose context and fail to replace the initial dead key character.

## The Fix
We modified the editor component to correctly hook into the browser's native composition lifecycle:

1. **Captured State on Start**: Added a listener for `compositionstart` (`#CStart`) to capture and store the user's initial cursor selection.
2. **Deferred State Updates**: Modified the `beforeinput` handler (`#D`) to ignore events when `t.isComposing` is `true` or `t.inputType === "insertCompositionText"`. This prevents `preventDefault()` from running and allows the browser to natively manage the text composition inside the DOM without interference.
3. **Re-synced on Completion**: Added a listener for `compositionend` (`#CEnd`). When the browser natively completes the text composition, it provides the finalized string (e.g., `á`) in `t.data`. We then replace the text at the originally captured cursor range with this string and allow the editor to safely perform its re-render and selection synchronization.

These changes were made to `vendor/javascript/house.min.js` and were subsequently checked and minified.

## Local Docker deployment
To test the fix, the Docker container was built locally (`writebook-local`) and run mapping port 80 to host port 8088. A missing `SECRET_KEY_BASE` environment variable was also patched dynamically to allow the Rails app to boot correctly in the containerized production environment.
