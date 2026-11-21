# Error Reporting Reference

How a plugin reports build problems through NDMF's ErrorReport, and
the policy that assigns a severity to each defect validation finds.
The parameter set validation reads is specified in Parameters.

## Validation policy

- A plugin never blocks the avatar build. It is non-destructive by
  construction, so no defect can damage data outside its own
  output; where a target cannot be identified unambiguously, it
  writes nothing; and where its own processing fails, it disables
  itself and leaves nothing written. Every defect therefore
  degrades to a skipped component or a defined result, and
  severity has a single role: how prominently the problem appears
  in the report. A required input that is not set is reported
  with severity Information, because the configuration may still
  be in progress. A value that is set but invalid, and a
  contention with another component, are reported with severity
  NonFatal. A build stops only from outside the plugin, through
  the framework or the platform.
- Validate per component, and keep generating the rest: a defect in one
  component must not take down unrelated components on the same avatar.
- Parameter names are not validated (SKILL.md, Loose coupling to
  referenced sets): a condition naming a parameter absent from the
  build-time set is not a defect, draws no report, and does not
  skip the component.
- Inapplicability is not a defect: a directive component whose
  target — the object or component it points at, not an input of
  its own — is unset or absent on this avatar, or whose settings
  do not apply to the avatar's current state, is skipped silently
  and draws no report. Validation reports defects in what the author
  configured, not the absence of something the configuration
  might have applied to.
- Sanitize before validating (P-04): validation, generation and name
  collection all read the sanitized arrays, never the raw ones.
- An enum value outside its declared range — reachable through
  serialized data the inspector never produced — is a set-but-invalid
  value and is reported NonFatal. The handling takes one of two
  forms. Where the value selects a mode of the component's own
  processing, sanitize it to the field's declared default and
  continue. Where the value decides what is generated, do not
  substitute a default: skip the item, because generating a
  different item than the one configured would hide the defect.

## Reporting with ErrorReport

- Report through NDMF: `ErrorReport.ReportError(error)` inside
  `using (ErrorReport.WithContextObject(component.gameObject))` so the
  author can click through to the offending component. Debug.Log is not a
  reporting channel (SKILL.md NEVER #1).
- The error type derives from `SimpleError`. Its abstract members are
  Localizer, TitleKey and Severity; TitleSubst, DetailsKey, DetailsSubst
  and FormatHint are virtual and overridable. (Re-check the members
  per version.)
- To display a literal message without a localization asset, use a
  passthrough Localizer: constructed for one language whose single lookup
  maps every key to the format string "{0}", with the message itself passed
  as the key and as its own substitution. Title and details both render
  verbatim in the error window. A product that ships PO assets for its
  inspector uses the same Localizer for its errors (Inspectors and
  Drawers, Localized labels); the passthrough form is for a product
  that ships none.
- Severity is one of the four ErrorSeverity members: Information,
  NonFatal, Error, InternalError. There is no Warning member. The
  plugin itself assigns only Information and NonFatal (Validation
  policy); Error and InternalError belong to stops that come from
  outside the plugin. Keep the message specific: name the
  unmet condition or the missing value, not just that "validation
  failed".
