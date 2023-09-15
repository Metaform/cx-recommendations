# Modules and services of the IssuerService

## DID Module

Contains the `DidResourceManager`. Its job is to

- CRU(D) DIDs in the `DidResourceStore`
- publish/overwrite DIDs using the publishers
- reacts to events from the [KeyPair module](#keypair-module)
- reacts to manual action via some management API

## KeyPair Module

Contains the `KeyPairStateMachine`. Its job is to

- generate and maintain key pairs using a state machine
- checks for automatic renewal, e.g. if keys are configured with a max lifetime
- send out events when a key is rotated
- reacts to manual action via some management API


## VcIssuanceModule

Its job is to
- accept issuance requests coming in through the Issuance API
- create VerifiableCredentials based on `VcDefinitions`
- 