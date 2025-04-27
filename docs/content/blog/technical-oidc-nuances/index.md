---
title: "Technical: OpenID Connect 1.0 Nuances"
description: "This is a commentary on several troubling trends in the security world, as well as an explainer on some fundamental OpenID Connect 1.0 concepts."
summary: "This is a commentary on several troubling trends in the security world, as well as an explainer on some fundamental OpenID Connect 1.0 concepts."
date: 2025-04-27T23:34:23+10:00
draft: false
weight: 50
categories: ["Technical", "OpenID Connect 1.0"]
tags: ["technical", "specifications"]
contributors: ["James Elliott"]
pinned: false
homepage: false
seo:
  title: "" # custom title (optional)
  description: "" # custom description (recommended)
  canonical: "" # custom canonical URL (optional)
  noindex: false # false (default) or true
---

----

Because of the technical nature we're intentionally not including it on the homepage; but we will probably publish more
articles like this in the future in the same [category](categories/technical).

Learning about [OpenID Connect 1.0] and the associated specifications has been quite a journey. There is a lot to
read and even more to understand. This journey has taken years even with preexisting knowledge; and the following
article represents the elements of the specifications that were not only the more time-consuming ones to grasp entirely,
but also what I now consider the most important ones; at least in the way I understand them today.

While writing this article it is becoming clearer and clearer to me that even though this article looks long it will
only take most interested people about 20 minutes to read. I find this frustratingly ironic. I can only hope that these
learnings help current and future [OpenID Connect 1.0] Providers and Relying Parties save time in understanding what I
now consider a wonderfully beautiful framework and technology.

The decision to write this article mainly comes the fact that I have found myself recently and regularly discussing a
few topics associated with [OpenID Connect 1.0] and in particular some of the nuances that don't seem well understood.
I fail to see today why these concepts are hard to understand, they are very clearly detailed in the relevant
specifications; I can only summarize at least from my anecdotal experience no one appears to discuss them in detail.

I suspect most people will find this topic rather boring. However, those of you who find technical articles interesting
will probably really enjoy this, especially if your understanding of the [OpenID Connect 1.0] specifications is
non-existent or just starting to develop.

Several of these concepts are heavily linked together and in my opinion once you realize how each of these concepts
interacts with the others, there is a sort of orchestral design behind each of them; it's almost like the specification
designers actually put a lot of thought into this.

----

## ID Tokens and Access Tokens

When these specifications were first envisioned, there existed no specific format for the [Access Token]. The format was
entirely up to the implementer. Effectively an [Access Token] was completely opaque and meaningless to the party that
was utilizing it. This concept is important, as it is the foundation of the intent of the [Access Token].

Some confusion has developed over the years surrounding the purpose of these tokens. You see they are meant to only be
understood and verified by the Authorization Server or integrated Resource Server. They can be completely opaque.

However as several providers started using the [JSON Web Token] or JWT for these tokens, and the
[JSON Web Token Profile for OAuth 2.0 Access Tokens] was ratified, people are increasingly using these to determine the
identity of a user. The problem is this is not the intent behind these tokens. In fact for other reasons that will be
discussed in the next section of this article it is actually a harmful decision to make.

The intended purpose of [OpenID Connect 1.0] was to solve this particular issue with OAuth 2.0 as well as a few others.
How it solves this particular issue is via the [ID Token]. These tokens are designed to carry information that will
uniquely identify a user.

