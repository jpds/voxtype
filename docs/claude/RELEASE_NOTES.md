# Release Notes and Website News

**Every GitHub release must have a corresponding news article on the website.**

When publishing a release to GitHub, also add a matching article to `website/news/index.html`. The
content should mirror the GitHub release notes.

## Capturing All Features

Before writing release notes, review all commits since the last release to ensure nothing is missed:

```bash
git log --oneline v0.4.14..HEAD  # Replace with previous version tag
```

Check for:
- New features and configuration options
- Bug fixes
- Performance improvements
- Deprecations
- Contributors to credit

Don't just document the most recent work - capture everything that shipped since the last release.

## Style Guide (follow v0.4.10 and v0.4.11 as examples)

**Avoid AI writing patterns:**
- No em-dashes (—). Use regular dashes, colons, or separate sentences instead.
- No "delve", "leverage", "utilize", "streamline", "robust", "seamless"
- No excessive hedging ("It's worth noting that...", "Interestingly...")
- No formulaic transitions ("Let's dive in", "Without further ado")
- No punchy one-liner endings to paragraphs ("And that's the point.", "Simple as that.", "No
  thoughts, just vibes.")
- No sentence fragments for dramatic effect ("The result? Faster builds.", "The fix? Simple.")
- Write plainly and directly. The existing news posts are the voice to match.

**GitHub Release Notes (Markdown):**
- Version and headline in title: "v0.4.11: Remote Whisper, Cancel Transcription, Output Mode
  Override"
- Brief intro paragraph summarizing the release
- `###` sections for each major feature
- **"Why use it:"** callouts explaining the user benefit
- Code blocks with examples (config snippets, CLI commands)
- Bug fixes as a bullet list
- Downloads table and checksums at the end

**Website News Article (HTML):**
- Add new article at the top of the articles list in `website/news/index.html`
- Use the `id` attribute for anchor links (e.g., `id="v0411"`)
- `article-meta` with date and `<span class="article-tag">Release</span>`
- Same h2 title as GitHub release
- h3 subsections matching the GitHub structure
- **Why use it:** in `<strong>` tags
- Code blocks wrapped in `<div class="code-block">` with optional `<div class="code-header">` for
  labels

**Example structure:**
```html
<article class="news-article" id="v0412">
    <div class="article-meta">
        <time datetime="2026-01-15">January 15, 2026</time>
        <span class="article-tag">Release</span>
    </div>
    <h2>v0.4.12: Feature Summary Here</h2>
    <div class="article-body">
        <p>Intro paragraph...</p>

        <h3>Feature Name</h3>
        <p>Description of what it does.</p>
        <p><strong>Why use it:</strong> User benefit explanation.</p>

        <div class="code-block">
            <div class="code-header"><span>config.toml</span></div>
            <pre><code>[section]
option = "value"</code></pre>
        </div>
    </div>
</article>
```

**Checklist for releases:**
1. Create GitHub release with notes following the style above
2. Add matching article to `website/news/index.html`
3. Update download examples in `website/index.html` (deb/rpm URLs with new version)
4. Update `packaging/arch-bin/voxtype-bin.install` post_upgrade() message with current version
   highlights
5. Commit and push website changes
6. Push AUR package updates