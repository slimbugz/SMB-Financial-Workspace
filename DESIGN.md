# Design System Specification: The Architectural Ledger

## 1. Overview & Creative North Star
**Creative North Star: "The Sovereign Curator"**

In the world of SMB finance, noise is the enemy. Most financial tools suffer from "Dashboard Fatigue"—a cluttered grid of borders, lines, and shouting data points. This design system rejects the "template" aesthetic in favor of **High-End Editorial Efficiency**. 

We treat financial data as a curated exhibit. By utilizing intentional asymmetry, deep tonal layering, and sophisticated typography scales, we move away from "software" and toward an "architectural experience." The goal is to provide a sense of calm authority, where the UI recedes to let the capital and the decisions take center stage.

---

## 2. Colors: Tonal Depth & The "No-Line" Rule

The palette is anchored in an authoritative navy (`primary: #001e40`) supported by a sophisticated slate-gray spectrum. 

### The "No-Line" Rule
**Explicit Instruction:** Designers are prohibited from using 1px solid borders to define sections or containers. Traditional "boxes" make financial data feel trapped and cluttered. Boundaries must be defined exclusively through:
1.  **Background Color Shifts:** Placing a `surface-container-low` section against a `surface` background.
2.  **Vertical Whitespace:** Using the spacing scale to create "invisible" partitions.
3.  **Tonal Transitions:** Subtle shifts between nested containers.

### Surface Hierarchy & Nesting
Treat the interface as physical layers of stacked, fine paper. 
*   **Base:** `surface` (#f6fafe) for the overall application background.
*   **Primary Containers:** `surface-container-low` (#f0f4f8) for sidebar or navigation regions.
*   **Active Workspaces:** `surface-container-lowest` (#ffffff) for the primary data-entry cards to provide maximum "lift" and focus.

### The "Glass & Gradient" Rule
To ensure the system feels premium and custom, use **Glassmorphism** for floating elements (e.g., Command Palettes, Popovers). Utilize `surface` colors at 80% opacity with a `24px` backdrop blur. 
*   **Signature Textures:** For high-impact CTAs or Hero Cards, apply a linear gradient from `primary` (#001e40) to `primary_container` (#003366) at a 135-degree angle. This adds "visual soul" and depth that a flat fill cannot provide.

---

## 3. Typography: The Editorial Hierarchy

We utilize a dual-typeface system to balance character with legibility.

*   **Display & Headlines (Manrope):** Chosen for its geometric precision and modern "tech-forward" feel. Use `display-lg` through `headline-sm` to anchor page sections. The wider apertures of Manrope convey a sense of openness and transparency.
*   **Body & Labels (Inter):** The workhorse of the system. Inter is used for all financial data (`body-md`) and functional labels (`label-md`). Its tall x-height ensures that even small-scale numbers in data tables remain highly legible.

**Hierarchy Tip:** Always pair a `headline-md` (Manrope) with a `body-sm` (Inter) for descriptions to create a high-contrast, editorial rhythm.

---

## 4. Elevation & Depth: Tonal Layering

We move beyond traditional "Material" shadows. Depth is achieved through the **Layering Principle**.

*   **Tonal Stacking:** Instead of a shadow, place a `surface-container-highest` element inside a `surface-container` to indicate a recessed area (like an inner-shadow effect without the blur).
*   **Ambient Shadows:** If an element must "float" (like a Modal), use a diffused shadow: `y-20, blur-40, spread--5`. The shadow color must be a tinted version of `on-surface` at 6% opacity, never pure black.
*   **The "Ghost Border" Fallback:** If a border is required for accessibility (e.g., in a high-density table), use the `outline_variant` (#c3c6d1) at **20% opacity**. It should be felt, not seen.
*   **Glassmorphism:** Use semi-transparent `surface_container_lowest` for top navigation bars to allow page content to bleed through softly as the user scrolls, maintaining a sense of spatial awareness.

---

## 5. Components

### Action Buttons
*   **Primary:** `primary` fill with `on_primary` text. Use `rounded-md` (0.375rem). Apply the signature 135° gradient on hover to increase "tactile" feel.
*   **Secondary:** `secondary_container` fill. No border.
*   **Tertiary/Ghost:** No fill, `primary` text. Use for low-emphasis actions like "Cancel" or "Export."

### Data Tables & Lists
*   **Rule:** **Forbid Divider Lines.**
*   **Alternative:** Use `surface_container_low` for the header row. Use alternating row shifts (Zebra striping) only if necessary, using `surface_container_lowest` and `surface_container`.
*   **Spacing:** Increase cell padding to `16px` vertically to give financial figures "room to breathe."

### Status Badges (Chips)
*   **Success:** `on_secondary_container` text on a subtle green tint.
*   **Error:** `error` (#ba1a1a) text on `error_container` (#ffdad6).
*   **Highlight (Amber):** `on_tertiary_fixed_variant` text on `tertiary_fixed` (#ffdcc3).
*   **Shape:** Always `rounded-full` (9999px) to contrast against the architectural `0.375rem` of the rest of the UI.

### Input Fields
*   **Style:** Minimalist. `surface_container_highest` background with a `2px` bottom-stroke using `outline_variant`. On focus, the bottom-stroke transitions to `primary`.
*   **Logic:** Move labels to `label-sm` above the field; never use placeholder text as a label.

---

## 6. Do’s and Don’ts

### Do
*   **Use Asymmetry:** Place a large `display-sm` headline on the left with a primary CTA on the far right to create a sophisticated, wide-screen balance.
*   **Embrace Whitespace:** If a screen feels "busy," increase the padding between sections rather than adding a border.
*   **Prioritize Typography:** Let the size and weight of the numbers (Inter) do the talking.

### Don’t
*   **Don't Use Pure Black:** Always use `on_background` (#171c1f) for text to maintain a premium, "ink-on-paper" look.
*   **Don't Use Default Shadows:** Avoid the "fuzzy gray" look. If it's not an Ambient Shadow, don't use it.
*   **Don't Crowd Data:** If a table has more than 8 columns, implement a horizontal scroll within a `surface-container` rather than shrinking the text.
*   **Don't Use Solid Dividers:** Avoid `1px` lines between list items. Use `12px` of vertical space instead.