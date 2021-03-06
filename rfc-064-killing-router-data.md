## Problem

Currently the official way to add ad-hoc redirects is to [add them into one of
the CSV files in router-data][router-data] and deploy that repo. However, this
is unnecessarily complex; it is slow to run, and requires an explicit
deployment every time; and it often fails because existing redirects have
themselves moved, meaning developers have to do extra work to resolve things.

[router-data]: https://github.digital.cabinet-office.gov.uk/gds/router-data

In addition, publishing-api has taken on the goal of containing the canonical
list of content and routes on the site. But items added directly to via
router-data are not recorded in publishing-api, making this harder to achieve.
Also, since publishing-api now does record history, it can also fulfil the role
of audit log which was one of the reasons behind creating router-data in the
first place.

Therefore we should aim to migrate the functionality of adding redirects into
that publishing-api. There are a few ways this could be done:

- Create a minimal standalone “redirects” app that puts redirect items into
  publishing-api
- Add a web interface to publishing-api to add redirects (and potentially other
  content-items) directly
- Get developers to PUT new redirects to publishing-api via curl or equivalent
- Write a rake task that adds a redirect, which can be called via the existing
  rake runner in Jenkins - or add a specific Jenkins job to call this task with
  the relevant parameters

Whichever option is chosen, it will also be necessary to backport all the
existing redirects from router-data into publishing-api via a rake task or
migration.

## Proposal

### Option 1: Standalone redirects app

We create a new publishing app whose only responsibility is to add redirects to
publishing-api. This could be very simple, perhaps only allowing the
functionality to put redirect content items and not even supporting browsing
existing items.

- Pros: simple interface, easy to add redirects
- Cons: added complexity of maintaining a whole new app for a single task

### Option 2: Add a web interface to publishing-api

Similar to option 1, except that the interface to add redirects lives directly
in publishing-api.

- Pros: simple interface, no need for separate app
- Cons: Needs added configuration to make publishing-api directly addressable;
  undesirable precedent in turning API into a web app

### Option 3: Get devs to send new redirects to publishing-api manually

This involves no up-front developer work. Under this option, to add a redirect
devs would have to send PUT requests manually on the command line via curl or
equivalent.

- Pros: No work required except for documentation
- Cons: Annoying for devs, as it would require constructing the request in the
  appropriate format on the command line each time

### Option 4: Create a rake task in publishing-api to add a redirect

We add a rake task in publishing-api which calls the existing command actions
to create a new redirect item, passing the correct payload for a redirect with
the specific parameters (old path, new path, redirect type) interpolated. This
task could be run from the generic Jenkins rake task runner, or a new job could
be added to Jenkins to accept the parameters specifically and run the rake
task.

- Pros: Simple to set up, relatively simple to use
- Cons: No helpful interface

## Summary of discussion

An alternative proposal was made, namely to modify the existing
short-url-publisher app to include this functionality. This will enable moving
this task so it is no longer the responsibility of developers but can be done
by content editors, for example.

The requirements for this work are as follows:

1. The purpose of this functionality is to manage ad-hoc redirects that are not
   owned by any other publishing app. This will be enforced by the existing
   path reservation feature in publishing-api.
2. This functionality should be “fully migrated” from the start: i.e. it does
   not keep redirect data locally (except for users from gds-sso), and writes
   directly to publishing-api. This may involve migrating the existing
   Redirects functionality within this app; however it probably does not make
   sense to migrate the "request a short url" part of the app.
3. Users can add redirect by specifying from\_path, to\_path, and type
   (exact/prefix).
4. Additionally it would be useful to have an optional Comment or Reason field
   to store the reason the redirect was added, to take the place of the commit
   comment in router-data. This will require a schema change.
5. Nice to have: a bulk add interface to allow multiple redirects to be created
   and published at once.
