# Encrypted PubNub Messaging for a Financial Trading Platform

## Problem/Feature Description

A fintech company runs a trading platform where brokers and clients communicate in real time. They use PubNub for three types of channels: public market commentary (no encryption needed), private broker-client conversations (moderate sensitivity), and trade execution confirmations (highly sensitive financial data). The compliance team has mandated that private and financial messages must be encrypted end-to-end so that even PubNub's infrastructure cannot read the message contents.

The engineering team needs a Node.js module that creates properly configured PubNub client instances for each security tier. The module should also include a utility to generate strong encryption keys, since the team has been using weak, manually-chosen strings as keys and the security audit flagged this as a risk.

## Output Specification

Create a file called `secure-messaging.js` containing:
- A function `generateCipherKey()` that produces a cryptographically strong key suitable for AES-256 encryption
- A function `createPublicClient(userId)` that returns a PubNub instance for unencrypted public channels
- A function `createPrivateClient(userId)` that returns a PubNub instance configured with encryption for private broker-client conversations, using a key passed as parameter
- A function `createFinancialClient(userId)` that returns a PubNub instance configured with encryption for trade execution channels, using a different key passed as parameter
- A comment block at the top of the file documenting which data types are and are not protected by the cipher key encryption

Also create a file called `key-setup.js` that demonstrates generating keys for each security tier and creating the three client types for a sample user.
