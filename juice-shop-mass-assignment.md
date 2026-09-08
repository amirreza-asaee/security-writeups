# Mass Assignment Leading to Privilege Escalation & Price Manipulation — OWASP Juice Shop

**Target:** OWASP Juice Shop (practice environment)
**Tool used:** Burp Suite (Repeater)
**Category:** Broken Access Control / Mass Assignment (OWASP Top 10 - A01:2021)

## Summary
While testing OWASP Juice Shop, I found that the application accepts extra, unexpected fields in JSON request bodies without validating whether the user is authorized to set them. This is a classic **Mass Assignment** vulnerability, and I was able to exploit it in two ways:

1. Escalating a normal customer account to admin during registration.
2. Manipulating product price and quantity in the shopping cart by sending a negative quantity value.

## Vulnerability 1: Privilege Escalation via Registration

### Steps to Reproduce
1. Intercepted the `POST` request sent during account registration using Burp Suite Repeater.
2. Added an extra field to the JSON body: `"role": "admin"`.
3. Sent the modified request.
4. Logged in with the newly created account — the admin dashboard (with admin-only controls) appeared immediately.

### Root Cause
The registration endpoint binds the incoming JSON directly to the user object without restricting which fields a client is allowed to set. Since `role` is not stripped or validated server-side, an attacker can self-assign admin privileges at signup.

### Impact
Full privilege escalation from an unauthenticated/new user to administrator, with no need to compromise an existing account. This is a **critical** severity issue — it grants complete control over admin-only functionality.

### Suggested Fix
- Never trust client-supplied role/permission fields.
- Use an explicit allow-list of fields the client is permitted to set (e.g. `email`, `password`) rather than binding the whole request body to the model.
- Assign `role` server-side only, defaulting to the lowest-privilege role.

---

## Vulnerability 2: Negative Quantity → Price Manipulation in Cart

### Steps to Reproduce
1. Added a product to the shopping cart.
2. Intercepted the "update cart item" request in Burp Suite Repeater.
3. Changed the `quantity` field to a negative number.
4. Sent the request — the cart accepted the negative value, which caused the total price calculation to go negative as well.
5. This made it possible to reduce the total order cost to a negligible or negative amount while still checking out high-value items.

### Root Cause
The cart update endpoint does not validate that `quantity` is a positive integer within a reasonable range before using it in the price calculation.

### Impact
Financial impact — a user could manipulate order totals to pay far less than the actual price, or potentially exploit the negative total further depending on how payment/refund logic handles it. Severity: **high**.

### Suggested Fix
- Validate `quantity` server-side: must be a positive integer, with a sane upper bound.
- Recalculate and verify final price server-side at checkout, never trusting a client-supplied total.

## Screenshots
*(Burp Suite Repeater — request/response for both cases)*

## Lessons Learned
Both issues stem from the same root problem: **trusting client-supplied data without server-side validation**. This reinforced for me why "never trust the client" is the first rule of API security testing, and why every writable field needs an explicit allow-list rather than being bound blindly.
