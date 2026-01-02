# QLMarkdown Improvement Plan

## Overview
Three main objectives:
1. **Remove deprecation warning** - Modernize legacy WebView code
2. **Remove all coffee reminders** - Eliminate donation prompts
3. **Optimize Quick Look preview speed** - Improve rendering performance

---

## 1. Remove Deprecation Warning

### Issue
The code uses deprecated `WebView` class (deprecated since macOS 10.14) marked with `@available(macOS, deprecated: 10.14)`:
- [PreviewViewController.swift:25-35](QLExtension/PreviewViewController.swift#L25-L35) - `MyWebView` class
- [PreviewViewController.swift:254-267](QLExtension/PreviewViewController.swift#L254-L267) - `WebFrameLoadDelegate` extension

### Changes Required
| File | Action |
|------|--------|
| `QLExtension/PreviewViewController.swift` | Remove `MyWebView` class (lines 25-35), remove `useLegacyPreview` conditional logic (lines 85-101), remove `legacyWebView` property (line 39), remove `WebFrameLoadDelegate` extension (lines 254-267) |
| `QLMarkdown/Settings.swift` | Remove `useLegacyPreview` property (line 158) |

### Risk: Low
- The legacy WebView path is only used when `useLegacyPreview = true`
- Default is `false`, so most users already use WKWebView
- macOS 11+ is the primary target (per `QLIsDataBasedPreview`)

---

## 2. Remove All Coffee Reminders

### Locations to Modify

| File | Location | Content to Remove |
|------|----------|-------------------|
| `QLExtension/PreviewViewController.swift` | Lines 219-232 | Nag screen logic (every 100 renders) |
| `qlmarkdown_cli/main.swift` | Lines 339-346 | CLI coffee message |
| `QLMarkdown/AppDelegate.swift` | Lines 148-151 | `buyMeACoffee()` method |
| `QLMarkdown/Settings.swift` | Lines 147-156 | `renderStats` property |
| `QLMarkdown/Settings.swift` | Line 190 | Coffee link in `app_version` |
| `QLMarkdown/Base.lproj/Main.storyboard` | Line 78 | Menu item "Buy me a coffee..." |
| `QLMarkdown/Base.lproj/Main.storyboard` | Line 771 | Toolbar item |
| `QLMarkdown/Base.lproj/Main.storyboard` | Line 2154 | Button |
| `Resources/stats.html` | Entire file | Delete file |

### Risk: Low
- Purely removes UI/messaging, no functional changes

---

## 3. Performance Optimizations for Quick Look Preview

### High-Impact Optimizations (Recommended)

#### 3.1 CSS Caching
**Current:** CSS re-read from disk on every render via synchronous XPC
**Location:** [Settings+render.swift:631-643](QLMarkdown/Settings+render.swift#L631-L643)

**Fix:** Cache loaded CSS in memory with invalidation on settings change
- Cache default CSS from bundle (static, never changes)
- Cache custom CSS in Settings instance
- Invalidate cache when `org.sbarex.qlmarkdown-settings-changed` notification fires (already monitored via `startMonitorChange()`)
- **CSS changes will render properly** - cache clears when you save settings in the app
- **Impact:** Eliminates disk I/O per preview
- **Risk:** Low - leverages existing notification system
- **Confidence:** High

#### 3.2 Bundle Resource Caching
**Current:** `getBundleContents()` reads from disk every call
**Location:** [PreviewViewController.swift:141-148](QLExtension/PreviewViewController.swift#L141-L148)

**Fix:** Use static lazy properties for bundled resources
- **Impact:** Eliminates repeated disk reads for default.css
- **Risk:** Low
- **Confidence:** High

#### 3.3 Remove Unnecessary Render Stats I/O
**Current:** `renderStats += 1` writes to UserDefaults on every render
**Location:** [PreviewViewController.swift:220](QLExtension/PreviewViewController.swift#L220), [Settings.swift:152-154](QLMarkdown/Settings.swift#L152-L154)

**Fix:** Remove entirely (since coffee reminders are being removed)
- **Impact:** Eliminates UserDefaults write per preview
- **Risk:** None
- **Confidence:** High

### Medium-Impact Optimizations (Recommended for QL speed)

#### 3.4 Extension Registration Optimization
**Current:** ~15 cmark extensions attached per render via repeated `cmark_find_syntax_extension()` calls
**Location:** [Settings+render.swift:64-356](QLMarkdown/Settings+render.swift#L64-L356)

**Fix:** Cache extension pointers at app launch
- Call `cmark_find_syntax_extension()` once per extension type
- Store pointers in static properties
- Reuse cached pointers in `render()`
- **Impact:** Reduces CPU overhead per render
- **Risk:** Medium - need to ensure extensions are registered before caching
- **Confidence:** Medium-High

#### 3.5 YAML Regex Pre-compilation
**Current:** Regex pattern compiled on every render
**Location:** [Settings+render.swift:112](QLMarkdown/Settings+render.swift#L112)

**Fix:** Use static `NSRegularExpression` instance
- **Impact:** Small CPU improvement
- **Risk:** Low
- **Confidence:** High

---

## Implementation Order

1. **Remove coffee reminders** - Clean removal, no dependencies
2. **Remove legacy WebView code** - Simplifies codebase
3. **Add CSS caching** - Biggest performance win
4. **Add bundle resource caching** - Quick win
5. **Cache cmark extension pointers** - Medium effort, good payoff
6. **Pre-compile YAML regex** - Small quick win

---

## Files to Modify

| File | Changes |
|------|---------|
| `QLExtension/PreviewViewController.swift` | Remove nag screen, legacy WebView, add caching |
| `QLMarkdown/Settings.swift` | Remove renderStats, useLegacyPreview |
| `QLMarkdown/Settings+render.swift` | Add CSS caching, extension caching, regex caching |
| `QLMarkdown/AppDelegate.swift` | Remove buyMeACoffee method |
| `qlmarkdown_cli/main.swift` | Remove coffee message |
| `QLMarkdown/Base.lproj/Main.storyboard` | Remove menu/toolbar/button items |
| `Resources/stats.html` | Delete file |
