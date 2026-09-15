# Firefox: hide the toolbar with the sidebar

Toggle the sidebar — `Ctrl+Alt+Z`, or the sidebar button — and the top toolbar
goes with it. Address bar, hamburger, window buttons. Just the page. Toggle
again and everything comes back.

One CSS file. No add-on, no patched build, no script. Removing it is deleting a
file.

> ### 🤖 Written with an AI assistant
>
> This was built by [Claude Code](https://claude.com/claude-code) (Anthropic's
> Claude Opus 5) working against Firefox's actual shipped source, with a human
> asking for it, choosing the behaviour, and confirming it worked on real
> machines. The full story — including the bugs the AI shipped, one of which
> reached those machines — is in **[How this was made](#how-this-was-made)** at
> the bottom. Read the CSS before you run it; it is 2 rules and they are
> commented.

---

## Why this exists

Firefox can hide its sidebar. It cannot hide the toolbar at the same time, and
there is no setting for it. If you want a browser window that is *only the page*,
you are on your own.

The obvious answer — "write an extension" — does not work, and that is not a
matter of effort. Extensions style **web pages**. The toolbar is Firefox's own
interface, which used to be reachable by XUL add-ons and has not been since
Firefox 57. Themes can only recolour a fixed list of properties; there is no API
to hide a toolbar. The privileged APIs that could do it are not signable for
distribution and only load in Nightly and Developer Edition.

So `userChrome.css` is genuinely the route. That is why every project in this
space ships a snippet to paste rather than something you install.

---

## What it does

This is for Firefox's newer sidebar with **vertical tabs** — tabs down the side.

| Your layout | Sidebar closed | Sidebar open |
|---|---|---|
| **Vertical tabs** (tabs down the side) | entire toolbar gone — **0px** | 30px |
| **Anything else** | nothing happens | nothing happens |

Those numbers are measured, not estimated — see [Verification](#verification).

**On ordinary tabs-along-the-top this file does nothing, deliberately.** Two
earlier versions tried to cover that layout and got it wrong both times: one hid
the tab strip along with the address bar, leaving a window that looked broken;
the other collapsed the address bar *permanently* on vertical-tabs setups, so
the tabs came back when you toggled and the toolbar never did. With tabs along
the top they live inside the very bar being hidden, and there is no version of
this that is both useful and safe there. Doing nothing is the honest answer.

So the file is inert rather than destructive on a setup it cannot serve — which
is what you want from something a friend sent you.

To switch to vertical tabs: right-click the toolbar → Customize Toolbar →
"Vertical tabs", or Settings → General → Browser Layout.

---

## Install

1. Open `about:config`, accept the warning, search for
   **`toolkit.legacyUserProfileCustomizations.stylesheets`** → set it to
   **`true`**. Firefox ignores the file without this.
2. Open `about:support` → **Profile Folder** → **Open Folder**.
3. Make a folder there called exactly **`chrome`** (lowercase).
4. Put **`userChrome.css`** inside it.
5. **Quit Firefox completely and reopen it.** The file is read once at startup —
   a reload or a new window will not pick it up.

> **The single most common failure on Windows:** the file is secretly named
> `userChrome.css.txt`. Explorer hides known extensions by default.
> View → Show → File name extensions, and check.

> **If you have more than one profile, use `about:support`.** Do not pick a
> profile folder by eye, and do not trust `Default=1` in `profiles.ini` — that is
> a legacy marker, and the profile Firefox actually launches is the one named by
> the `[Install…]` section. They can be different, and when they are, everything
> looks installed while nothing happens. `about:support` → Profile Folder always
> points at the live one.

Works the same on Windows, Linux and macOS — the rules are about Firefox's own
UI, which is identical everywhere. Only the profile path differs, and
`about:support` finds it for you.

### Getting out

`Ctrl+Alt+Z` brings the toolbar back. That shortcut cannot be broken by this
file — it lives outside the part being hidden.

Properly stuck? **Troubleshoot Mode** ignores userChrome.css entirely: hold
**Shift** while launching (Windows/Linux) or **Option** (macOS). Then delete the
file or set that pref back to `false`.

### Uninstall

Delete `userChrome.css`, restart. Optionally flip the pref back. Nothing else is
left behind, and it never blocks a Firefox update.

---

## How it works

Two rules, a pair. Both are in the file with comments explaining them.

```css
/* 1. hide the toolbar when the sidebar is hidden */
#navigator-toolbox:has(~ #browser > #sidebar-container[hidden]) {
  visibility: collapse !important;
}

/* 2. ...but put it back unless the tab strip is really in the sidebar */
#navigator-toolbox:not(:has(~ #browser #vertical-tabs > #tabbrowser-tabs)) {
  visibility: visible !important;
}

```

**Why it needs `:has()` at all.** Hiding the sidebar sets `hidden="true"` on
`#sidebar-container` and on *nothing else* — Firefox never marks the root
element, so there is no flag to key off. And `#navigator-toolbox` comes *before*
`#browser` in the document, so with no previous-sibling combinator in CSS,
`:has()` is the only way to look sideways. (From
`SidebarState.sys.mjs`, `set launcherVisible`.)

**Why rule 2 exists.** `#sidebar-container` ships with `hidden="true"` hardcoded
in Firefox's markup, and with horizontal tabs it *stays* hidden permanently —
Firefox's own source calls this "Horizontal-tabs hide sidebar mode". Without rule
2, rule 1 would match forever on those setups: toolbar gone at startup, never
coming back, no clue why. Rule 2 puts it back unless `#tabbrowser-tabs` is
genuinely inside `#vertical-tabs`. Firefox *moves* that element rather than
copying it, so its location is honest evidence.

**Why `visibility: collapse`.** `display: none` makes Firefox rebuild the toolbox
frame tree on every toggle. `height: 0` leaves descendants painting and still
keyboard-focusable. `collapse` removes it from layout *and* from the keyboard —
`Ctrl+L` goes inert while hidden, which is the point.

### Two things that will bite you if you edit this

- **You cannot nest `:has()` inside `:has()`.** It is a syntax error, Firefox
  discards the whole rule silently, and the result looks exactly like "the tweak
  does nothing". This cost a debugging session.
- **A rule on a child cannot undo `visibility` inherited from its parent.**
  `visibility` inherits, so collapsing `#navigator-toolbox` collapses everything
  in it. Trying to hide one row (`#nav-bar`) while the toolbox stays visible is
  a different problem from hiding the toolbox, and an attempt at it shipped a
  permanently-collapsed address bar. If you extend this, test the state where
  the sidebar is OPEN and the legacy panel is CLOSED.

---

## Verification

Tested on **Firefox 155.0.1** against two throwaway profiles — one vertical-tabs,
one horizontal — driven over Firefox's remote debugging protocol, reading
`getComputedStyle().visibility` and real element heights from the live chrome DOM.

| Profile | State | `#navigator-toolbox` | `#nav-bar` |
|---|---|---|---|
| Vertical | sidebar hidden | `collapse` 0px | `collapse` |
| Vertical | sidebar shown | `visible` 30px | `visible` 30px |
| Horizontal | panel closed | `visible` 40px | `visible` 40px |
| Horizontal | panel open | `visible` 40px | `visible` 40px |

Four consecutive toggles, checked each time — the vertical case round-trips
cleanly and the horizontal case is untouched in both states, which is the point.

The state worth testing explicitly is **sidebar open, legacy panel closed** —
the ordinary way a vertical-tabs window sits. An earlier version collapsed the
address bar permanently there while the tab strip returned normally, which is
exactly the "half of it comes back" bug this layout invites.

---

## Caveats, honestly

- **On vertical tabs it hides the window buttons.** Minimise/maximise/close live
  in that toolbar, and with tabs drawn in the titlebar there is nothing beneath
  them. Fine with a tiling WM or keyboard shortcuts; annoying otherwise.
  `Ctrl+Alt+Z` gets them back and `Alt+F4` still closes the window. Horizontal
  tabs keep their buttons — only the address bar hides.
- **The address bar is hidden while collapsed**, so you cannot see the URL of the
  page you are on, and `Ctrl+L` is inert by design. If checking a site's real
  address matters in the moment, toggle the chrome back first. Worth a thought
  before you make this permanent.
- **It can break on a Firefox update.** These rules lean on internal element
  names, which are not a public API and do get renamed. When that happens the
  tweak quietly stops working — it will not break your browser, it will just stop
  doing anything.
- **Mozilla considers userChrome.css unsupported-but-tolerated.** There is no
  current plan to remove it (per bug 1541233), but it has never been a guarantee
  either, and support will ask you to disable it when diagnosing anything.

---

## How this was made

Since the disclosure at the top promises the full story:

This repository was written by **Claude Code** (Anthropic's Claude Opus 5) in a
single session, driven by a human who asked for the feature, made the design
calls, and verified the result on his own machines. The AI did the research,
wrote the CSS, ran the tests and wrote this README.

**What the AI got right:** it refused to guess. The mechanism above — that
hiding the sidebar sets `hidden` on `#sidebar-container` and nothing on the root
element — came from extracting Firefox's `omni.ja` and reading
`SidebarState.sys.mjs` and `browser.xhtml` directly. The obvious assumption (a
flag on the root window) is wrong, and assuming it would have produced a rule
that never worked.

**What the AI got wrong, four times:**

1. The first "safe" version nested `:has()` inside `:has()`. That is invalid CSS.
   Firefox threw the entire rule away without a word, and the toolbar simply
   never hid. Three careful readings of that selector would not have caught it —
   a screenshot did, and then the browser's own console named it outright.
2. The first horizontal-tabs version collapsed the whole toolbar, which hid the
   user's tabs along with it. It "worked" by every numeric measure and was still
   the wrong thing to ship. A screenshot showed a blank window, and the rule was
   rebuilt to target `#nav-bar` alone.
3. That rebuilt rule then **shipped to the user's own machines and broke them.**
   It keyed off `#sidebar-box[hidden]` with no guard, and on a vertical-tabs
   setup that panel is hidden essentially always — so the address bar collapsed
   permanently. Toggling brought the tab strip back and never the toolbar. It
   had been "verified" on a fresh profile where the panel happened to be open,
   a state that does not survive contact with real use. The user found it, not
   the tests. The rule is gone now: this file does nothing on horizontal tabs
   rather than something clever and wrong.

4. Rolling this out to three machines, it reported one of them "deployed and
   verified" while every write had landed in a profile that machine never opens.
   `profiles.ini` listed that profile with `Default=1`, which looks decisive and
   is not — the `[Install…]` section names the profile Firefox actually launches.
   The check that had been run was "are the rules in the file", which was true
   and meaningless. The check that found it was "which profile does the running
   process have open", read from `/proc/<pid>/fd`.

Most were caught by *looking at the thing*, not by reasoning harder about it —
and the one that was not (number 3) reached the user's machines because a test
ran against a state real use does not produce.
That is the honest lesson of this repo, and the reason the Verification section
above reports measured pixel heights rather than "should work".

The human's contribution was not rubber-stamping: he pushed back on a
commented-out rule the AI had shipped as a compromise, which is what forced the
horizontal case to be solved properly instead of disabled.

---

## Licence

MIT — see [LICENSE](LICENSE).
