# Phase 1 Diagnostic Report: Strict Cascade Integration Plan

## Current single-stage detection surface (before refactor)

### 1) Fallback non-red scan path
- `EatventureBot._scan_and_click_non_red_assets()` used direct `image_matcher.find_template()` on:
  - `upgradeStation`
  - `box1`..`box5`
- This path used a single match-template confidence gate and then clicked if `is_in_forbidden_zone()` passed.

### 2) Upgrade station FSM path
- `EatventureBot.handle_search_upgrade_station()` previously used direct `find_template()` and optional positional refine.
- False positives in blue-ish environment could pass this single-stage check.

### 3) Box opening FSM path
- `EatventureBot.handle_open_boxes()` previously used direct `find_template()` loop across `box1`..`box5`.
- Floor-like brown textures could satisfy correlation and trigger bad clicks.

## FSM and gatekeeper constraints audited
- Routing preserved: no state enum, transition ordering, or route priorities were changed.
- Persistent counters/hold flow preserved:
  - `cycle_counter`, `consecutive_failed_cycles`, `upgrade_found_in_cycle`
  - Hold loop and upgrade mechanics remain in existing handlers.
- Coordinate gatekeeper preserved:
  - Every click in the modified paths still requires `self.mouse_controller.is_in_forbidden_zone(...) == False`.

## New injection points (strict funnel)

### Profile load injection (init-time)
- Added one-time profile build in `EatventureBot.__init__`:
  - `self.asset_profiles = load_asset_profiles(self.templates)`
- Purpose:
  - Store template image (Stage 3)
  - Store canonical contour profile (Stage 2)

### Runtime verifier injection
- Replaced single-stage matching with `verify_asset_strict(...)` in:
  1. `_scan_and_click_non_red_assets()` (fallback path)
  2. `handle_search_upgrade_station()` (station acquisition path)
  3. `handle_open_boxes()` (box opening path)

## Three-stage enforcement behavior
1. Stage 1: strict BGR inRange mask for asset-specific color family.
2. Stage 2: contour extraction + area filtering + `cv2.matchShapes` vs loaded profile contour.
3. Stage 3: template correlation on Stage-2 ROI crop; only returns payload if threshold is met.

No click is emitted unless all three stages pass sequentially.
