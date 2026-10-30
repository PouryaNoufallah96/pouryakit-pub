# Email

`PouryaKit.Email`: one contract, `IEmailSender`, that sends a plain-text mail to one recipient and returns a `Result`. It references `PouryaKit.Domain` only. `PouryaKit.Email.MailKit`: the one provider, the Host's sender on MailKit, configured once from validated options. Namespaces `PouryaKit.Email` and `PouryaKit.Email.MailKit`.

- [Registering](#registering)
- [Sending from a Notification Handler](#sending-from-a-notification-handler)
- [What the Kit guarantees](#what-the-kit-guarantees)
- [Limits](#limits)
- [Testing](#testing)

## Registering

On the Host, which references `PouryaKit.Email.MailKit`:

```csharp
using PouryaKit.Email.MailKit;

builder.Services.AddPouryaKitMailKit();
builder.Services.Configure<PouryaKitMailKitOptions>(builder.Configuration.GetSection("PouryaKit:Email"));
```

```json
"PouryaKit": {
  "Email": {
    "Host": "smtp.example.com",
    "Port": 587,
    "UserName": "shop",
    "Password": "from a secret store",
    "From": "Shop <shop@example.com>"
  }
}
```

- `AddPouryaKitMailKit()` registers the Host's one `IEmailSender`, a singleton. `AddPouryaKitMailKit(configure)` sets the options in code instead, such as `AddPouryaKitMailKit(mail => mail.Security = SecureSocketOptions.None)` for a local test server. A Host that registers another `IEmailSender`, before or after the call, a second call included, fails when its container is built, naming each registration's type, or the factory that builds it, and so does one that removes every `IEmailSender` after it, naming the contract: the registration adds a composition check. A Module references only `PouryaKit.Email` and never names MailKit.
- `PouryaKitMailKitOptions`: `Host` and `From` are required; `Port` is 587 by default; `Security` is MailKit's `SecureSocketOptions`, `StartTls` by default, which refuses a server that offers no TLS (set `None` only for a local test server such as Papercut); `UserName` and `Password` are set together, or neither to send unauthenticated.
- The Host fails when it starts, naming the option, when `Host` is blank (`PouryaKitMailKitOptions.Host names no SMTP server; …`), the port is outside 1 to 65535, `From` is not a mailbox address with a domain, `Security` is not a defined `SecureSocketOptions` value (a number such as `99` binds, and MailKit would connect in plain SMTP for it), or only one of `UserName` and `Password` is set.
- Each send connects to the server, authenticates when a user name is set, sends and disconnects, on a client of its own: concurrent sends never wait for each other, and no idle connection is held for the server to drop. The Kit chose this over one connected client guarded for one send at a time. Once `SmtpClient.SendAsync` has returned (the server's acceptance read), MailKit's own disconnect, as read in its 4.18.1 source, whose `SmtpClient` 4.18.2 leaves unchanged when decompiled, and not proven by a test of the Kit's, swallows a QUIT that fails or is cancelled, so the send stays successful; a failure or cancellation before that return, the window between the server's 250 and MailKit reading it included, and any other exception after it fall under the sent-twice limit below.

## Sending from a Notification Handler

Send mail from a Notification Handler, never from a command's handler or a Domain Event Handler, which run before the commit ([messaging.md](messaging.md)):

```csharp
using PouryaKit.Domain;
using PouryaKit.Email;
using PouryaKit.Messaging;

internal sealed class WelcomeCustomerNotificationHandler(IEmailSender email)
    : INotificationHandler<CustomerRegisteredDomainEvent>
{
    public Task<Result> Handle(CustomerRegisteredDomainEvent domainEvent, CancellationToken cancellationToken) =>
        email.SendAsync(
            new EmailMessage(domainEvent.Email, "Welcome", $"Welcome, {domainEvent.Name}."),
            cancellationToken
        );
}
```

- `SendAsync` succeeds once the mail server has accepted the mail.
- It fails with `EmailErrors.InvalidAddress` (`Email.InvalidAddress`, target `To`), sending nothing, when the recipient is not one mailbox address with a domain, such as `ada@example.com` or `Ada <ada@example.com>`.
- It throws MailKit's exception when the server cannot be reached, refuses the login or refuses the mail. Either way the delivery is retried and, once its retries are spent, set aside as a dead letter with the reason ([messaging.md](messaging.md)).

## What the Kit guarantees

- **A rolled-back Operation sends no mail.** Its Domain Events never reach the Outbox, so no Notification Handler runs for them.
- **A committed Operation's mail is attempted by a durable delivery**: its Notification Handler's delivery is retried with the Host's bounded retries and then set aside as a dead letter. The server's acceptance is not guaranteed, and a send that succeeded may repeat (below).
- **The options are checked before the Host serves**, not at the first send.

## Limits

- **A mail may be sent twice.** Mail is not transactional: when the delivery that sent a mail fails afterwards, such as the handler's own later failure or a crash before its Receipt commits, its retry sends the mail again. Do the send last in its handler, and give each mail its own Notification Handler, so a failure elsewhere does not repeat it. The Kit has no transactional mail and no exactly-once external effect.
- One recipient, one plain-text body, the Host's one sender address. No attachments, HTML body, templates or bulk sending.
- One provider per Host, chosen by the package the Host references; no switch at run time.

## Testing

- The Proving Sample's `EmailTests` run the application on a database of its own with its mail sent to a Papercut SMTP container (`Testcontainers.Papercut`, image `changemakerstudiosus/papercut-smtp:7.8.0`, configured with `Security` `None` and the container's SMTP port), and read what arrived from Papercut's HTTP API (`GET /api/messages`, then `GET /api/messages/{id}` for its recipients and body).
- A test Host that composes a Module with a Notification Handler that sends mail registers a sender too: the Kit's container is validated when it is built.
