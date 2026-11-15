# Template: Severity assignment in a product's reports

## When it applies

A plugin reports author-facing problems through NDMF's
`ErrorReport`. The product's severity policy is fixed: the plugin
never blocks the avatar build (it is non-destructive by
construction, writes nothing where a target cannot be identified
unambiguously, and disables itself where its own processing
fails), so severity is a visibility choice only.

## Requirements

- The assignment is by the kind of defect, decided once for the
  product:
  - A required input that is not set (a missing reference, an
    empty list, an entry with nothing in it): **Information**.
  - A value that is set but invalid (an out-of-range mode, a
    target outside the avatar), and a contention with another
    component (the same object driven twice, conflicting
    assignments): **NonFatal**.
  - **Error** has no assigned cases.
- In every reported case the component is skipped (or the single
  assignment is skipped); reporting and skipping are one act, so
  there is no separate failure helper to keep consistent.
- The report carries the component's GameObject as its context
  object. The error type is one `SimpleError` subclass holding
  the detail string and the severity, with a passthrough
  localizer (the detail text is the message).

## Skeleton

```csharp
// [Editor: NDMF pass]
static void Warn(NDMFComponent component, string detail)
{
    using (ErrorReport.WithContextObject(component.gameObject))
        ErrorReport.ReportError(new PluginError(detail, ErrorSeverity.NonFatal));
}

static void Info(NDMFComponent component, string detail)
{
    using (ErrorReport.WithContextObject(component.gameObject))
        ErrorReport.ReportError(new PluginError(detail, ErrorSeverity.Information));
}

static readonly Localizer Passthrough = new Localizer("en-US", () => new List<(string, System.Func<string, string>)>
{
    ("en-US", key => "{0}"),
});

class PluginError : SimpleError
{
    readonly string detail;
    readonly ErrorSeverity severity;

    public PluginError(string detail, ErrorSeverity severity)
    {
        this.detail = detail;
        this.severity = severity;
    }

    const string Title = "NDMFComponent";

    public override Localizer Localizer => Passthrough;
    public override string TitleKey => Title;
    public override string[] TitleSubst => new[] { Title };
    public override string DetailsKey => detail;
    public override string[] DetailsSubst => new[] { detail };
    public override string FormatHint() => null;
    public override ErrorSeverity Severity => severity;
}
```
