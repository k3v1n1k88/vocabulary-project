# Design Guidelines

**Vocabulary Builder Chrome Extension v1.0.5**

---

## Design System Overview

The extension uses **Tailwind CSS** for utility-first styling with a cohesive color palette, spacing scale, and typography system. All UI surfaces (popup, options, sidepanel) share common components and design tokens.

---

## Design Tokens

**Location:** `src/styles/design-tokens.css` (NOT FULLY SCANNED — see open questions)

Expected token categories:
- **Colors:** Primary, secondary, success, warning, error, neutral (with light/dark variants)
- **Spacing:** 4px base unit (0, 1, 2, 3, 4, ... 12 units = 0, 4px, 8px, 12px, ...)
- **Typography:** Font family, sizes, line heights, weights
- **Shadows:** Elevation levels (base, card, modal, tooltip)
- **Radius:** Border radius (small, medium, large, full)
- **Z-index:** Layer stacking (base, dropdown, tooltip, modal, notification)

**Integration:** Tailwind config reads from `design-tokens.css` via CSS variables (e.g., `var(--color-primary)`).

---

## Color Palette

### Core Colors (Assumed, verify in design-tokens.css)

| Role | Value | Usage |
|------|-------|-------|
| **Primary** | #3B82F6 (blue) | Buttons, links, highlights, active states |
| **Secondary** | #10B981 (teal/green) | Success states, checkmarks, positive actions |
| **Warning** | #F59E0B (amber) | Alerts, cautious actions |
| **Error** | #EF4444 (red) | Errors, delete buttons, failed states |
| **Neutral** | #6B7280 (gray) | Text, borders, disabled states |
| **Background** | #FFFFFF (light) / #1F2937 (dark) | Page background |
| **Surface** | #F3F4F6 (light) / #374151 (dark) | Cards, panels, containers |

### Dark Mode

The extension supports light and dark themes via Tailwind `dark:` prefix:

```html
<div class="bg-white dark:bg-gray-900 text-gray-900 dark:text-white">
  <!-- Auto-respects system preference; user can toggle in options -->
</div>
```

---

## Typography

### Font Family

- **Primary:** `-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif` (system font stack)
- **Mono:** `"SF Mono", Monaco, "Cascadia Code", monospace` (for API keys, code snippets)

### Sizes & Weights

| Scale | Size | Weight | Usage |
|-------|------|--------|-------|
| **xs** | 12px | 400 | Labels, helper text, badges |
| **sm** | 14px | 400 | Body text, captions |
| **base** | 16px | 400 | Default body, list items |
| **lg** | 18px | 600 | Subheadings, card titles |
| **xl** | 20px | 700 | Section headers |
| **2xl** | 24px | 700 | Page titles (Dashboard, Settings) |
| **3xl** | 30px | 800 | Modal headers, main title |

### Line Height

- **Tight:** 1.2 (headings)
- **Normal:** 1.5 (body)
- **Relaxed:** 1.75 (long-form text, descriptions)

---

## Spacing Scale

Based on 4px base unit:

| Value | px | Usage |
|-------|----|----|
| xs | 4px | Padding inside buttons, small gaps |
| sm | 8px | Padding around content, small margins |
| md | 12px | Standard card padding, medium gaps |
| lg | 16px | Panel padding, section spacing |
| xl | 24px | Between major sections |
| 2xl | 32px | Page-level spacing |

Example:
```html
<div class="p-lg m-md"> <!-- padding-16px, margin-12px -->
  Content
</div>
```

---

## Component Catalog

### Shared Components

Located in `src/shared/components/`, reused across UI surfaces:

#### **Toggle** (`toggle.tsx`)
Accessible on/off switch for binary settings.

```jsx
<Toggle
  checked={useLLMTranslation}
  onChange={(checked) => updateSetting('useLLMTranslation', checked)}
  label="Enable AI Translation"
/>
```

**Accessible:** `<input type="checkbox">` hidden, styled with CSS; responds to keyboard (Space/Enter).

#### **Lang Dropdown** (`lang-dropdown.tsx`)
Select target language from 12 options.

```jsx
<LangDropdown
  value={targetLanguage}
  onChange={(lang) => updateSetting('targetLanguage', lang)}
/>
```

Options: VI, ZH, JA, KO, ES, FR, DE, PT, RU, TH, ID, AR (with native names and flags emoji).

#### **Stat Item** (`stat-item.tsx`)
Display stat (XP, streak, level) with icon and value.

```jsx
<StatItem icon={XPIcon} label="XP" value={1250} />
```

Used in Dashboard and popup header.

#### **AI Badge** (`ai-badge.tsx`)
Small "AI" indicator when LLM translation used.