This becomes more visible when you look at the final audience (`aud` claim) of a [ID Token] vs an [Access Token] using
the JWT format. According to [RFC7519 Section 4.1.3](https://www.rfc-editor.org/rfc/rfc7519.html#section-4.1.3) the
audience identifies the recipients that the JWT is intended for.

The below examples are the example [ID Token] and [Access Token] which are effectively valid in content for an
Authorization Code Flow with the following parameters:

1. Client ID: K2LQE4XRC54N7C2F5ZLF
2. Authorized Audience: https://auth.example.com/api/oidc/introspection
3. Scopes: openid profile email

```json {title="ID Token"}
{
  "jti": "91de5882-ff69-46b6-b13b-165199f3191f",
  "iss": "https://auth.example.com",
  "sub": "d2fdc83d-d7ad-4ced-81d8-0bb87db4a127",
  "aud": "K2LQE4XRC54N7C2F5ZLF",
  "exp": 1745755215,
  "iat": 1745755000
}
```

```json {title="Access Token"}
{
  "jti": "f30450c1-a60c-43ab-b855-e670f84ba45a",
  "iss": "https://auth.example.com",
  "sub": "d2fdc83d-d7ad-4ced-81d8-0bb87db4a127",
  "aud": "https://auth.example.com/api/oidc/introspection",
  "exp": 1745755215,
  "iat": 1745755000,
  "client_id": "K2LQE4XRC54N7C2F5ZLF",
  "scope": "openid profile email"
}
```

As opposed to the Authorization Code Flow where the `sub` should normally be the Resource Owner's subject identifier,
the Client Credentials Flow has an interesting effect on the [Access Token] where the `sub` should normally be the
Client ID. While you could technically try to use this to validate the identity of the end-user this is not the intended
purpose behind this [Access Token] format. In fact it's not even guaranteed by any normal area of the specification
that this will be the case.

If you take a look the audience of the [Access Token] is not the Client ID, this is because it is not the intended
recipient and should not use it to validate the identity of a user. You will also note the audience in the [ID Token] is
the Client ID, it is required to be the Client ID, though it optionally can have additional values. This clearly
expresses that the Client is the intended recipient.

The short version of this is that the [Access Token] is meant to be understood by the Authorization Server for uses at
the [UserInfo Endpoint], [Token Introspection] Endpoint, [Token Revocation] Endpoint, and various other endpoints it
decides to implement; or the endpoints of a Resource Server with deep understanding over how to validate the token. The
[ID Token] is intended to be used by the Relying Party. It's used as a means of a verifiable proof the user is a unique
individual, or at the very least has granted access to their account (under normal circumstances).

Another interesting difference between the [ID Token] and [Access Token] above is that the [Access Token] has a `scope`
claim, but the [ID Token] does not. This may seem like a mistake but I assure you that it isn't. There is no need for
the [ID Token] to have a `scope` claim, as the token has a very clear intention; sharing user identity, and it's only
used for this purpose. The only clearly expressed intention behind an [Access Token] is in the name of the token type;
it's used to _access_ things. This is why the concepts of [Scope] and [Audience] exist; rather than the token having
blanket access, they limit the access to very specific endpoints and actions in a fairly explicitly expressed way.

This concept of what kinds of things these tokens can access specifically in [OpenID Connect 1.0] is going to be touched
on later, specifically when discussing claims availability.

----

## Claim Stability and Uniqueness: The effect on Identity Binding

There are various [Claims] available to implementers in most [OpenID Connect 1.0] Provider implementations. These [Claims]
vary from email addresses, to usernames, to readable names. A troubling trend that I have seen in both Enterprise and
Open Source projects is that they use whatever claim they feel like to bind identities together. In fact many don't even
use them to perform any kind of binding at all, they just decide because the username or email matches that the user is
signed in.

This is not how the specification indicates this should be done. In fact rather than just leaving it up to everyone to
decide how to handle this [OpenID Connect 1.0] clearly spells out that these [Claims] must not be used for this purpose.
Instead it directs our attention the `sub` and `iss` [Claims] being the only
[Stable and Unique](https://openid.net/specs/openid-connect-core-1_0.html#ClaimStability) [Claims] that an end-user can
cleanly be identified by.

The intent and gravity of neglecting this element is very clear as soon as you realize that providers may allow users to
change their email address or username. These values are meant to be anchored to a user, never changing, regardless of
what other values change. Not only for security, but for the user experience to be seamless. Just because they change
their email they should not be prevented from signing into an application; and this would be the case if you do not use
the intended [Claims].

This is linked heavily to the first concept because the [Access Token] may in some way identify the end-user, but it's
not required to. In fact the [Access Token] may not even be a JWT, it's up to the [OpenID Connect 1.0] Provider how
they deliver it, as long as they understand it.

So some astute readers may be thinking. Why do these [Claims] exist in the first place then? Well it comes down to three
primary functions. Obviously these functions are not an exhaustive list, but it should be enough to explain the
principles.

The first function of these [Claims] is to provide helpful information or hints to the Relying Party during a
Registration Flow, or in some instances an Identity Binding Flow (i.e. binding the `sub` and `iss` claims to an identity
that already exists) when they're not already logged in. For example they may prefill a form.

The second function of these [Claims] is to provide the Relying Party with information about a Resource Owner with an
already bound identity that they may not want to store themselves; such as they may perform a Authorization Flow to
temporarily obtain the Resource Owners address information
for a purchase.

The third function of these [Claims] is to provide the Relying Party a way to obtain updated details about a Resource
Owner who already has a bound identity. For example updating their contact details. This neaty ties into the next
concept.

----

## Claims Availability

There are various ways to request and grant [Claims] in the [OpenID Connect 1.0] specification. Many assume that the
[Claims] are either exclusively available in the [ID Token], will always be in the [ID Token], or even worse as we've
previously found out in the [Access Token]. The assumption stems from the fact certain scopes grant the Relying Party
access to certain sets of [Claims]. But what does the specification really intend?

Well if we dive into the [Claims] that are standard in the [ID Token], it has a very minimal set of [Claims] by default.
It can contain more, and in some scenarios it should contain a lot of [Claims].

### Scope Parameter

Scopes seem to be mostly be assumed to always mint an [ID Token] with all the scopes relevant claims. Why is this
assumption mostly flawed?

The section on [Requesting Claims using Scope Values] has a crucial passage regarding this:

> The Claims requested by the profile, email, address, and phone scope values are returned from the UserInfo Endpoint,
> as described in Section 5.3.2, when a response_type value is used that results in an Access Token being issued.

These scopes are very clearly not intended to include the [Claims] in the [ID Token]. In fact the [UserInfo Endpoint] is
meant to return them. The specification does not specifically prevent a [OpenID Connect 1.0] Provider from returning
them in the [ID Token], but it strongly suggests you shouldn't do this normally.

There's another crucial sentence directly after the first one.

> However, when no Access Token is issued (which is the case for the response_type value id_token), the resulting Claims
> are returned in the ID Token.

This seems to solidify this point. When the [Implicit Flow] is used, as the [Implicit Flow] is the only flow that does
not result in an [Access Token]; the [OpenID Connect 1.0] Provider should mint an [ID Token] that's populated with the
claims normally accessible at the [UserInfo Endpoint]. In fact it's only in one variation of the [Implicit Flow], when
the `response_type` parameter is only `id_token`. In this instance the [ID Token] is categorically required to be
populated with every one of the [Claims] the [Access Token] would normally be able to access at the [UserInfo Endpoint].

It's amazing how this neatly ties back into the use case for the [Access Token] at the very start of this article. It's
intended for use cases like accessing specific API's at the Authorization Server. If you then consider the fact the
[ID Token] is a static snapshot of the unique identity of a Resource Owner as described in the above concept
surrounding Claim Stability and Uniqueness, and then extrapolate that the request to the [UserInfo Endpoint] could
realistically have the most up-to-date information about the user; I feel like it all just makes complete sense.

The baffling question is if a Relying Party is not intending on using the [Access Token] to access the
[UserInfo Endpoint] why are they not using the [Implicit Flow] with the [Response Mode] `form_post` since this flow is
intended specifically to identify the user. Since they will not be issued an [Access Token], and only the [ID Token],
and it's performed over `form_post`, and the [ID Token] is signed and potentially encrypted multiple times; the typical
concerns around security of this flow are not relevant and it produces the output they desire.


### Claims Parameter

To cement the above point there is actually another means by which clients using **_any_** flow other than the
[Implicit Flow] which only returns an [ID Token] can obtain additional claims in the [ID Token].

This is done via the [Claims Parameter]. The [Claims Parameter] allows requesting specific claims be present in the
[ID Token]. In addition it has the added benefit of allowing granular requests for specific claims rather than granting
an entire [Scope] which may have many useless claims to the Relying Party.

This clearly has several impactful elements to security, privacy, and usability. This is also the parameter that is used
to indicate elements which are optional to consent to. I suspect most users have seen these dialogs which ask the user
what properties they want to allow the Relying Party to be able to access, even if they were not fully conscious of it.


[OpenID Connect 1.0]: https://openid.net/specs/openid-connect-core-1_0.html
[ID Token]: https://openid.net/specs/openid-connect-core-1_0.html#IDToken
[UserInfo Endpoint]: https://openid.net/specs/openid-connect-core-1_0.html#UserInfo
[Requesting Claims using Scope Values]: https://openid.net/specs/openid-connect-core-1_0.html#ScopeClaims
[Implicit Flow]: https://openid.net/specs/openid-connect-core-1_0.html#ImplicitFlowAuth
[Claims]: https://openid.net/specs/openid-connect-core-1_0.html#Claims
[Claims Parameter]: https://openid.net/specs/openid-connect-core-1_0.html#ClaimsParameter

[Access Token]: https://datatracker.ietf.org/doc/html/rfc6749#section-1.4
[Token Introspection]: https://datatracker.ietf.org/doc/html/rfc7662
[Token Revocation]: https://datatracker.ietf.org/doc/html/rfc7009

[Scope]: https://www.oauth.com/oauth2-servers/scope/defining-scopes/
[Audience]:

[JSON Web Token]: https://datatracker.ietf.org/doc/html/rfc7519
