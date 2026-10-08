# Osysharp.Resend — transactional email, in a few lines of Osy#

Send email through [Resend](https://resend.com) — one message or a batch of a hundred, plain text and HTML, replies
and attachments — from a class written entirely in Osy# over the public `Http.*` surface.

## Use case

Any app that has to tell somebody something: a receipt after a payment, a password-reset link, an invoice with the
PDF attached, a "your export is ready", a maintenance notice to every account at once. The app writes the message
and hands it over; the provider does the delivering.

Reach for this when you want mail out of the box and you are happy for Resend to be the sender. It is a good default
— a verified domain, a decent deliverability story, and an API small enough that the whole binding fits on one
screen. If you already send through somebody else, fork it: the shape below is the same for any provider with an
HTTP API.

It has no user interface, so there is nothing to screenshot. What you see is an email.

## Install and use

```osy
// app.osy
app Shop {
  model "**/*.osy";
  use Osysharp.Resend@0 { egress "api.resend.com"; }
}
```
```osy
using Osysharp.Resend;

app.Secrets = [ new Secret("ResendApiKey") ];

ResendMailer Mail() { return new ResendMailer { ApiKey = Secret.ResendApiKey, From = "Acme <noreply@acme.com>" }; }

void EmailReceipt(Order order) {
  Mail().Send(new EmailMessage {
    To      = [order.Customer.Email],
    Subject = $"Your receipt for order {order.Reference}",
    Text    = $"Thank you. We charged {order.Total:C} to the card ending {order.CardLast4}.",
  }, order.Reference);           // ← the idempotency key: a retry of this handler sends ONE email
}
```
⭐ **One factory, then every call site uses it.** A function that RETURNS the configured client compiles: the compiler
notes that the credential field was assigned a secret, so the object may travel between your functions while the field
itself cannot be read back out — returned, logged, stored, copied or sent to a page. Holding the client in a component
is refused (an object reaches the browser with its fields), and so is calling the factory from a request, so the key
cannot leave the server by any of those routes. See `osy docs security` → credential fields.

```console
osy secret set ResendApiKey
```

That is the whole surface. `Send` answers Resend's id for the queued message — worth storing beside whatever the
mail is about, because it is the only handle that ties a support question to a delivery.

**The `egress` grant is not optional and not a formality.** A package reaches only the hosts its own manifest asks
for *and* your app grants. Leave it out and the compile fails naming the exact line to add; it is how you decide,
once and in your own source, that this dependency may talk to `api.resend.com` and nowhere else.

**An API key that is not yours.** `ApiKey` is a setting on the mailer rather than a secret the package declares, so
the app names its own secret and this package never learns which. Nothing here can read a secret it was not handed.

## What it ships

| file | what it is |
|---|---|
| `model/ResendMailer.osy` | `class ResendMailer` — `Send` and `SendBatch`, the pre-call refusals, and the provider's own words on a failure |
| `model/ResendWire.osy` | the request and response shapes, with `[ExternalName]` for the JSON spellings |
| `model/ResendWebhook.osy` | `class ResendWebhook` — verifies a webhook (Svix's signature over `{svix-id}.{svix-timestamp}.{body}`, with a replay window) and answers a `ResendEvent`: the type (`email.bounced`, `email.complained`, …), the addresses, and whether a bounce is permanent |

**List mail.** `EmailMessage.Headers` carries headers of the mail itself; `ListUnsubscribeHeaders(url)` builds the
`List-Unsubscribe` / `List-Unsubscribe-Post: List-Unsubscribe=One-Click` pair (RFC 2369 + RFC 8058) that gives a
reader a one-click "Unsubscribe" beside the sender — Gmail and Yahoo require it on bulk mail. `Osysharp.MailingLists`
hands every list mail's address for it to the app's mailer, and a bounce or complaint verified by `ResendWebhook` is
what its `StopAddress` wants to be told.

Three things it does that a thin wrapper would not:

- **`SendBatch` is ONE call, not a loop.** A hundred separate sends is a hundred round trips and a hundred chances
  to be rate-limited half way through, leaving a job that has mailed some of its recipients and cannot tell you
  which. One call either takes the batch or does not. Resend's batch endpoint takes no attachments, so a message
  carrying one is refused *by name* rather than sent without its file.
- **A message that cannot be sent is refused BEFORE the call**, in your own words: no sender, no recipient, no body.
  Resend answers those with a 422 naming a JSON field you never wrote; these name the property you did write and the
  subject of the mail you meant to send.
- **A failure quotes the provider.** `"422"` sends the reader to a status-code table; `validation_error: The from
  address is not a verified domain` sends them to their DNS, which is where the problem is. A failed send throws
  `ExternalServiceException` — the right default for mail, because a receipt that silently did not go out is the
  worst of the three outcomes. A caller that genuinely wants to carry on can catch it and say so.

### Its own tests

`tests/` is the producer's test project — an ordinary app that depends on the kit by name and stubs `Http.Post`, so
every case asserts **what went on the wire** rather than that a call returned. It is named by the `tests` setting in
`package.osy`, which means two things: it never ships (a consumer does not receive it, compile it or pay for it),
and `osy publish` runs it and refuses to publish a version whose own tests fail.

## Build your own on top

**Another provider** — Postmark, SES, SendGrid, your own relay — is a fork and a rewrite of one file. The shape is
the point: a class with settings, a `Wire(…)` that builds the request, `Http.Post` to send it, and the manifest
declaring the one host it may reach. Fork this repository into your account, rename the package to match
(`package Acme.Mail` lives at `github.com/acme/mail`), change the host in its manifest, and `osy publish`.

**A house style** — every mail from one sender, with a footer and a reply-to — is a class of your own holding a
`ResendMailer` and filling those in, so your app's functions say `Notify(user, "…")` and nothing else knows about
email at all.

⭐ **Nothing in this package is privileged.** It is ordinary Osy# using only public APIs — `JsonSerializer` to build
the body, `Http.*` to send it — so anyone could have written it exactly as it is, for a provider the platform has
never heard of. That is deliberate: **the platform has no `Email.Send`, and will not have one.** Delivery is
somebody's account with somebody's provider, and a stdlib verb would have to pick one. What the platform owes
instead is that expressing it takes a few lines — which is what this kit is, and why the whole of it is under 200
lines with no C# anywhere.
