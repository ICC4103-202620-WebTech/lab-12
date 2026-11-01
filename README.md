# Lab 12 — Wheelhouse: Who May Do What

**Monday, November 2** · Covers Class 13 · Learning Outcome 3

## Objective

Every account in Wheelhouse can open every page and press every button. The owner wants each person
to reach only their part of the workshop: the counter takes bikes in and talks to customers, each
mechanic works on the repairs assigned to them, and the owner runs everything, the price list
included.

Authorization is built with **Pundit**. Each rule is written once, in a policy, and the controller,
the index and the view all ask that policy. A request that the policy refuses ends as a `403` page,
whether it came from a button or was written by hand.

## The deliverable

**The same repository.** `webtech-wheelhouse`, with this lab's commits on top of the Lab 11 history.

**Canvas.** Submit the URL of your repository. This lab is **individual**.

---

## Requirements

### 1. Pundit

- Authorization is built with the **Pundit** gem.
- **There is one policy per model that has pages**: customers, bikes, repairs, services and staff.
  Each policy has one method per action that the controller exposes.
- **The base policy refuses everything.** An action whose policy method has not been written is
  refused, never allowed.
- The policies receive the account of the current session. No policy reads the session, `params`
  or `Current` on its own.
- **No role name is a string literal in a policy.** The policies use the methods of the enum.

### 2. The rules

These are the rules of the workshop. Every one of them is enforced by a policy.

| Resource | Owner | Counter | Mechanic |
|---|---|---|---|
| Services | Creates, edits and deletes | Reads | Reads |
| Staff | Creates, edits and deletes | Reads | Reads |
| Customers and bikes | Everything | Creates, edits and reads | No access |
| Repairs: reading | Every repair | Every repair | **Only the repairs assigned to them** |
| Repairs: taking in | Yes | Yes | No |
| Repairs: editing | Every repair | Every repair **not handed back** | **Their own repairs, not handed back** |
| Repairs: deleting | Yes | No | No |

- The list and the page of the services stay public, as they are.
- **A mechanic's own repairs are the ones whose mechanic is the member of the staff joined to their
  account.** An account with the role `mechanic` that belongs to no member of the staff has no
  repairs of its own, and reads none.
- **A handed-back repair** is a repair in the final state of the lifecycle in
  `docs/domain-model.md`. Only the owner edits it.
- The account with the default role, created from the sign-up page, has exactly the permissions of
  its role.
- **No page lists, edits or deletes accounts.** This does not change.

### 3. Every index shows only what its reader may read

- **Every index of a resource with pages is built from a policy scope.** The scope of the repairs
  returns every repair to the owner and the counter, and only their own to a mechanic.
- **The scope and the method that protects the record's page give the same answer.** A repair that
  a mechanic does not see in the index is a repair they cannot open, and the reverse.
- The scope is a relation: the filters and the order of the index are chained on it. A mechanic who
  filters the repairs by state, or who sees the overdue ones, sees only their own.
- Every other page that lists repairs follows the same scope. A mechanic who reaches a list of repairs
  from any page sees only their own.
- The repairs index runs a fixed number of queries for each of the three roles. Doubling the seed
  does not change the count.

### 4. No action is left unchecked

- **Every action of every controller calls the policy**, or is a public action listed in
  `docs/decisions.md` with the reason it is public.
- **An action that forgets to ask the policy fails in development**, and so does an index that is
  not built from a scope. A new controller is covered without a line written in it.
- Signing in, signing out, signing up and the pages of a forgotten password keep working, for a
  visitor and for an account.
- **The API does not change.** Every route of `/api/v1` answers as before, with the token, and with
  the same statuses and documents.

### 5. A refused request

- **A request that the policy refuses answers `403`**, with a page in the layout of the application
  that tells the person they are not allowed to do that. A `500`, or an error page of Rails, does not
  meet this requirement.
- A refused request writes nothing. The record is unchanged after a refused `update` or `destroy`.
- **A refused form shows the `403` page.** The browser does not stay on the form with nothing on the
  screen.
- **A mechanic who opens the page of another mechanic's repair, by changing the id in the URL,**
  receives either a `403` or a `404`. The same answer is given for the page of the repair and for
  its edit form.
- A visitor with no session is still sent to sign in. A visitor never sees the `403` page.

### 6. The views follow the policies

- **A button, a link or a field that leads to an action the person may not perform is not on the
  page.** This applies to the navigation bar, the indexes, the record's pages and the forms.
- The navigation bar of each role links only to the indexes that role may open.
- The page of a repair shows its bike and its customer to everybody who may read the repair. They are
  links only for a person who may open the bike and the customer.
- No view checks a role. The views ask the policy.

### 7. Each role writes its own fields

The repair's form is shared by the three roles, and the policy decides which fields each one writes.

- **A mechanic changes the state and the diagnosis of their repairs, and adds intake photos.**
  Nothing else: not the bike, not the mechanic, not the promised day, not the lines or their prices.
- The owner and the counter change every field the form offers.
- **The list of permitted attributes comes from the policy**, according to the person who asks. The
  controller has no conditional on the role.
- The form shows each person only the fields they may change.
- **A field outside the person's list, added from DevTools, is dropped** and the request goes on,
  as with any field outside the permitted list. A mechanic who sends `repair[mechanic_id]` or a
  charged price, by hand, leaves both unchanged.
- The account that took a repair in is still read from the session and is never a field.

### 8. Brakeman

- **`bin/brakeman` reports no warnings** on the application.
- A warning that is a false positive is ignored with Brakeman's own ignore file.

### 9. Seed data

- **At least two accounts have the role `mechanic`**, each joined to a member of the staff, and
  each with at least two repairs assigned.
- At least one repair has no mechanic assigned.
- At least one repair of each mechanic is handed back, and at least one is not.
- From an empty database, `bin/rails db:seed` finishes without errors. Running it a second time
  leaves the same number of rows in every table, and every account still signs in with its password.

### 10. The permissions document

`docs/permissions.md` holds one table per resource: one row per action, one column per role, and in
each cell whether that role may perform it and under which condition. The table says what the
policies do. A rule in the table that the policies do not enforce, or a rule in a policy that the
table does not show, does not meet this requirement.

### 11. The README

The README is graded on one criterion: **someone who has never seen your repository can clone it
and get it running by following only the README.**

- Next to each account of the seed, it says what that account can do, in one line, and links to
  `docs/permissions.md`.

---

## Documentation

- Pundit — [README](https://github.com/varvet/pundit): policies, scopes, `authorize`, the methods
  that ensure policies and scopes are used, rescuing a denied authorization, the current user, and
  strong parameters
- Rails Guides — [Securing Rails Applications](https://guides.rubyonrails.org/security.html):
  the sections on authorization and on cross-site request forgery
- Rails Guides — [Action Controller Overview](https://guides.rubyonrails.org/action_controller_overview.html):
  filters, `after_action` and `skip_after_action`, `rescue_from`
- Rails Guides — [Layouts and Rendering in Rails](https://guides.rubyonrails.org/layouts_and_rendering.html):
  `render` with a status
- OWASP Top 10 — [Broken Access Control](https://owasp.org/www-project-top-ten/) and
  [Mass Assignment Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html)
- Brakeman — [Documentation](https://brakemanscanner.org/docs/) and
  [Ignoring False Positives](https://brakemanscanner.org/docs/ignoring_false_positives/)

## Deadline

**Tuesday, November 3, 20:00**, on Canvas.
