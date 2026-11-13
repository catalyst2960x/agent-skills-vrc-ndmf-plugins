# Error Reporting Reference

How a plugin reports build problems through NDMF's ErrorReport, and
the policy that assigns a severity to each defect validation finds.
The parameter set validation reads is specified in Parameters.

## Validation policy

- Validate per component, and keep generating the rest: a defect in one
  component must not take down unrelated components on the same avatar.
- Parameter names are not validated: a condition naming a parameter absent from the build-time
  set is not a defect, draws no report, and does not skip the
  component; the name is generated as entered. Name mistakes are
  prevented in the editor, by suggestions, not detected here.
- Inapplicability is not a defect: a directive component whose
  target is unset or absent on this avatar, or whose settings do
  not apply to the avatar's current state, is skipped silently and
  draws no report. Validation reports defects in what the author
  configured, not the absence of something the configuration
  might have applied to.
- Structural defects are Errors: target object unset, target outside the
  avatar, no usable payload (for example no non-null clips).
- Sanitize before validating (P-04): validation, generation and name
  collection all read the sanitized arrays, never the raw ones.

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
  verbatim in the error window.
- Severity is one of the four ErrorSeverity members: Information,
  NonFatal, Error, InternalError. There is no Warning member; the
  generate-anyway case of the validation policy above is NonFatal,
  and the blocking case is Error. Keep the message specific: name the
  unmet condition or the missing value, not just that "validation
  failed".
