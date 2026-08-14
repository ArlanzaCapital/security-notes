# Access Control Notes

## Core idea

Authentication answers **who the user is**. Authorization answers **what that user is allowed to do**.

A resource identifier being difficult to guess is not, by itself, an authorization control. The server should verify that the current user is permitted to read or modify the requested resource.

## Questions I ask while testing

- What object or action is being protected?
- Which user or role should be allowed to access it?
- Where is the authorization decision enforced?
- Does the server make the same decision for read and write operations?
- What changes when the request is replayed from another authorized test account?

## Useful comparison pattern

When testing an object-level authorization boundary in an authorized environment:

1. Create or identify an object owned by test account A.
2. Observe the legitimate request made by account A.
3. Use test account B and change only the minimum necessary identifier or context.
4. Compare status code, response body, side effects, and subsequent application state.
5. Confirm whether the server actually exposed or modified data that account B should not control.

## Common false positives

- The identifier changes, but the returned object still belongs to the current user.
- The server returns only public information.
- A request appears successful but no protected state changes.
- The UI hides an action but the server still correctly rejects unauthorized requests.
- Different accounts are intentionally allowed to access the same shared resource.

## Reporting reminder

A useful report should describe the broken authorization boundary, not just the fact that an identifier can be changed.

Include:

- the two relevant user roles or accounts;
- the resource that should be protected;
- the minimal reproduction steps;
- evidence of unauthorized read or write access;
- the practical impact.
