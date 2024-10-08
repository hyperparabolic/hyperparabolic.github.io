---
title: Microsoft Entra MFA EAM integration
date: 2024-10-08
author: Spencer Balogh
description: Thoughts and notes from integrating as an external multi-factor authentication method against Microsoft Entra.
tags:
  - Identity
  - OIDC
  - Microsoft
toc: true
---

Microsoft is starting to allow external identity providers to act as multi-factor authentication as a part of their authentication flows. This is currently documented as a preview feature [on their website](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-external-method-provider).

I worked with a client to implement support for this integration. This post has some thoughts, notes, surprises, and feedback from that experience.

<!--more-->

<mark>NOTE: Feature Preview.</mark> All of the notes from this integration are against a *preview* feature. This may not match the final deployment of the Microsoft feature.

## Leveraging OIDC

A lot of less frequently used OIDC features are in play here. What are they and why?

### On the implicit flow

Microsoft is using the OIDC implicit flow here, meaning that the `id_token` is being returned directly, rather than an authorization code that may be used to retrieve the token directly from the external IDP. Looking at it, `response_type=id_token` resembles the implicit grant (`response_type=token`) from the OAuth 2.0 spec. This was known to have [security issues](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics-29#name-implicit-grant) and was removed from the OAuth 2.1 spec. What's different here and why is this okay?

The issues documented with the implicit grant are resolved here by two different implementation details. Most of the browser specific issues are resolved with `response_mode=form_post` in the OIDC request. The `id_token` gets transmitted back to microsoft via a form `POST` response, and their OIDC requests are also made to the IDP with a POST method request. If implemented correctly there are never URI fragments containing the `id_token` exposed to the browser history, referrer headers, or elsewhere. Additionally, [nonce is required for implicit flow OIDC requests](https://openid.net/specs/openid-connect-core-1_0.html#ImplicitAuthRequest), and Microsoft also [requires nonce to be included in the id_token for the response](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-external-method-provider#microsoft-entra-id-processing-of-the-provider-response). This addition of a signed nonce can be used to prevent token replay.

### The claims parameter

The OIDC spec optionally allows requesting claims as a part of the OIDC request. This is an optional feature that not all IDPs implement, and support for this parameter is part of what was implemented as a part of this integration.

Microsoft is using this to specify authentication requirements on a request-by-request basis utilizing the `amr` and `acr` claims.

`amr`, or Authentication Methods References, indicates the authentication methods used as a part of the authentication. This claim consists of well known strings representing specific authentication methods, like `face` for facial biometrics, `hwk` for Proof of Possession of a hardware secured key, or `pwd` for password authentication.

`acr`, or Authentication Context Class Reference, is a claim with vendor specific values that indicate more generally what confidence levels are met by a specific authentication. [Microsoft's acr values](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-external-method-provider#supported-acr-claims) indicate whether `knowledge`, `possession`, or `inherence` values are required for a specific authentication. These are more flexible concepts. As an example, both fingerprint and facial recognition based biometrics both meet the `inherence` requirement.

This parameter allows Microsoft to make these claims required, and to require specific values for each claim. This allows them to extend their policy management to an external IDP.

### id_token_hint

This one is a bit more unusual. Microsoft is sending a Microsoft issued `id_token` in the `id_token_hint` parameter as a part of the OIDC request. They've specified [requirements for validating these tokens](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-external-method-provider#validating-tokens-issued-by-microsoft-entra-id). They additionally require that the subject of the `id_token` returned to them by the IDP matches the subject of the `id_token` sent with the request via `id_token_hint`.

This is acting as a form of request signing for the OIDC requests to the external IDP.

## Surprises

Like I said before, this is a feature preview. Hopefully some of these are just temporary pain points that will be ironed out before the final implementation.

### The `amr` claim only accepts 1 value

This was really just a minor annoyance, but it was surprising regardless.  When included in a token, `amr` and `acr` claims are specified as a list of values.  Microsoft is only allowing one `amr` or `acr` value to be specified in the `id_token`s returned to them.

I hope that this is a temporary restriction. The `amr` claim in particular is always going to contain multiple values if you're providing MFA. Just about anybody else who is requesting and consuming this claim is going to be interested in all of the included values to inform their risk management or compliance requirements.

When specifying `values` requirements as a part of the claims parameter, the spec says to treat the order of the values as an order of preference. This led to some surprisingly finicky vendor specific implementation, and this only seems to limit the extent that an external IDP can meet multiple MFA policy requirements in Azure.

### Not so standard `id_token_hint`

I definitely understand why [signed OIDC requests](https://openid.net/specs/openid-connect-core-1_0.html#SignedRequestObject) aren't being used here.  It's very underspecified as a part of the OIDC spec, and I wouldn't want to deal with integration mess from all of the "standard" implementations either. I am still ran into a few things that were surprising about their use of `id_token_hint`.

#### Standard `id_token_hint`

The OIDC spec defines `id_token_hint` as an optional parameter containing an `id_token` from the end user's current or previously authenticated session to use as a login hint, and primarily indicates it may be used to bypass OIDC permissions prompts.

[Authentication Request Validation](https://openid.net/specs/openid-connect-core-1_0.html#AuthRequestValidation) is explicit about instructions for how OpenID Providers must validate these tokens, and how they may be used as a part of establishing a session. In particular, the use of MUST rather than SHOULD or MAY:

> the OP MUST validate that it was the issuer of the ID Token.

Along with suggesting that expired tokens are fine for use in the `id_token_hint` parameter, and that the `aud` does not have to be the IDP (after all, the idenity token was issued to an external audience).

#### Integration implementation and differences

Microsoft specifies an alternate set of [required methods for validating these tokens](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-external-method-provider#validating-tokens-issued-by-microsoft-entra-id). They recommend implementing OIDC discovery to get the Microsoft Entra JWKS, and suggest a validation method that matches how a client would perform validation for an `id_token` issued by a IDP.  In particular, the token has to be signed by microsoft (rather than self issued / signed), timestamps must be enforced (as a means of expiring OIDC requests so they cannot be stored off for later use), and the `aud` claim should match the app ID for your application. Just about all of these requirements directly conflict with normal handling of `id_token_hint`.

I was also a little bit surprised when the token was issued to an Azure application audience, rather than being issued directly to the external IDP directly, or the external IDP's application client_id. The configuration in Azure didn't really click for me until I saw this. 

#### Preferences and Peeves

I think that in an ideal world, a vendor specific implementation that contradicts the spec isn't ideal, and in the realm of standards it should be avoided. In reality, you need to implement compatibility with whatever is the prevailing implementation as long as it doesn't sacrifice your security posture.

In this reality, I still can't help but look at and think of this conflicting use of the `id_token_hint` parameter the same way I look at and think of shadowing variables in programming. It isn't explicitly wrong, but I wouldn't do it in a security critical context. It opens up potential for bugs when new implementers in an existing codebase aren't aware that there are alternate requirements for a vendor specific implementation. I definitely would prefer an alternate parameter name, and that's very compatible with the open to extension nature of OAuth / OIDC ("Authorization Servers SHOULD ignore unrecognized request parameters").

### `nonce` is not bound to the OIDC request

I was definitely expecting the request nonce to be present in the `id_token_hint`. There isn't a nonce included there so it isn't cryptographically bound to the OIDC request.  I'm not sure if this is an oversight or just a result of how `id_token_hint` is issued, and general architecture.  I'm also not sure if this is of any real consequence alone, but it smells like the start of an obscure exploit when combined with other potential bugs.

## I'll take the combined package regardless

I hope that this comes off as more constructive than critical. I'm still very happy with a lot of the decisions Microsoft is making here. The idea of opening up their authentication processes and policy management to external implementers isn't a given, and I'm pleased that they're making any push in that direction. I'm also very happy that they're pushing toward adopting new technologies with OIDC. XML parsing and signing is **really** difficult to get right, and I think moving away from SAML is the right move.

This was still a delightful experience compared to most Microsoft integrations I've implemented in the past. Cheers to their team and I hope they keep improving this way.
