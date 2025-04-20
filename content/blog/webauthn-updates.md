+++
title = "WebAuthn has gotten easier"
date = "2025-04-19"
in_search_index = true
+++

## What is WebAuthn?

WebAuthn (Web Authentication API) is an extension of the modern browser's Credential Management API.
It uses public key cryptography to authenticate a user instead of relying on a password. It is more
resistant to phishing and man-in-the-middle attacks than something like a password with SMS as a 2FA
mechanism. It's also a lot more convenient than having to leave the context of the application to
copy over a generated token from an app or email. It does, however, require you to have authenticated
the user in some way initially, so ideally you would set up WebAuthn during the user registration
process or provide a page to manage passkeys after they have signed in some other way, such as a magic
link.

To start using WebAuthn with a website (also called the Relying Party), you must first register a
credential. The website initiates the request by creating a sequence of random data called a challenge.
This challenge needs to be signed, so it will have to be persisted somewhere for the duration of the flow.
This is combined with some additional data we'll get to later to create a [PublicKeyCredentialsOptions](https://developer.mozilla.org/en-US/docs/Web/API/PublicKeyCredentialCreationOptions).
Once you have your options, you call `navigator.credentials.create`. This will prompt an interaction
from the user to go through an authentication flow on the device of their choice. If it succeeds,
the browser receives a credential which includes a signed attestation based on the challenge. The client
submits this to the server which validates the credential based on the origin and challenge. Once this is
done, the credential is ready to use. The server only ever receives the public key for the credential,
which is easier to store safely than a password or private key for a TOTP generator.

To log the user in, the server once again generates a challenge and persists it somewhere. The client
takes that challenge and prompts the browser for credentials. Once a credential is chosen, the challenge
is signed and submitted to the server where it will be validated. If that succeeds, the user is authenticated
and the flow is complete.

