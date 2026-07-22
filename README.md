# Contribution #1: Exclude files from Searches | Mark directories as excluded

**Contribution Number:** 1   
**Student:** Satvik Agarwal  
**Issue:** https://github.com/defold/defold/issues/11267  
**Status:** Phase I Complete  

---

## Why I Chose This Issue

I chose this issue because I have experience using Unity with C# to develop 2D games in my free time, and Defold seems like a healthy lightweight alternative empowering many developers. I have experience with C/C++ from class, but would like to strengthen my skills in a production environment. I hope to learn how to understand complex public repositories in C++ and how to effectively contribute to open source with my agentic programming skills. 

---

## Understanding the Issue

### Problem Description

When game developers search for files or import assets into the engine, the file explorer currently displays all directories, which makes it cumbersome to locate a specific file. Developers want the option to exclude selected directories so they can focus on relevant paths only. This requires adding directory-exclusion support to both file search and general navigation within the file explorer of the editor. 

### Expected Behavior

[What should happen?]

### Current Behavior

[What actually happens?]

### Affected Components

[Which parts of the codebase are involved?]

---

## Reproduction Process

### Environment Setup

Setting up my local development environment was challenging, since there wasn't clear documentation on all the dependencies I needed and what process I needed to follow to start development. The documentation provided information on how to install development tools such as Visual Studio with the C++ extensions, SDK, and Lein for working on the editor. However, it wasn't explicit on how to use these tools to start Defold. One challenge I had was that to start the Defold editor, Lein expected me to build the engine from source. CMake wasn't working properly in the system, so I had to look around online for prebuilt binaries, since my work was primarily on the editor and not the engine. The editor would also only start if these files I found online were put in specific places, so I had to manually make these directories and place files.  

### Steps to Reproduce

1. Launch development build of Defold editor using ```lein run```  in the ```defold/editor/``` directory   
2. Create or open an empty project workspace  
3. Put a directory without assets in the /builtin directory of the project  
4. Press ```CTRL + P``` to open the Open Asset modal  
5. You will find the folder you placed, and have no way of excluding it  

### Reproduction Evidence

