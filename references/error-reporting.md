# Error Reporting Reference

How a plugin reports build problems through NDMF's ErrorReport, and
the policy that assigns a severity to each defect validation finds.
The parameter set validation reads is specified in Parameters.

## Validation policy

- Validate per component, and keep generating the rest: a defect in one
  component must not take down unrelated components on the same avatar.
- Missing condition parameters are an Error and the component is skipped:
  a transition gated on an undefined parameter can never fire, and the
  author gets no other signal.
- A missing feature parameter whose absence the end behaviour handles by a
  documented fallback (for example, a clip-selection order whose
  documentation defines the first clip as the fallback) is NonFatal
  and the component is still generated. Decide each case
  from the behaviour's official documentation, not from convenience.
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
  and FormatHint are virtual and overridable. (As of NDMF
  1.15.0-beta.5; re-check per version.)
- To display a literal message without a localization asset, use a
  passthrough Localizer: constructed for one language whose single lookup
  maps every key to the format string "{0}", with the message itself passed
  as the key and as its own substitution. Title and details both render
  verbatim in the error window.
- Severity is one of the four ErrorSeverity members: Information,
  NonFatal, Error, InternalError. There is no Warning member; the
  generate-anyway case of the validation policy above is NonFatal,
  and the blocking case is Error. Keep the message specific: name the missing
  parameters or the unmet condition, not just that "validation failed".
