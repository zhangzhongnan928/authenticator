# Dauth, an Oauth-based login protocol

## Overview

Dauth (decentralised authentication) provides Web2 developers with a login and authentication method based on Oauth and the password-less Ethereum cryptography. It will use the same Oauth API formats and similar communication flow that Web2 developers are already familiar with, as well as its key management principles. This method is superior to logging in through signing a message with an Ethereum key because of the following reasons.

**1. Privacy**: Users can *either* a) login **anonymously** *or* b) bind the login session with an Ethereum Address / ENS name. If the user logs in anonymously, the website does not learn the user's Ethereum address. And during the session, if the user chooses to identify themselves with Eth Address or ENS name, they can do so without changing the session or authentication token.

**2. Security**: The user's Ethereum key is not directly used. Instead, a delegated key is generated for each session and not shared between websites. This 1) prevents man-in-the-middle attack, where the attacking website forwards the login challenge from another site to itself; 2) allows the delegated key to be revoked, so the user can stolen devices to stay in the login session; 3)  by using one key per site, if delegated auth keys are compromised, its originating Ethereum key is not and safety of crypto assets are not affected.

**3. User Experience**: The user, having logged in, can decide how long the session remains active (e.g. at home, longer sessions are allowed). The user doesn't need to do anything on the UI level, even if the website session times out and requires re-authentication; the authentication happens silently. This prevents modern-day awkwardness such as "session expired, please log in again". (The website might disable such behavior from their side.)

**4. Decentralisation**: No other servers are involved when a webserver allows users to log in using ETH Address. However if ENS is used, a node is involved.

## High-level Mechanism

![Communication Flow Chart](compared_with_oauth.svg "Compare DAuth with OAuth")

Terms:

Server
: In Dauth only one web server is involved, which is the web server of the website the user logs in into. Hence, 'server' always refer to the website's server.

Client
: The Dauth library at work in the client (browser) side. For wallets that support native Dauth, this code works in the wallet; for wallets without Dauth support, JavaScript library code works in the browser, with access to the user's wallet.

### Login

When a user logs in to a website, a webserver creates a challenge. A *refresh key pair* is created from an Ethereum address on the client-side, using the website's domain name along with some high entropy randomness. The method to obtain such a key is discussed later. In this version of the draft, the randomness comes from the web server's challenge. This key expires in a server-specified amount of time, for example, one week.

The client then creates and sends a response to the challenge with the following information:

- Refresh key's public key, together with proof that the key is generated from an Ethereum address (this address does not have to exist in the Ethereum blockchain, see the "anonymous login" section later) in response to the challenge.

- A signature using that key, proving to the server that the client has the private key of the refresh key. 

This information unambiguously associates the user's session with the said Ethereum address.

Optionally, if an ENS name is provided, it is sent to the server in the same challenge-response. This binds the session to the ENS name instead of the Ethereum address, allowing the user to change the Ethereum address later. Any relevant account metadata is provided at this opportunity, such as attestation from 3rd party - e.g. this Ethereum address is held by a recent Olympic Games Tokyo athlete.

### Session Management

Whenever an HTTP request is sent to the server, it comes with a JSON authentication token, which contains an 'access token', which is the signature of the timestamp in the HTTP request.

The client should keep the refresh key to provide a different access token each time the user needs to access a resource. 

For clients with a Dauth compatible wallet, the refresh key is kept in the wallet until it expires. The wallet's policy decides whether or not the user is still the same user, and the key should be kept in use. For example, suppose a user enabled a mobile phone with fingerprint authentication or face-recognition authentication. In that case, the wallet knows when it's the same user, so it should use the same refresh key without the user logging in again. This is an improvement over today's Oauth based solutions.

For clients without a Dauth compatible wallet, a JavaScript library provided to the web developer will be at work to determine if the key should still be kept or discarded, based on the website's policy. This is a similar situation to today's Oauth based website.

### Anonymous authentication

This protocol supports anonymous authentication and adaptive identification. In plain words:

- A user can sign up as an anonymous user by not linking its identity with any Ethereum address at use. The user will still be identified by a seemingly random Ethereum address, except that this address will never hold any Ether or be used for smart contracts. 
- However, this seemingly random Ethereum address is determined by an actual Ethereum address which may hold assets, tokens or ENS names. The user can prove to the website that his anonymous identity is directly derivable from his real Ethereum address and tokens at a certain point in time.

This provides adaptive identification. For example, a user who visits an e-commerce website to buy some products would, at first, not want to be identified but still desires convenient login based on their Ethereum key. Later, when the user starts to be interested in community participation, such as leaving feedback on their purchase, they would want to be identified. They can submit their Ethereum address and cryptographically prove that they have always been the owner of that Ethereum address. The anonymous identity was constructed based on this, for precisely this website.

## Work plan

- The refresh key generation method is a cryptographic protocol that has to be developed and scrutinised. Most simply, the key is either derived from the user's Ethereum address using the website's domain name and the challenge as a derivation factor or ccreated from the signature on such a derivation factor. In any way, the security of it has to be described separately and proved.
- The communication protocol, largely based on Oauth, needs to be defined and tested with web2 developers to determine if it is easy to adopt.
- A wallet extension needs to be defined and implemented in one of the opensource Ethereum wallets. This extension is not required for Dauth to work, but it improves security and user experience.
- A JavaScript package needs to be developed for web2.0 developers to adopt Dauth similarly to Facebook/Google's Oauth 2.0.
- A backend package that can verify the signature, as well as the refresh key's public key, is correctly derivated. Such a package may need to be in both JavaScript and PHP.

## FAQ

### How is Oauth used in this protocol

- Oath's concept of refresh token and access token to secure sessions is used in this protocol. We modified the refresh token to an asymmetric key pair, whose public key is used in place of Oauth's refresh token.
- We re-use the concept of "magic links" to forward authentication tokens in Oauth's native format. Although forwarding tokens through links is not technically needed, since there is no centralised authentication provider, the look-and-feel of it is retained to make Web2 developers comfortable.
- The communication flow of Oauth has authentication delegated to the centralised authentication provider, and the token is passed back to the website. Such a flow is mirrored by replacing such a provider with a wallet.

Together, this creates a decentralised login process that has Oauth's look and feel, a flow mirroring Oauth and a data set based on Oauth's native data format in a trustless, decentralised environment where a confidential 3rd party authentication provider does not exist.

### How is "silent login" achieved

The goal is to build session management in the protocol so that the user does not get kicked out for not being active on a web page for a few minutes. This is achieved by having a long-lived (1 to 2 weeks) refresh key that stays with the browser. As long as the environment remains secure, the browser should know the key is kept alive and keep producing access tokens for accessing resources.

### What if the user's device is compromised

It's assumed that a compromised device leads to compromised sessions, but not a compromised wallet. If a user's mobile phone is lost, the user can restore their wallet on another device, log in to the same website and choose to disable any other session. This expires the *refresh key* from the server-side; therefore any other active session users will be 'kicked out', and need to create a new refresh key. Since creating a new refresh key depends on accessing the Ethereum address' private key, the person who stole the device can't resume the session.

### Is the underlying cryptography sound?

It is crucial that the *refresh key pair* is created securely. The exact algorithm is to be developed, with minimal models already existing pending further proof. An earlier algorithm is described in [an earlier version of this draft](Dauth-without-session-mgmt.md). The team working on this is the same set of people who worked on the zk identifier attestation protocol to be used in the Devcon ticket attestations and various other cryptographic endeavours. 