- **Commit showing reproduction:** (https://github.com/softwaresat/defold)  
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

There is no functionality to exclude directories from the open assets explorer.   

### Proposed Solution

First create a UI text input for users to specify what directories to exclude. Then implement filtering logic in the same code that loads assets for the Open Assets function, and implement filtering based on the specified paths.   

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Implement a feature to filter out directories listed in a user-defined exclusion input   

**Match:** There is a file called .defignore that is used to exclude directories from the game completely, but it also excludes them from compilation, which is not what we want.   

**Plan:** [Step-by-step implementation plan]
1. Add [:search :exclude-patterns] (array of [pattern enabled] pairs) and [:search :filtering] (boolean) to the prefs schema so they persist in .editor_settings per project.   
2. Extend make-select-list-dialog to accept :extra-initial-state, :extra-event-handler, :refilter-atom, and :header-extra-desc-fn so callers can inject sidebar UI and custom events without touching the dialog internals.   
3. In resource-dialog/make, when the caller supplies :exclude-patterns-atom and related atoms, build a filter button that opens a cljfx popup (checkbox list + add text field), mount it via header-extra-desc-fn, and apply patterns inside the existing filter-fn by reading from the atom on every filter call. Wire a refilter-atom so popup changes immediately re run the filter.    
4. In the add handler, normalise !exclude dirname $\rightarrow$ store as dirname (stripping the ! entirely since all stored patterns are exclusions by definition). Write compile-exclude-pred as a simple substring match predicate over [pattern enabled] pairs with no ! needed.   
5. In query-and-open!, create the atoms from prefs on open, provide save back callbacks, and pass everything through to resource-dialog/make.   
6. Apply the same patterns to Search in Files by accepting exclude-patterns in make-search-data-future and reading them from prefs in search_results_view.   
7. Write unit tests for compile-exclude-pred and integration tests for make-search-data-future with patterns.   

**Implement:** (https://github.com/softwaresat/defold/tree/Issue-11267-Exclude-File-From-Searches)      
**Review:** 
1. Are the new :exclude-patterns and :filtering preferences schema updates clean, backward-compatible, and properly namespaced?   
2. Does the refactoring of make-select-list-dialog keep the core dialog components modular without introducing regression or code smell?   
3. Is the atom lifecycle cleanly managed inside query-and-open! to ensure state resets correctly upon closing and reopening the asset dialog?   
4. Are UI interactions snappy, ensuring that updating checkboxes or adding a string pattern instantly triggers the refilter-atom sequence without UI lag?   

**Evaluate:** 
Manual UI Testing:
1. Open the updated project and access the Open Assets/Resource dialog window. Ensure the new filter button mounts cleanly via header-extra-desc-fn.   
2. Click the filter button to open the ClojureFX popup. Input an exclusion directory pattern (e.g., test input variations like !exclude tests or /tests) and verify that normalization strips the ! and handles strings correctly.   
3. Toggle individual patterns on and off in the checkbox list. Verify that the asset list updates immediately to hide or reveal matching directories.   
4. Trigger a "Search in Files" command using the exact same exclusion patterns to confirm that the future results filtering in search_results_view aligns with the dialog view.   
5. Close and reopen the editor to confirm that the exclusion patterns persist accurately inside .editor_settings.

Automated Testing:
1. Execute the new Clojure unit tests targeting compile-exclude-pred against a variety of matching substring test cases.   
2. Execute the integration test suite targeting make-search-data-future to guarantee multi-threaded search results are filtered robustly under simulated background conditions.   

---

## Testing Strategy

### Unit Tests

- [x] Test case 1: compile-exclude-pred with nil/empty patterns returns nil (no filtering)   
- [x] Test case 2: compile-exclude-pred with all-disabled patterns returns nil   
- [x] Test case 3: Single enabled pattern hides resources whose proj-path contains the substring
- [x] Test case 4: Disabled pattern does not exclude anything
- [x] Test case 5: Multiple enabled patterns exclude all matching resources (any match hides)
- [x] Test case 6: Mix of enabled/disabled: only enabled ones filter
- [x] Test case 7: Pattern with no matches shows everything
- [x] Test case 8: Empty string pattern is ignored
- [x] Test case 9: Single-pattern fast path (first preds) works correctly (regression guard)   
- [x] Test case 10: Three-pattern path uses every-pred correctly
- [x] Test case 11: hidden-path-segment? detects dot-prefixed files and directories
- [x] Test case 12: hidden-path-segment? detects nested hidden paths
- [x] Test case 13: hidden-path-segment? does not treat a dot in the middle of a segment as hidden
- [x] Test case 14: filter-popup badge count reflects active default and custom exclusions
- [x] Test case 15: filter-popup event handler covers adding, toggling, and removing patterns
- [x] Test case 16: filter-popup event handler invokes filtering, library, and hidden-setting callbacks

### Integration Tests

- [x] Integration scenario 1: make-search-data-future with nil patterns returns full file set   
- [x] Integration scenario 2: make-search-data-future with empty patterns returns full file set   
- [x] Integration scenario 3: Enabled pattern removes files whose proj-path contains the substring, leaves others   
- [x] Integration scenario 4: Disabled pattern leaves full file set unchanged   
- [x] Integration scenario 5: Multiple enabled patterns each independently filter matching files   
- [x] Integration scenario 6: Mix of enabled/disabled: only enabled ones reduce the result set   
- [x] Integration scenario 7: Pattern matching nothing returns full file set unchanged   

### Manual Testing

Tested the Open Assets dialog (Ctrl+P / Cmd+P):   
- "Filter" button appears to the right of the search box with correct toolbar styling   
- Clicking it opens a popup anchored below the button   
- "Enable filtering" checkbox at the top toggles filtering globally; state persists across dialog opens   
- Adding !exclude main stores main in the list (no ! displayed)   
- Adding without !exclude prefix is silently ignored   
- Checkbox per item enables/disables individual patterns   
- Clicking the × button removes a pattern; list disappears when empty   
- With items enabled, matching files are absent from the search results immediately   
- Prompt text "Add filter (e.g. !exclude main)" is readable (220px min-width enforced)   
- Patterns and filtering-enabled state survive closing and reopening the dialog (persisted to .editor_settings)   
- Search in Files also respects the saved patterns   
- Search in Files now includes the same Filter button and popup as Open Asset   
- Disabling "Enable filtering" from either dialog disables custom patterns in both dialogs   
- Changes to patterns or filter toggles while Search in Files is open restart the search with the new settings   
- The pinned "Libraries" option replaces the old standalone "Search Inside Libraries" checkbox   
- The pinned "Hidden Files" option excludes dot-prefixed files and directories such as `.vscode/`   
- Filter labels and prompt text are loaded from the English localization resources   

---

## Implementation Notes

### Week [3] Progress

I implemented all of the functionality needed this week. In my experience, the backend logic was the easiest part because most of it was adapted from other parts of the code that also filtered. The hardeset part for me was utilizing the AI to assist in creating the UI component, it had trouble aligning the checkbox and having the dropdown aesthetically pleasing.

### Week [5] Progress

This week I received initial maintainer feedback on my draft pull request and implemented all of the requested changes. I removed the stray unintended file change and refactored the Open Assets filtering state so that the exclusion patterns, popup state, and global filtering flag use the dialog's existing cljfx state map instead of multiple side-channel atoms. The popup now updates and re-filters through the normal cljfx rendering cycle without a separate refilter atom.

I also replaced raw substring matching with a shared whole-path-segment predicate in editor.resource. An exclusion such as `test` now removes resources inside a `/test/` directory without incorrectly removing unrelated files such as `latest_scores.lua` or paths such as `/testing/`. Both Open Asset and Search in Files now use the same shared predicate, and I added regression tests for whole-segment and multi-segment directory matching. I pushed the updated changes to PR #12657 and am waiting for follow-up review from the Defold maintainers.

I also identified Intel LLVM issue #22581 as a possible next contribution. Since the issue contains three deprecated oneAPI extensions and recommends handling each extension in a separate pull request, I asked the maintainers which individual extension would be most suitable for a first-time contributor and whether I could be assigned to it. I am currently waiting for their response before beginning implementation.

### Week [6] Progress

This week I received a second round of detailed maintainer feedback after Joseph cloned my branch and manually tested the feature. I first resolved the new merge conflicts with the latest `dev` branch. I then localized the previously hardcoded UI text for "Filter," "Enable filtering," and the add-pattern prompt by adding the strings to the English editor localization resources.

I fixed the inconsistent behavior between Open Asset and Search in Files by giving Search in Files the same Filter button and popup as Open Asset. Both dialogs now share the same exclusion patterns, global filtering toggle, library setting, and hidden-file setting, so changes made from either dialog affect both search interfaces. Disabling "Enable filtering" now prevents custom exclusion patterns from filtering Search in Files as expected, and changing a filter while Search in Files is open rebuilds and restarts the search with the new settings.

I also unified the existing "Search Inside Libraries" behavior into the shared filter popup as a pinned "Libraries" option and added a pinned "Hidden Files" option for dot-prefixed files and folders such as `.vscode/`. To avoid duplicating the popup UI and event logic between the two dialogs, I extracted a shared `editor.filter-popup` namespace. I added unit tests for hidden-path detection and the shared popup's event handling, including add, toggle, remove, and preference callback behavior. I pushed all of these changes to PR #12657 and am waiting for another round of maintainer review.

### Code Changes

- **Files modified:**    
    - resources/localization/en.editor_localization   
    - resources/search-in-files-dialog.fxml   
    - src/clj/editor/prefs.clj   
    - src/clj/editor/dialogs.clj   
    - src/clj/editor/filter_popup.clj (new)   
    - src/clj/editor/resource_dialog.clj   
    - src/clj/editor/app_view.clj   
    - src/clj/editor/defold_project_search.clj   
    - src/clj/editor/recent_files.clj   
    - src/clj/editor/resource.clj   
    - src/clj/editor/search_results_view.clj   
    - test/editor/resource_dialog_test.clj (new)   
    - test/editor/defold_project_search_test.clj   
    - test/editor/filter_popup_test.clj (new)   
    - test/editor/resource_test.clj   
- **Key commits:**
1. add search.exclude-patterns and search.filtering to project prefs schema   
2. extend make-select-list-dialog with extra-state, extra-event-handler, refilter-atom, and header-extra-desc-fn   
3. add exclude-patterns filter popup to Open Assets dialog with per-pattern enable/disable and global toggle   
4. wire exclude-patterns and filtering-enabled prefs into query-and-open!   
5. apply exclude-patterns filter to Search in Files results   
6. add unit and integration tests for exclude-patterns filtering   
7. fix exclude patterns to match whole path segments, not substrings   
8. add hidden-path-segment? predicate and exclude-hidden? support to search   
9. extract shared editor.filter-popup namespace   
10. add shared exclude-libraries and exclude-hidden prefs and wire them into Open Asset   
11. update Open Asset to use the shared filter popup with Libraries and Hidden Files defaults   
12. add the shared exclusion filter popup to Search in Files and synchronize its state with Open Asset   
13. add unit tests for hidden-path detection and editor.filter-popup event handling   
   
- **Approach decisions:**

Pattern format — Patterns are stored as [pattern-string enabled-boolean] pairs to match the console filter's data model, enabling each entry to be toggled without changing the prefs schema shape. Inputs are trimmed and stored directly as directory or path patterns. Matching is performed against complete `/`-separated path segments so a pattern such as `test` matches `/test/foo.lua` but not `latest_scores.lua` or `/testing/foo.lua`.

State and refilter mechanism — The exclusion patterns, popup state, and global filtering flag are stored as keys in the select-list dialog's existing cljfx state map. The dialog's :filter-key-fn includes the filter term, exclusion patterns, and filtering flag, so popup changes trigger normal re-rendering and re-filtering without a separate side-channel refilter atom.

Shared filter UI — The exclusion popup is implemented in a dedicated `editor.filter-popup` namespace and reused by both Open Asset and Search in Files. This prevents the two interfaces from developing separate behavior and keeps the popup rendering, badge calculation, pattern mutations, and preference callbacks in one place.

Global shared settings — Custom patterns, the global filtering flag, the Libraries option, and the Hidden Files option are stored under shared `[:search ...]` preferences. Both dialogs read and update the same values, so changing a setting in either location immediately affects the other.

Default exclusions — "Libraries" and "Hidden Files" are pinned options rather than editable custom patterns. Libraries reuse the existing library-restriction behavior, while Hidden Files uses `hidden-path-segment?` to detect dot-prefixed path segments such as `.vscode/` without treating ordinary dots in the middle of a filename or directory segment as hidden.

Localization — User-facing labels and prompts are retrieved from `en.editor_localization` rather than being hardcoded in the Clojure UI code.

Popup positioning — Uses window-top-left anchor location (opposite of the console which uses window-bottom-left) because the filter button is in the dialog header (top) rather than a toolbar at the bottom of the screen.

CSS — Popup content explicitly loads editor.css via :stylesheets because popups are separate JavaFX windows that don't inherit the parent scene's stylesheet. The button itself uses hardcoded hex values from the editor palette rather than CSS variable references, since CSS variables are only resolved if the dialog scene has the stylesheet loaded.

---

## Pull Request

**PR Link:** (https://github.com/defold/defold/pull/12657)

**PR Description:** 
Adds per-project editor search exclusion patterns for Open Asset and Search in Files. In the Open Asset dialog, users can open a filter dropdown, add directory or path patterns to exclude, and see each added exclusion as an entry in the dropdown. Each exclusion can be enabled or disabled with a checkbox, removed when no longer needed, and filtering can also be turned on or off globally. The exclusions are saved per project in .editor_settings, so they persist between editor sessions without affecting game.project, builds, bundling, or runtime behavior.

Fixes #11267

This is opened as a draft PR to make the proposed approach easier to review before final polishing.

### Technical Changes

- Adds shared per-project search preferences for custom exclusion patterns, the global custom-pattern toggle, library exclusion, and hidden-file exclusion.
- Uses whole `/`-separated path-segment matching for custom exclusions so a pattern such as `test` does not incorrectly match `latest_scores.lua` or `/testing/`.
- Adds `editor.resource/hidden-path-segment?` to identify dot-prefixed files and folders such as `.vscode/`.
- Extracts the reusable Filter button, popup rendering, badge count, pattern-list mutations, and callbacks into a shared `editor.filter-popup` namespace.
- Adds the same Filter button and popup to both Open Asset and Search in Files.
- Shares all filter state between Open Asset and Search in Files, so changes made in either dialog apply globally.
- Replaces the standalone "Search Inside Libraries" checkbox with a pinned "Libraries" option in the shared popup.
- Adds a pinned "Hidden Files" option for excluding dot-prefixed path segments.
- Rebuilds and restarts Search in Files when exclusion settings change so the visible results immediately reflect the current configuration.
- Localizes the Filter button, section labels, default-option labels, global toggle, and add-pattern prompt in the English editor localization file.
- Adds unit and integration coverage for whole-segment matching, hidden-path detection, popup badge state, popup events, pattern mutations, and preference callbacks.


**Maintainer Feedback:**
- July 14, 2026: Received initial feedback to remove a stray unintended file change, reduce the four atoms previously created in query-and-open!, integrate the filter state into the cljfx rendering mechanism, and replace raw substring matching with actual directory/path-segment matching.
- July 14, 2026: Addressed the feedback by removing the stray change, moving the popup and exclusion state into the dialog's existing state map, eliminating the side-channel refilter atom, extracting the shared editor.resource/compile-exclude-patterns-pred function, applying it to both Open Asset and Search in Files, and adding regression tests for whole and multi-segment paths.
- July 21, 2026: Received a second round of feedback after the maintainer cloned and tested the branch. The requested changes included resolving merge conflicts, localizing the filter UI, making the global filtering toggle apply consistently to Search in Files, adding the Filter popup to Search in Files, and considering shared default options for Libraries and Hidden Files.
- July 21, 2026: Addressed the feedback by resolving the conflicts, adding English localization entries, sharing the same filter state and popup between Open Asset and Search in Files, replacing the standalone library checkbox with a shared Libraries option, adding a Hidden Files option, restarting active file searches when settings change, and adding unit tests for the shared popup and hidden-path logic.

**Status:** Awaiting follow-up review after implementing the second round of feedback

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
