# Mail (@nestjs/mail)

## Contents
- Registration
- Templates
- Sending reliably
- Transports
- Localized mails
- Testing and production

## Registration

- `MailModule` is global and provides `Mailer`, `MailEvents`, `MailTransport`, `MailTemplateEngine`. `transport` is required. A missing transport or invalid `from` stops startup. https://docs.nestjs.com/application/mail#module-options
  ```typescript
  MailModule.forRoot({
    transport: new FileMailTransport({ directory: 'var/mail' }),
    templates: new FileTemplateEngine({ dir: join(import.meta.dirname, 'mail/templates'), layout: 'layout' }),
    from: 'Orders <orders@example.com>',
  }),
  ```
  https://docs.nestjs.com/application/mail#register-the-mail-module
- Build the production transport in a `forRootAsync()` factory chosen from env, so a typo or missing `SMTP_URL` fails at startup. Classes, `imports` and `isGlobal` sit next to `useFactory`, the factory returns instances. https://docs.nestjs.com/application/mail#send-in-production
- TypeScript does not copy templates: add `"assets": ["mail/templates/**/*"]` and `"watchAssets": true` to `nest-cli.json`. https://docs.nestjs.com/application/mail#register-the-mail-module

## Templates

- `FileTemplateEngine` speaks a logic-less Handlebars subset: no helpers, no expressions, own properties only. Format prices and links in the mail class. https://docs.nestjs.com/application/mail#template-syntax
- A missing value renders as nothing. A double-braced value outside element content or a quoted attribute is a compile error. Triple braces insert raw HTML. https://docs.nestjs.com/application/mail#write-the-order-confirmation-mail
- `cache` defaults to `true`, keep it on in production. Locale lookup for `pt-BR`: `name.pt-BR.html`, `name.pt.html`, then `name.html`. https://docs.nestjs.com/application/mail#template-engine
- The text part is derived from the HTML unless `name.txt` exists. https://docs.nestjs.com/application/mail#write-the-order-confirmation-mail
- Mail classes are injectable providers implementing `Mailable<T>`: https://docs.nestjs.com/application/mail#write-the-order-confirmation-mail
  ```typescript
  @Injectable()
  export class OrderConfirmationMail implements Mailable<OrderConfirmation> {
    render({ order, customer }: OrderConfirmation) {
      return { subject: `Your order #${order.number}`, template: 'order-confirmation', context: { customer } };
    }
  }
  await this.mailer.send(OrderConfirmationMail, { to: { name, address }, data: { order, customer } });
  ```
- A recipient string must hold exactly one address (a comma list throws). Prefer `{ name, address }`. https://docs.nestjs.com/application/mail#write-the-order-confirmation-mail
- Never build an attachment `path` from user input. Every mail whose layout references a `cid` image must attach it. https://docs.nestjs.com/application/mail#attach-the-invoice-and-preview-the-mail
- Register the preview controller only outside production. https://docs.nestjs.com/application/mail#attach-the-invoice-and-preview-the-mail

## Sending reliably

- Never send inside the database transaction. Sending right after commit can lose the mail (crash) or fail an already successful request: send from the outbox. https://docs.nestjs.com/application/mail#send-after-the-order-commits
  ```typescript
  @OnOutboxMessage('order.placed', { consumer: 'order-confirmation-mail' })
  // in the handler, with message and signal from the handler context:
  await this.mailer.send(OrderConfirmationMail, { /* to, data */ idempotencyKey: message.id, signal, retry: false });
  ```
- `retry: false` in the handler avoids multiplying retries (the mailer retries 3 times by default). Turn `error.permanent` into `NonRetryableMessageError`. https://docs.nestjs.com/application/mail#send-after-the-order-commits
- `idempotencyKey` derives the `Message-ID`, so SMTP duplicates share it. `ResendTransport` sends it as `Idempotency-Key`. Postmark, SendGrid and SES have no keys, so a redelivery can reach the customer twice. https://docs.nestjs.com/application/mail#send-after-the-order-commits
- Errors extend `MailError` with `permanent`. SMTP 4xx, 429 and timeouts are transient, 5xx permanent. https://docs.nestjs.com/application/mail#errors

## Transports

- `SmtpTransport` (own client): port 587 requires STARTTLS by default and refuses plaintext. Keep certificate verification, add a private CA with `tls.ca`. `pool: true` for up to 5 connections. https://docs.nestjs.com/application/mail#send-in-production
- DKIM: RSA is the safe choice (not every receiver verifies Ed25519). https://docs.nestjs.com/application/mail#send-in-production
- HTTP transports (`ResendTransport`, `PostmarkTransport`, `SendGridTransport`, `SesTransport`) use `fetch`, no SDK. https://docs.nestjs.com/application/mail#send-in-production
- `FileMailTransport`, `LogMailTransport` and `InMemoryMailTransport` are for development and tests. The module warns when the file or log transport runs with `NODE_ENV=production`. https://docs.nestjs.com/application/mail#production-checklist
- `app.enableShutdownHooks()` lets the relay finish mails and close the SMTP pool on deploy. https://docs.nestjs.com/application/mail#send-in-production

## Localized mails

- Pass `locale` to `mailer.send()` explicitly. `I18nService` otherwise falls back to the current request's locale, which is wrong in outbox handlers or an admin's request. https://docs.nestjs.com/application/mail#send-it-in-the-customers-language
- Custom engines receive `{ locale }`: put only a validated language code into file names. https://docs.nestjs.com/application/mail#use-another-template-engine

## Testing and production

- `overrideProvider(MailTransport).useValue(new InMemoryMailTransport())`, then `assertSent({ to, mail })` and `.link('/reset-password')`. https://docs.nestjs.com/application/mail#testing
- Configure SPF, DKIM and DMARC. Watch `MailEvents` or the `nestjs:mail:*` diagnostics channels. https://docs.nestjs.com/application/mail#production-checklist