```jsx
<AIBadge provider="openai" />
```

Displays provider name in tooltip on hover.

#### **Icons** (`icons.tsx`)
Heroicon wrappers for consistent 24px icon size.

```jsx
<MagnifyingGlassIcon className="w-6 h-6" />
```

Available: Search, Bookmark, Play, Settings, Trash, Copy, Speaker, etc.

#### **Donate Bar** (`donate-bar.tsx`)
Optional CTA for user support (links to donation page).

```jsx
<DonateBar message="Love the extension? Support development!" />
```

#### **Footer Credits** (`footer-credits.tsx`)
Attribution and links to GitHub, privacy policy, etc.

```jsx
<FooterCredits version="1.0.5" />
```

---

## Layout Patterns

### Popup Layout

```
┌─────────────────────────────────────┐
│  Header (logo + stats bar)          │
├─────────────────────────────────────┤
│  TabNav (Dashboard | Study | Vocab) │
├─────────────────────────────────────┤
│  Content Area (variable height)     │
│  - Dashboard: Stats + CTA           │
│  - Study: Flashcard + buttons       │
│  - Vocabulary: Word list + search   │
├─────────────────────────────────────┤
│  Footer (optional: donate bar)      │
└─────────────────────────────────────┘
```

**Dimensions:** 400px × 600px (standard popup size, resizable).

### Options Layout

```
┌─────────────────────────────────────┐
│  Header (logo, version)             │
├─────────────────────────────────────┤
│  Sidebar (nav: Learning, Translation│
│            Highlight, Data, About)  │
│           │ Content Panel          │
│           ├─────────────────────────┤
│           │ Settings controls      │
│           │ (toggles, inputs, etc) │
│           └─────────────────────────┘
└─────────────────────────────────────┘
```

**Dimensions:** 800px × 600px (preferred), responsive to window.

### Sidepanel Layout

```
┌─────────────────────────────────────┐
│  Header (Clear history, nav)        │
├─────────────────────────────────────┤
│  Content Area                       │
│  - Empty state: "No lookups yet"    │
│  - History: Recent lookup cards     │
│  - Error state: Lookup failed       │
├─────────────────────────────────────┤
│  Footer (credits)                   │
└─────────────────────────────────────┘
```

**Dimensions:** 400px (width, fixed) × variable height (expands with content).

---

## Component Styling

### Button Styles

**Primary (Call-to-action):**
```jsx
<button className="px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-500">
  Save Word
</button>
```

**Secondary (Alternative action):**
```jsx
<button className="px-4 py-2 bg-gray-200 text-gray-900 rounded-md hover:bg-gray-300 dark:bg-gray-700 dark:text-white">
  Cancel
</button>
```

**Danger (Delete):**
```jsx
<button className="px-4 py-2 bg-red-600 text-white rounded-md hover:bg-red-700">
  Delete
</button>
```

**Disabled:**
```jsx
<button disabled className="px-4 py-2 bg-gray-400 text-gray-600 cursor-not-allowed opacity-50">
  Disabled
</button>
```

### Card Styles

```jsx
<div className="bg-white dark:bg-gray-800 rounded-lg shadow-md p-4 mb-4">
  <h3 className="text-lg font-semibold text-gray-900 dark:text-white mb-2">
    Card Title
  </h3>
  <p className="text-sm text-gray-600 dark:text-gray-400">
    Card content
  </p>
</div>
```

### Input Styles

```jsx
<input
  type="text"
  className="w-full px-3 py-2 border border-gray-300 dark:border-gray-600 rounded-md bg-white dark:bg-gray-700 text-gray-900 dark:text-white placeholder-gray-500 dark:placeholder-gray-400 focus:outline-none focus:ring-2 focus:ring-blue-500"
  placeholder="Search..."
/>
```

---

## Tooltip Design

### DOM Structure

Tooltips are positioned absolutely near the right-clicked word:

```html
<div id="vocab-tooltip" class="absolute z-50 bg-white dark:bg-gray-800 rounded-lg shadow-lg border border-gray-200 dark:border-gray-700 p-4 max-w-sm">
  <div class="text-lg font-semibold text-gray-900 dark:text-white">
    {{ word }}
  </div>
  <div class="text-sm text-gray-600 dark:text-gray-400 mb-2">
    /{{ phonetic }}/
  </div>
  <!-- Definitions, buttons, etc. -->
</div>
```

### Tooltip Positioning

- **Preferred:** Top-right of selection (if viewport space available)
- **Fallback:** Adjust to keep within viewport (top/bottom, left/right)
- **Saved position:** Floating menu (mouseup) saves position; tooltip reuses for consistency
- **Clamping:** `tooltip-positioning.ts` ensures tooltip stays within window bounds

