# Pi Developer Architecture Guide

The authoritative orientation hub is [`index.qmd`](index.qmd). The rendered
order is defined in [`_quarto.yml`](_quarto.yml); numeric filenames are stable
identifiers, not reading order.

## Rendered order

1. **Start here**
   - `index.qmd`
   - `01-typescript-for-pi.qmd` — short Pi-oriented TypeScript route
   - `02-architecture.qmd`
2. **Runtime**
   - `03-prompt-lifecycle.qmd`
   - `10-agent-core.qmd`
   - `08-built-in-tools.qmd`
3. **Build extensions**
   - `04-extensions.qmd` — canonical first runnable extension
   - `05-customization-and-workflow.qmd` — production follow-up
4. **Subsystems and contribution**
   - `06-sessions-and-context.qmd`
   - `07-resources-settings-and-trust.qmd`
   - `09-models-providers-and-auth.qmd`
   - `11-tui-and-interactive-mode.qmd`
   - `12-testing-debugging-and-contribution.qmd`
5. **Practice and reference**
   - `13-guided-source-tours.qmd`
   - `14-extension-event-reference.qmd`
   - `15-modes-sdk-and-operations.qmd`
   - `16-failure-and-recovery-reference.qmd`
   - `glossary.qmd`
6. **Optional appendix**
   - `17-typescript-foundations-workbook.qmd` — full foundations, exercises,
     TypeBox practice, and TypeScript reference material

Filesystem readers should follow the same sequence. Chapter numbers remain in
filenames so source links and stable identifiers do not need to change merely
because the learning route changed.