To get more details, see the [MDN article](https://developer.mozilla.org/en-US/docs/Web/API/Web_Authentication_API#webauthn_concepts_and_usage).
There are also other credential types available for different use cases. I have not interacted with them.

### Highlights

*   The server never has to worry about persisting secrets about the user. It only ever receives the public key
    for the credential. There's no password database to be stolen.
*   Credentials are scoped to the origin of the relying party (website), which makes phishing much harder. If
    someone is domain squatting a similar URL with a spoofed login page, your credentials won't be available.
*   When generating credentials, the relying party can require an authenticator that performs an authentication gesture.
    This would be something like entering a PIN or biometric scan. This provides two factors: the private key (something you have)
    and the gesture (something you are or something you know). A properly configured passkey can avoid the need for an out-of-
    band second factor.
*   Credentials can be made [discoverable](https://developer.mozilla.org/en-US/docs/Web/API/Web_Authentication_API#discoverable_credentials_and_conditional_mediation), which
    means you don't even need to enter an email first so that the server can find credential IDs with which to prompt the browser. GitHub currently appears
    to operate this way.

## Dealing with PublicKey options and credentials

When WebAuthn first came out, web developers had to do some unfamiliar data format shuffling to handle
creating the credential creation options and again to send the credentials to the server. You would receive
a challenge from the server, likely as a string, and then had to convert it to one of the browser's binary
formats like `TypedArray`. Other values had to be converted as well.

Once you had a credential on the client, you had to go the other direction for the server. Generally, this
exchange was done with `base64url`. Server-side libraries often encouraged you to use a client library like [webauthn-json](https://github.com/github/webauthn-json)
or provided a demo app with some JS to shuffle the data for you. This was inconvenient at best. You would also
be beholden to the library to continue to support new API options.

I am happy to report that things have gotten a bit better recently. `PublicKeyCredential` now has a
[parseCreationOptionsFromJSON](https://developer.mozilla.org/en-US/docs/Web/API/PublicKeyCredential/parseCreationOptionsFromJSON_static) method
which will handle those format differences for you natively. It also now has `toJSON` which makes it much easier to send the credential
to your server once ready. This is supported now by all major browsers. You should be able to make use of it in your code right now.

This is an exciting advancement from browsers. The data format shuffling didn't add any value or extension points for the developer
and so was just tedious. My first implementation of this technology was mostly me fixing my implementation of the shuffle and remembering
which fields required it and which ones were normal values.

## WebAuthn Sandbox

When you are developing the registration and authentication flows, you might not want to pollute your real devices with throwaway
passkeys. Not all of them provide an easy way to clean up unused keys. Fortunately, Chrome offers a sandbox for credentials which
you can enable temporarily while your dev tools are open. From your dev tools screen, open the options menu and select "More tools."

![webauthn menu example](/webauthn-updates/webauthn_tool.png)

Once you select it, enable the sandbox. From here, you can configure your authenticator, such as allowing it to support user verification.

![configure example](/webauthn-updates/configure.png)

As an added benefit, Chrome will respond to these authenticator requests instantly and not actually require you to perform any authentication
gestures with them. This means less time entering PINs and more time testing your implementation. Here's how it looks once you've added
and made use of the authenticator:

![use example](/webauthn-updates/example_credential.png)

## Discoverable Credentials

When I did my first implementation of WebAuthn, the authentication flow required the user to first input their
email or username. From there, the server could look the user up, find their stored credential IDs, and provide
that in the `allowCredentials` option.

Now, there is an option to use [discoverable credentials](https://developer.mozilla.org/en-US/docs/Web/API/Web_Authentication_API#discoverable_credentials_and_conditional_mediation).
You instead provide an empty array for `allowCredentials` and use `mediation: "conditional"`. The browser will show the user all credentials available for that origin and allow the
user to pick from them. Credentials have a unique ID associated with them that you can then use to look the user up on the server to complete the flow.
When you create credentials for the user, make sure to populate the `user` field accurately, as that will help them pick the correct credential if they have multiple logins for the
application.

It's pretty nice not having to type anything at all to get authenticated.

## Are passkeys bad UX?

There is a post that has been making the rounds for a while, [Passwords have problems, but passkeys have more](https://world.hey.com/dhh/passwords-have-problems-but-passkeys-have-more-95285df9) that
I am frequently reminded of when I discuss passkeys. I'll start this out by admitting that I was an early adopter of physical authenticators like YubiKey, and I am obviously
someone who is interested in these things. I am likely to have biases that make me want to use passkeys more than the average user. I am also curmudgeonly and do not at all appreciate
getting stuck in basic login flows trying to get back into a site I rarely use. It might balance out a little.

The first point brought up is that the implementation is complex. There is some truth to this. There is cryptography involved and several validations to perform in order to have a
correct implementation. Most popular web stacks have working libraries for this, though. The APIs into them are pretty straightforward.
It is definitely more complex in the short run than a password hashing mechanism. It is less complex than adding in a 2FA mechanism, though. You don't have to worry about the deliverability
of your messages or their associated costs, and there is no vendor to sign up with to get things working. Authenticator apps for generating TOTP codes have some of the same issues as passkeys
and also may require user education. While they do not require an additional communication mechanism, you do have to store an additional secret safely, and the user might lose the authenticator.
There are also some additional considerations with password storage. Are you peppering your passwords? Have you ever had to rotate hashing algorithms in a production system?

Another issue mentioned is the storage of passkeys. The user has to be able to present the passkey if they want to log into a service. Similarly, they may simply lose the device that had their passkey.
I would like to address this from two directions:

1.  Users already forget their passwords all the time. Having a reset flow to help them is going to be required in any case. Anecdotally, it seems like people are better at holding onto their
    phones than remembering which password they used on your website. If they're using a password manager, that can just store the passkeys as well, which will be available anywhere they use the password
    manager without the hassle of fighting for control of the password field on the page. More likely it will just end up stored on their mobile device and synced to a cloud account.
2.  The user's phone is already going to be their cross-platform authenticator in most cases. I know the author disputes the QR code scan, but I find that flow to be much more intuitive and pleasant
    than leaving the browser context to find the website in my authenticator list or wait for an email. It's also frightening the number of banking apps that still offer SMS as a second factor. My impression
    here is that the author is comparing passkeys to a flow with no 2FA and assuming the user has "abc123!" as their one and only password for all websites. Compared to that I suppose passkeys are less convenient.
    In most scenarios however, I wouldn't consider that an acceptable arrangement.

Many users will not require education because the login flow they are going to experience is functionally identical to native apps on their mobile device.
2FA with email, SMS, and authenticator apps used to be weird not that long ago and people got used to it. I don't think the UX is perfect,
but compared to the drawbacks of passwords, I believe passkeys offer a better alternative.