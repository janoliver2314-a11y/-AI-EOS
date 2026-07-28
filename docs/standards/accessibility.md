# Standard: Accessibility

## Purpose

Define what an interactive control must deliver once it claims a role, so
that what assistive technology is told about the interface matches what the
interface actually does. This standard exists because the failure it guards
against has now occurred twice, in consecutive features, in the same
project.

## Scope

Applies to all user-facing interface code — web, mobile, and terminal UI —
and to any component that sets an ARIA role, state, property, or
accessible name, whether written by hand or inherited from a library's
defaults. Purely internal tooling with no human interface is out of scope.

## Goals

- Ensure a declared role is a description of behavior that exists, never a
  promise of behavior that does not.
- Make the keyboard path a first-class deliverable of interactive work,
  reviewed like any other requirement, rather than a follow-up that never
  arrives.
- Ensure accessibility fixes are verified against the accessibility tree,
  not against the fact that a prop was passed.
- Keep an announced-but-absent affordance out of production, because it is
  worse than a missing one — the key the announcement names usually still
  does something, and that something is often destructive.

## Rules

1. **A declared role obliges its whole contract.** Setting `role="tab"`,
   `role="tablist"`, `role="menu"`, `role="combobox"`, `role="dialog"` or
   any other composite-widget role commits you to that role's full WAI-ARIA
   Authoring Practices contract: the required owned elements, the required
   states and properties (`aria-controls`, `aria-expanded`,
   `aria-selected`, `aria-labelledby`), the keyboard model (arrow keys,
   `Home`/`End`, `Escape`), and focus management (roving tabindex or
   `aria-activedescendant`). **Deliver the contract or do not declare the
   role.** Native elements with plain, honest semantics — a `<button>` that
   says it is a button — are always preferable to a partially implemented
   composite widget.

2. **Keyboard operability is part of "done".** Any control a pointer can
   operate must be reachable and operable by keyboard alone, with a visible
   focus indicator. A task that ships a new interactive control ships its
   keyboard path in the same change, not as a follow-up.

3. **Audit what a library still asserts after you narrow its behavior.**
   Omitting a sensor, handler, or plugin removes a capability but does not
   retract the default ARIA text, `aria-roledescription`, help strings, or
   keyboard hints that describe it. When you deliberately disable part of a
   library's interaction model, override what it announces to match what
   remains.

4. **State conveyed visually is conveyed non-visually too.** Selection,
   validity, and status must be exposed through a programmatic state
   (`aria-pressed`, `aria-selected`, `aria-invalid`, `aria-current`) and
   must never rest on color alone — pair color with weight, underline,
   icon, or text.

5. **Verify against the accessibility tree, not the source.** An
   accessibility fix is unverified until the name, role, or state is
   observed where assistive technology would read it — via an accessibility
   -tree inspection or a test querying by role and accessible name.
   Confirming that a string was passed to the right prop proves only that
   the prop was set; the id it points at may not exist in the rendered DOM.

6. **At least one test asserts on the accessibility output of any new
   interactive control** — query by role and accessible name rather than by
   test id or class, so that the assertion exercises the same channel a
   screen reader uses. An output channel with no assertions drifts from the
   behavior it reports.

## Design Decisions

- **Role completeness is a rule rather than a guideline** because the
  failure is silent to sighted testing, invisible in a green suite, and
  actively misleading to the one group of users it affects. A guideline
  would be re-litigated per component; a rule makes the reviewer's question
  binary: is the contract delivered, yes or no?
- **The standard prefers "drop the role" as an equally valid remedy.** The
  goal is truthful semantics, not maximal ARIA. Reducing a half-declared
  tablist to plain buttons is a legitimate fix and is usually cheaper and
  more robust than completing the widget.
- **Verification is specified as accessibility-tree-level** because a
  previous fix in this family passed correct text to the correct prop while
  the `aria-describedby` it relied on pointed at an id absent from the DOM.
  Both the defect and its fix were therefore unobservable.

## Best Practices

- Reach for a native element first; adopt a composite ARIA role only when
  no native element expresses the interaction.
- When adding a role, open the WAI-ARIA Authoring Practices pattern for it
  and treat its "Keyboard Interaction" and "WAI-ARIA Roles, States, and
  Properties" tables as an acceptance checklist for the change.
- Generate ids for `aria-controls` / `aria-labelledby` with a framework id
  primitive (e.g. React's `useId`) so they are stable and collision-free
  across instances.
- Prefer `aria-pressed` on a real `<button>` over a custom toggle: the
  keyboard and screen-reader paths are then the browser's own.

## Examples

**Compliant** — a toggle that claims only what it delivers:

```tsx
<button type="button" aria-pressed={selected} onClick={toggle}>
  {label}
</button>
```

**Compliant** — a tab that delivers the contract it declares:

```tsx
<div role="tablist" onKeyDown={handleArrowKeys}>
  <button
    role="tab"
    id={`${uid}-tab-${key}`}
    aria-selected={isActive}
    aria-controls={`${uid}-panel`}
    tabIndex={isActive ? 0 : -1}
  >
    {label}
  </button>
</div>
<div role="tabpanel" id={`${uid}-panel`} aria-labelledby={`${uid}-tab-${activeKey}`}>
  …
</div>
```

**Violation** — declares a composite role, delivers none of its contract:

```tsx
<div role="tablist">
  {tabs.map((t) => (
    <button role="tab" aria-selected={t.key === activeKey} onClick={…}>
      {t.label}
    </button>
  ))}
</div>
{/* no tabpanel, no aria-controls, no aria-labelledby, no roving tabindex,
    no arrow-key handler. A screen-reader user hears "tab, 1 of 3",
    presses ArrowRight as the pattern trains them to, and nothing moves. */}
```

## Common Mistakes

- Declaring `role="tab"` / `role="menu"` / `role="combobox"` for the
  announcement while implementing only click handling.
- `role="button"` on a `<div>` with no `keydown` handler and no `tabIndex`.
- Keeping a library's default screen-reader instructions after disabling
  the sensor they describe, so the key named still fires a different,
  often destructive, handler.
- Treating an accessibility gap as a "fast-follow" — the follow-up
  competes with new work and loses.
- Asserting on a test id where a role query would have exercised the
  accessibility channel.
- Conveying selection with background color only.

## Future Improvements

- Add an automated axe/`eslint-plugin-jsx-a11y` gate to CI so
  role-contract violations fail the build rather than relying on review.
- Extend the rule set with a documented keyboard-interaction checklist per
  composite role adopted in this codebase, as those roles are introduced.

## Related Documents

- `docs/standards/testing.md`
- `docs/standards/documentation.md`
- `memory/lessons-learned.md` — LL-0030 (a library's ARIA narration
  outliving the sensor it described) and LL-0033 (why per-task review does
  not catch this class)