### Highlight Styling

Highlights use a `<span>` with configurable background color:

```html
<span class="vocab-text-highlight" data-color="#FFF59D" style="background-color: #FFF59D; cursor: pointer;">
  word
</span>
```

**Color options:**
- Yellow: `#FFF59D`
- Blue: `#BBDEFB`
- Green: `#C8E6C9`
- Pink: `#F8BBD0`
- Custom: User picks via color picker in settings

---

## Accessibility (A11y)

### Keyboard Navigation

- **Tab:** Move between interactive elements
- **Enter/Space:** Activate buttons, toggle switches
- **Arrow Keys:** Navigate dropdowns, rating buttons
- **Escape:** Close tooltip, modal, dropdown

### Screen Reader Support

- **ARIA labels:** All buttons, inputs have descriptive labels
- **Semantic HTML:** Use `<button>`, `<input>`, `<select>` (not `<div role="button">`)
- **Focus visible:** `:focus-visible` outline on keyboard nav
- **Alt text:** Icons have titles (hover text)

### Color Contrast

- **Text on background:** WCAG AA minimum 4.5:1 (body), 3:1 (large text)
- **Don't rely on color alone:** Pair color with icon, pattern, or text label

### Touch Targets

- **Minimum size:** 48x48px for buttons/inputs on mobile
- **Spacing:** 8px minimum between interactive elements

---

## Animation & Transitions

### Timing

- **Fast:** 100ms (hover states, toggles)
- **Standard:** 300ms (modal open/close, tab switch)
- **Slow:** 500ms (progress animations)

### Easing

- **ease-in-out:** Default for most animations
- **ease-out:** For dismissals (toast, modal close)

### Flashcard Flip

```css
.flashcard-inner {
  transition: transform 300ms ease-in-out;
  transform-style: preserve-3d;
}
.flashcard-inner.flipped {
  transform: rotateY(180deg);
}
```

### Progress Bar

```css
.progress-bar {
  transition: width 500ms ease-in-out;
}
```

---

## Dark Mode

The extension respects system color scheme preference and allows user override in settings.

Implementation:
- Tailwind `dark:` classes for dark-specific styles
- CSS variable fallbacks for design tokens
- Toggle in options page updates `localStorage` + applies `dark` class to `<html>`

```jsx
// Example in component
<div className="bg-white dark:bg-gray-900 text-gray-900 dark:text-white">
  Content
</div>
```

---

## Responsive Design

The extension UI is optimized for fixed sizes (popup: 400x600, options: 800x600), but uses responsive principles for flexibility:

- **Flex layouts:** Use `flex`, `flex-col`, `justify-between`, `items-center` for adaptive spacing
- **Mobile-first:** Start with compact mobile layout, expand for desktop
- **Overflow handling:** Use `overflow-y-auto` for scrollable panels (sidepanel, vocabulary list)

---

## Icons & Visual Assets

### Icon Style

Use Heroicons (24px) throughout for consistency:

```jsx
import { MagnifyingGlassIcon, BookmarkIcon, ... } from '@heroicons/react/24/outline';

<MagnifyingGlassIcon className="w-6 h-6 text-blue-600" />
```

**Variations:**
- Outline: Default, unfilled
- Solid: Use sparingly (emphasis)

### Logo

Extension logo: 128x128 PNG with rounded corners, suitable for Chrome toolbar (16x16, 32x32, 48x48, 128x128 variants).

### Color Presets in Highlight Settings

Visual color picker with 4 presets + custom option:

```
┌─────────────────────────────────────┐
│ Highlight Color:                    │
│ [Yellow] [Blue] [Green] [Pink]      │
│ [Custom color picker button]        │
│ Example: "The quick brown fox"      │
└─────────────────────────────────────┘
```

---

## Branding & Voice

### Tone

- **Friendly:** "You've studied 5 words today! Great job!"
- **Clear:** Error messages explain issue and suggest action
- **Respectful:** Acknowledge user's effort; avoid condescension

### Microcopy Examples

| Context | Text |
|---------|------|
| Empty state | "No words saved yet. Start by looking up a word on any webpage!" |
| Study complete | "Excellent work! You've reviewed all your words for today." |
| API key error | "Connection failed. Check your API key and try again." |
| First highlight | "Highlighted! This word is saved to your vocabulary." |

---

## Open Questions / TODO

1. **design-tokens.css contract** — Verify exact token definitions, color values, and Tailwind integration
2. **A11y audit** — WCAG 2.1 AA full compliance verification needed (current partial)
3. **Mobile web version** — Extension UI is desktop-focused; responsive design future enhancement
