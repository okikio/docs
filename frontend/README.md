# Build a browser interface that users can follow

A frontend connects a person's goal to a system operation. The user needs to know
what they can do, whether an action succeeded, and how to recover when it fails.
The maintainer needs to locate the code and state that produce those results.

This is an overview. The repository does not yet contain separate frontend topic
guides or a runnable frontend application.

## Start with one complete user flow

Write down the starting state, action, result, and recovery path before choosing
components. For example, a form that creates a record needs more than a submit
button:

| State | What the user needs | What the system must define |
| --- | --- | --- |
| Before submission | Clear labels and required fields | Input shape, defaults, and authorization |
| Submitting | Visible progress | Request ownership and whether another submission is allowed |
| Success | Confirmation and the saved result | Authoritative response and local state update |
| Invalid input | Field-specific correction | Validation errors that map back to the form |
| Request failure | An explanation and a safe next action | Whether the request reached the server and whether retry can duplicate work |

Trace that flow through the route, component, request, server operation, and state
update. Keep product language in the interface; put diagnostic detail in logs or
developer tools. Test refresh, back navigation, keyboard use, and a failed request
where they affect the flow.

## Use the web platform deliberately

Semantic HTML gives elements defined roles and behavior. Use a `<button>` for an
action and an `<a>` for navigation. Give form controls visible labels. Use CSS
for layout and presentation, and add JavaScript for the behavior the flow needs.
See the [HTML Standard](https://html.spec.whatwg.org/) for element definitions.

Choose a framework from the application's requirements, existing code, and team
experience. Record the reason for a new dependency. A framework does not remove
the need to understand document structure, focus, navigation, or request behavior.

### Understand work on the main thread

Synchronous JavaScript runs until it returns or throws. A long task can delay
input handling and rendering. Promise reactions use microtasks; a microtask
checkpoint runs queued microtasks before the event loop continues. Timers queue
tasks and do not promise an exact execution time.

This is why `await` does not automatically move CPU-intensive work off the main
thread. Split or move expensive work when measurements show it blocks the user.
The [HTML event-loop processing model](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model)
defines task selection, microtask checkpoints, and rendering opportunities.

## Measure the experience users receive

[Core Web Vitals](https://web.dev/articles/vitals) cover loading, responsiveness,
and layout stability:

| Metric | What it measures | Good threshold |
| --- | --- | --- |
| Largest Contentful Paint (LCP) | Loading of the largest visible content element | 2.5 seconds or less |
| Interaction to Next Paint (INP) | Responsiveness to user interaction | 200 milliseconds or less |
| Cumulative Layout Shift (CLS) | Unexpected layout movement | 0.1 or less |

Assess these thresholds at the 75th percentile of page visits, separating mobile
and desktop. INP replaced First Input Delay (FID) as a Core Web Vital. Laboratory
measurements help diagnose a problem; field measurements show what users encounter.
The metrics do not establish whether a user can complete the task, so also test
the flow itself.

## Make the flow accessible

Choose the accessibility requirements that apply to the product and verify the
whole flow against them. [WCAG 2.2](https://www.w3.org/TR/WCAG22/) organizes its
success criteria under four principles: perceivable, operable, understandable,
and robust. Conformance levels are A, AA, and AAA.

Start with semantic controls, labels, visible focus, keyboard navigation, useful
error messages, and sufficient contrast. Check what happens when focus moves,
content updates, or a dialog opens. Automated checks can find some failures;
keyboard and assistive-technology review are also needed for the affected flow.

## Connect the interface to its API

Document which component starts each request, where pending and saved state live,
and how responses update that state. Keep server authorization and validation
independent of client-side checks. Describe retry behavior before adding a retry
button; a failed network response does not prove the server did no work.

Use the [API guides](../apis/README.md) for error responses, authentication,
idempotency, and observability. When a flow fails, maintainers should be able to
connect the user-visible error to the responsible request without exposing
sensitive diagnostic data in the interface.

## Verify and hand over

Record the command that starts the application, required configuration, and a
representative account or fixture. Exercise the normal path and important error
states. State which browser and environment you used, what passed, and what remains
unverified. Link the flow description to its route and controlling components so
the next maintainer can change it with confidence.

[Return to the guide index](../README.md)
