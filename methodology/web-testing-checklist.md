# Web Testing Checklist

A personal checklist for authorized web security testing. It is intentionally small and will evolve with practical experience.

## Before testing

- Confirm the target is in scope.
- Read the program rules and prohibited actions.
- Note rate limits, testing windows, and disclosure requirements.
- Use a dedicated test account when possible.
- Avoid collecting or retaining unnecessary personal or sensitive data.

## Application mapping

- Identify main application areas and user roles.
- Record authentication and account-recovery flows.
- Note important object identifiers and state-changing actions.
- Map API endpoints observed during normal use.
- Identify file upload, export, search, and integration features.

## Authentication and session

- Check registration and login behavior.
- Review password-reset and account-recovery flows.
- Observe session invalidation after logout and password changes.
- Check whether sensitive actions require recent authentication where appropriate.
- Compare behavior across multiple sessions and accounts.

## Access control

- Compare the same request across accounts with different ownership or roles.
- Test whether changing an object identifier changes authorization behavior.
- Distinguish object existence from permission to access the object.
- Check read and write operations separately.
- Verify server-side enforcement rather than relying on hidden UI elements.

## Input and application behavior

- Identify reflected and stored user-controlled input.
- Observe how special characters and unexpected values are handled.
- Check server-side validation for important state changes.
- Review redirects, URLs, and externally fetched resources where in scope.
- Note verbose errors or unintended information exposure.

## API-specific checks

- Identify object-level authorization decisions.
- Compare fields returned to different users and roles.
- Test whether server-side filters can be bypassed by changing request parameters.
- Check mass-assignment behavior on writable objects.
- Review pagination, filtering, and search for unintended data exposure.

## Before reporting

- Reproduce the behavior at least once when safe.
- Minimize the proof of concept to the smallest reliable sequence.
- Explain the security boundary that is being bypassed.
- Separate observed facts from assumptions.
- Describe realistic impact without exaggeration.
- Include remediation ideas only when they are useful and proportionate.

## Notes to improve later

- Add concrete checks learned from authorized labs.
- Add false-positive patterns encountered in practice.
- Split large sections into dedicated methodology files when needed.
