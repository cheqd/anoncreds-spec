## AnonCreds Setup Data Flow

The following sequence diagram summarizes the setup operations performed by a [[ref: Schema Publisher]], the [[ref: issuer]] (one required and one optional) in preparing to issue an AnonCred credential based on provided [[ref:schema]], and the one setup operation performed by each [[ref: holder]]. On successfully completing the operations, the [[ref: issuer]] is able to issue credentials based on the given [[ref:schema]] to the [[ref: holder]]. The subsections below the diagram detail each of these operations.

```mermaid
sequenceDiagram
    autonumber
    participant L as Verifiable<br>Data Registry
    participant SP as Schema Publisher
    participant I as Issuer
    participant H as Holder

    Note over L, H: Schema Publisher: Publish Schema

    SP ->> L: Publish Schema (Schema)
    L ->> I: Schema ID,<br>Schema Transaction ID

    Note over L, H: Issuer: Create, Store and Publish CredDef

    I ->> I: createAndStoreCredentialDef<br>(Schema, tag, supportRevocation)
    Note right of I:   store public / <br> private keys and<br>correctness proof
    I ->> L: Publish CredDef (credDef)

    Note over L, H: Issuer: Create, Store and Publish Revocation Registry (Optional)

    I ->> I: createAndStoreRevocReg (intCredDef)
    Note right of I: get keys
    Note right of I: store revocRegDef,<br>revocRegAccum,<br>privKey,<br>tailsGenerator
    I ->> L: Publish RevReg <br>(revocRegId,<br>revocRegDefJson,<br>revocRegEntryJson)

    Note over L, H: Holder: Create and Store Link Secret

    H ->> H: anoncredsProverCreateLinkSecret
    H ->> H: store link secret
    rect rgb(191, 223, 255)
      Note left of H: 💡The "Verifier" role is<br>omitted in this<br>diagram, since<br>it is not required<br>for the setup
    end
```

::: note

Those with a knowledge of DIDs might expect that in the flow above, the first
step would be for the [[ref: issuer]] to publish a DID. However, in AnonCreds,
DIDs are not used in the processing of credentials, and notably, the public keys
used in AnonCreds signatures come not from DIDs, but rather from [[def:
credDef]] objects. DIDs may be used to identify the entity publishing the
objects that are then used in the processing of credentials -- the [[def:
SCEHMA]], [[def: credDef]], [[def: revRegDef]] and [[def: revRegEntry]]
objects. There is an enforced relationship between an identifier (such as a DID)
for the entity publishing the AnonCred objects, and the objects themselves. For
example, in the Hyperledger Indy implementation of AnonCreds, for a credential
issuer to publish a [[def: credDef]] on an instance of Indy it must have a DID
on that instance, and it must use that DID to sign the transaction to write the
[[def: credDef]].

The DID of the publisher of an AnonCreds object MUST be identifiable from the
published object and enforcement of the relationship between the DID and the
object must be enforced. For example, in the Hyperledger Indy implementation of
AnonCreds, the DID of the object publisher is part of the identifier of the
object -- given the identifier for the AnonCreds object (e.g. one found in
proving a verifiable credential), the DID of the publisher can be found.
Further, the Hyperledger Indy ledger enforces, and makes available for
verification, the requirement that the writing of the AnonCreds object must be
signed by the DID that is writing the object.

If a DID-based messaging protocol, such as
[DIDComm](https://identity.foundation/didcomm-messaging/spec/) is used between
the AnonCreds participants (the [[ref: issuer]], [[ref: holder]] and [[ref:
verifier]]) the use of DIDs for messaging is independent of their use (or not)
in the publishing AnonCreds objects. Such DIDs are used to facilitate secure
messaging between the participants to enable the issuing of credentials and the
presentation of proofs.

:::

### Schema Publisher: Publish Schema Object

Each type of AnonCred credential is based on a [[ref:schema]] published to a Verifiable
Data Registry (VDR), an instance of Hyperledger Indy in this version of
AnonCreds. The [[ref:schema]] is defined and published by the [[ref: Schema Publisher]]. Any issuer
who can reference the [[ref:schema]] (including the [[ref: Schema Publisher]]) MAY issue
credentials of that type by creating and publishing a [[ref: credDef]] based on the
[[ref:schema]]. This part of the specification covers the operation to create and
publish a [[ref:schema]]. The flow of operations to publish a [[ref:schema]] is illustrated in
the `Schema Publisher: Publish Schema` section of the [AnonCreds Setup Data
Flow](#anoncreds-setup-data-flow) sequence diagram.

The [[ref:schema]] is a JSON structure that can be manually constructed,
containing the list of attributes (claims) that will be included in each
AnonCreds credential of this type. The following is an example [[ref: Schema]]:
```json
{
    "id": "https://www.did.example/schema.json",
    "name": "Example schema",
    "version": "0.0.1",
    "attrNames": ["name", "age", "vmax"]
}
```

* `id` - (string) The identifier of the [[ref: schema]]. The format of the identifier is dependent on the [[ref: AnonCreds Objects Method]] used in publishing the [[ref: schema]].
* `name` (string) - the name of the schema
* `version` (string) - the schema version
* `attrNames` (str[]) - an array of strings with each string being the name of an attribute of the schema

Once constructed, the [[ref: schema]] is published to a Verifiable Data Registry
(VDR) using the Schema Publishers selected [[ref: AnonCreds Objects Method]].
The `schemaId` identifier for the [[ref: schema]] is dependent on where the
[[ref: schema]] is published. For example, see [this
 Schema](https://indyscan.io/tx/SOVRIN_MAINNET/domain/73904) that is published on
the Sovrin MainNet instance of Hyperledger Indy. The `schemaId` for that object
is: `Y6LRXGU3ZCpm7yzjVRSaGu:2:BasicIdentity:1.0.0`

### Issuer Create and Publish CRED_DEF Object

Each Issuer of credentials of a given type (e.g. based on a specific [[ref: schema]]) must
create a [[ref: credDef]] for that credential type. The flow of operations to create and
publish a [[ref: credDef]] is illustrated in the `Issuer: Create, Store and Publish credDef`
section of the [AnonCreds Setup Data Flow](#anoncreds-setup-data-flow) sequence
diagram.

In AnonCreds, the [[ref: credDef]] and [[ref: credDef]] identifier include the following elements.

- A link to the Issuer of the credentials via the DID used to publish the
  [[ref: credDef]].
* A link to the [[ref: schema]] upon which the [[ref: credDef]] is based (the credential type).
* A set of public/private key pairs, one per attribute (claim) in the
  credential. The private keys will later be used to sign the claims when
  credentials to be issued are created.
- Other information necessary for the cryptographic signing of credentials.
- Information necessary for the revocation of credentials, if revocation is to
  be enabled by the Issuer for this type of credential.

We'll initially cover the generation and data for a [[ref: credDef]] created without the
option of revoking credentials. In the succeeding
[section](#generating-a-cred_def-with-revocation-enabled), we describe the
additions to the generation process and data structures when
credential revocation is enabled for a given [[ref: credDef]].

#### Retrieving the Schema Object

Prior to creating a [[ref: credDef]], the Issuer must get an instance of the
[[ref: schema]] upon which the [[ref: credDef]] will be created. If the Issuer
is also the [[ref: Schema Publisher]], they will already have the [[ref:
schema]]. If not, the Issuer must request that information from the [[ref: VDR]]
on which the [[ref:schema]] is published. In some [[ref: AnonCreds Objects
Methods]] there is a requirement that the [[ref: schema]] and [[ref: credDef]]
must be on the same [[ref: VDR]].

#### Generating a crefDef Without Revocation Support

The [[ref: credDef]] is a JSON structure that is generated using cryptographic primitives
(described below) given the following inputs.

* A [[ref: schema]] for the credential type.
* A `tag`, an arbitrary string defined by the Issuer, enabling an Issuer to
  create multiple [[ref: credDef]]s for the same [[ref: schema]].
* An optional flag `supportRevocation` (default `false`) which if true
  generates some additional data in the [[ref: CRED_DEF]] to enable credential
  revocation. The additional data generated when this flag is `true` is covered
  in the [next section](#issuer-create-and-publish-revocation-registry-object)
  of this document.

The operation produces two objects, as follows.

- The [[ref: privateCredDef]], an internally managed object that includes the private keys
  generated for the [[ref: credDef]] and stored securely by the issuer.
- The [[ref: credDef]], that includes the public keys generated for the [[ref:
credDef]], returned to the calling function and then published on a VDR
  (currently Hyperledger Indy).

The following describes the process for generating the [[ref: credDef]] and
[[ref: privateCredDef]] data.

::: todo
Describe the generation process for the credDef.
:::

The [[ref: privateCredDef]] produced by the generation process has the following format:

```json

To Do.

```

The [[ref: credDef]] has the following format (from [this example
credDef](https://indyscan.io/tx/SOVRIN_MAINNET/domain/99654) on the Sovrin
MainNet):

```json
{
  "data": {
    "primary": {
      "n": "779...397",
      "r": {
        "birthDate": "294...298",
        "birthLocation": "533...284",
        "citizenship": "894...102",
        "expiryDate": "650...011",
        "facePhoto": "870...274",
        "firstName": "656...226",
        "linkSecret": "521...922",
        "name": "410...200",
        "uuid": "226...757"
      },
      "rctxt": "774...977",
      "s": "750..893",
      "z": "632...005"
    }
  },
  "ref": 54177,
  "signatureType": "CL",
  "tag": "latest"
}
```

The [[ref: credDef]] contains a cryptographic public key that can be used to
verify CL-RSA signatures over a block of `L` messages `m1,m2,...,mL`. The [[ref:
credDef]] contains a public key fragment for each message being signed by
signatures generated with the respective private key. The length of the block of
messages, `L`, being signed is defined by referencing a specific Schema with a
certain number of attributes, `A = a1,a2,..` and setting `L` to `A+1`. The
additional message being signed as part of a credential is for a `linkSecret`
(called the [[ref: linkSecret]] everywhere except in the existing open source
code and data models) attribute which is included in all credentials. This value
is blindly contributed to the credential during issuance and used to bind the
issued credential to the entity to which it was issued.

All integers within the above [[ref: credDef]] example json are shown with ellipses (e.g. `123...789`). They are 2048-bit integers represented as `617` decimal digits. These integers belong to an RSA-2048 group characterised by the `n` defined in the [[ref: credDef]].

* `primary` is the data used for generating credentials.
* `n` is a safe RSA-2048 number. A large semiprime number such that `n = p.q`, where `p` and `q` are safe primes. A safe prime `p` is a prime number such that `p = 2p'+ 1`, where `p'` is also a prime. Note: `p` and `q` are the private key for the public CL-RSA key this [[ref: credDef]] represents.
* `r` is an object that defines a CL-RSA public key fragment for each attribute in the credential. Each fragment is a large number generated by computing `s^{xri}` where `xri` is a randomly selected integer between 2 and `p'q'-1`.
  * `masterSecret` (***deprecated*** - should be [[ref: link secret]]) is the name of an attribute that can be found in each [[ref: credDef]]. The associated private key is used for signing a blinded value given by the [[ref: holder]] to the [[ref: issuer]] during credential issuance, binding the credential to the [[ref: holder]].
  * The rest of the attributes in the list are those defined in the [[ref: schema]].
  * The attribute names are normalized (lower case, spaces removed) and listed in the [[ref: credDef]] in alphabetical order.
* `rctxt` is equal to `s^(xrctxt)`, where `xrctxt` is a randomly selected integer between `2` and `p'q'-1`. (I believe this is used for the commitment scheme, allowing entities to blindly contribute values to credentials.)
* `s` is a randomly selected quadratic residue of `n`. This makes up part of the CL-RSA public key, independent of the message blocks being signed.
* `z` is equal to `s^(xz)`, where `xz` is a randomly selected integer between `2` and `p'q'-1`. This makes up part of the CL-RSA public key, independent of the message blocks being signed.
* `ref` is the identifier of the schema. The format of the identifier is dependent on the [[ref: AnonCreds Objects Method]] used in publishing the [[ref: schema]].
* `signatureType` is always `CL` in this version of AnonCreds.
* `tag` is the `tag` value (a string) passed in by the [[ref: issuer]] to an AnonCred's [[ref: credDef]] create and store implementation.

::: todo
Evaluate the impact in the existing implementations of making the `ref`
an identifier vs. the current Hyperledger Indy Txn number -- an integer.
:::

The `credDefId` identifier for the [[ref: credDef]] is dependent on the [[ref:
AnonCreds Objects Method]] used in publishing the [[ref: schema]].

#### Generating a credDef With Revocation Support

The issuer enables the ability to revoke credentials produced from a [[ref: credDef]] by
passing to the [[ref: credDef]] generation process the flag `supportRevocation` as
`true`. When revocation is to enabled for a [[ref: credDef]], additional data related to
revocation is generated and added to the [[ref: credDef]] JSON objects defined above. In
the following the additional steps in the [[ref: credDef]] generation process to enable
revocation are described, along with the additional data produced in that
process.

The following describes the process for generating the revocation portion of the
[[ref: credDef]] data when the [[ref: credDef]] is created with the `supportRevocation` flag
set to `true`. This process extends the process for generating a [[ref: credDef]] in the
[previous section](#generating-a-creddef-without-revocation-support) of this document.

::: todo
Describe the revocation data generation process for the credDef.
Provide a reference to the published articles on revocation used here.
:::

A [[ref: privateCredDef]] with revocation enabled has the following format. In this, the
details of the `primary` element are hidden, as they are the same as was covered
above.

```json

To Do.

```

A [[ref: credDef]] with revocation enabled has the following format (from [this
example credDef](https://indyscan.io/tx/SOVRIN_MAINNET/domain/55204) on the
Sovrin MainNet). In this, the details of the `primary` element are hidden, as
they are the same as was covered above.

```json
{
  "data": {
    "primary": {...},
    "revocation": {
      "g": "1 154...813 1 11C...D0D 2 095..8A8",
      "gDash": "1 1F0...000",
      "h": "1 131...8A8",
      "h0": "1 1AF...8A8",
      "h1": "1 242...8A8",
      "h2": "1 072...8A8",
      "hCap": "1 196...000",
      "htilde": "1 1D5...8A8",
      "pk": "1 0E7...8A8",
      "u": "1 18E...000",
      "y": "1 068...000"
    }
  },
  "ref": 54753,
  "signatureType": "CL",
  "tag": "state_license"
}
```

The elements with ellipses (e.g. `1F0...000`) in `g` are 64 digits hex integers.
The rest of the elements are the same structure as `g` but containing either 3
or 6 hex integers, as noted below. In the following, only the `revocation` item
is described, as the rest of items (`primary`, `ref`, etc.) are described in the
previous section of this document.

- `revocation` is the data used for managing the revocation status of
  credentials issued using this [[ref: credDef]].
- `g` is the ...
- `gDash` is the ...
- `h` is the ...
- `h0` is the ...
- `h1` is the ...
- `h2` is the ...
- `hCap` is the ...
- `htilde` is the ...
- `pk` is the ...
- `u` is the ...
- `y` is the ...

#### Publishing the CRED_DEF on a Verifiable Data Registry

Once constructed, the [[ref: credDef]] is published by the Issuer to a [[ref:
Verifiable Data Registry]] using the issuers preferred [[ref: AnonCreds Objects
Method]]. For example, see [this
credDef](https://indyscan.io/tx/SOVRIN_MAINNET/domain/73905) that is published
in the Sovrin MainNet instance of Hyperledger Indy. The full contents of the
[[ref: credDef]] is placed in the ledger, including the revocation section if
present.

### Issuer Create and Publish Revocation Registry Objects

Once the [[ref: issuer]] has created a [[ref: credDef]] with revocation
enabled, the [[ref: issuer]] must also create and publish a [[ref: revRegDef]] and
create and publish the first [[ref: revRegEntry]] for the registry.

In this section, we'll cover the create and publish steps for each
of the [[ref: revRegDef]] and [[ref: revRegEntry]] objects. The creation and
publishing of the [[ref: revRegDef]] includes creating and publishing the
[[ref: tailsFile]] for the [[ref: regRev]].

#### Creating the Revocation Registry Object

A secure process must be run to create the revocation registry object, taking
the following input parameters.

- `type`: the type of revocation registry being created. For Hyperledger Indy
  this is always "CL_ACCUM."
- `credDefId`: the ID of the [[ref: credDef]] to which the [[ref: revReg]]
  is to be associated
- `tag`: an [[ref: issuer]]-defined tag that is included in the identifier for
  the [[ref: revReg]]
- `issuanceType`: an enumerated value that defines the initial state of
  credentials in the [[ref: revReg]]: revoked ("issuanceOnDemand") or
  non-revoked ("IssuanceByDefault"). See the [warning and recommendation
  against the use of
  `issuanceOnDemand`](#recommend-not-using-issuanceondemand).
- `maxCredNum`: The capacity of the [[ref: revReg]], a count of the number of
  credentials that can be issued using the [[ref: revReg]].
- `tailsLocation`: A URL indicating where the [[ref: tailsFile]] for the [[ref
revReg]] will be available to all [[ref: holders]] of credential issued using
  this revocation registry.

Three outputs are generated from the process to generate the [[ref; revReg]]:
the [[ref: revReg]] object itself, the [[ref: tailsFile]] content, and the
[[ref: privateRevReg]] object.

##### Recommend Not Using ISSUANCE_ON_DEMAND

::: warning

Based on the experience of the AnonCreds community in the use of revocable
credentials, it is highly recommended the `issuanceOnDemand` approach **NOT** be
used unless absolutely required by your use case.

The reason this approach is not recommended is that if the [[ref: issuer]]
creates the [[ref: revReg]] with the `issuanceType` item set to
`issuanceOnDemand`, the [[ref:issuer]] must publish a [[ref: RevRegEntry]] (as
described in the [revocation
section](#anoncreds-credential-activationrevocation-and-publication)) as each
credential is issued resulting in many [[ref: revRegEntry]] transactions being
performed, one per credential issued. Of course, with either `issueanceType`, there must
still be one transaction for each batch of (1 or more) credentials revoked.

Further, if a credential contains some kind of "Issue Date" attribute in the
credential and it is shared with verifiers, those verifiers can use that value
to find the [[ref: revRegEntry]] transaction that activated the credential at
the same time to learn the index of the holder's credential within the [[ref:
revReg]], giving the verifiers both a correlatable identifier (revRegId+index)
for the holder's credential and a way to monitor if that credential is ever
revoked in the future.

For these reasons we anticipate the deprecation or removal of the
`issuanceOnDemand` approach in the next version of AnonCreds specification.
Feedback from the community on this would be appreciated. We are particularly
interested in understanding what use cases there are for `issuanceOnDemand`.

:::

##### revRegDef Object Generation

The [[ref: revRegDef]] object has the following data model. This example is from
[this transaction](https://indyscan.io/tx/SOVRIN_MAINNET/domain/140386) on the
Sovrin MainNet and instance of Hyperledger Indy.

```json
{
  "credDefId": "Gs6cQcvrtWoZKsbBhD3dQJ:3:CL:140384:mctc",
  "id": "Gs6cQcvrtWoZKsbBhD3dQJ:4:Gs6cQcvrtWoZKsbBhD3dQJ:3:CL:140384:mctc:CL_ACCUM:1-1024",
  "revocDefType": "CL_ACCUM",
  "tag": "1-1024",
  "value": {
    "issuanceType": "ISSUANCE_BY_DEFAULT",
    "maxCredNum": 1024,
    "publicKeys": {
      "accumKey": {
        "z": "1 0BB...386"
      }
    },
    "tailsHash": "BrCqQS487HcdLeihGwnk65nWwavKYfrhSrMaUpYGvouH",
    "tailsLocation": "https://api.portal.streetcred.id/agent/tails/BrCqQS487HcdLeihGwnk65nWwavKYfrhSrMaUpYGvouH"
  }
}
```

The items within the data model are as follows:

::: todo
Update this to be the inputs for generating a REV_REG vs. the already published object
:::

- `credDefId`: the input parameter `credDefId`
- `id`: the identifier of the [[ref: revReg]]. The format of the identifier is dependent on the [[ref: AnonCreds Objects Method]] is to publish the object.
- `revocDefType`, the input parameter `type`
- `tag`, the input parameter `tag`
- `issuanceType`, the input parameter `issuanceType`
- `maxCredNum`, the input parameter `maxCredNum`
- `z`, a public key used to sign the accumulator (described further below)
- `tailsHash`, the calculated hash of the contents of the [[ref: tailsFile]],
  as described in the [next section](#tails-file-and-tails-file-generation) on
  [[ref: tailsFile]] generation.
- `tailsFileLocation`, the input parameter `tailsLocation`

As noted, most of the items come directly from the input parameters provided by
the [[ref: issuer]]. The `z` [[ref: revReg]] accumulator public key is
generated using (TODO: fill in details) algorithm. The use of the accumulator
public key is discussed in the Credential Issuance section, when the publication
of revocations is described. The calculation of the tailsHash is described in
the [next section](#tails-file-and-tails-file-generation) on [[ref: tailsFile]]
generation.

##### Tails File and Tails File Generation

The second of the outcomes from creating of a [[ref: revReg]] is a [[ref:
tailsFile]]. The contents of a [[ref: tailsFile]] is an array of calculated
prime integers, one for each credential in the registry. Thus, if the [[ref:
revReg]] has a capacity (`maxCredNum`) of 1000, the [[ref: tailsFile]] holds
an array of 1000 primes. Each credential issued using the [[ref: revReg]] is
given its own index (1 to the capacity of the [[ref: revReg]]) into the array,
the index of the prime for that credential. The contents of the [[ref;
tailsFile]] is needed by the [[ref: issuer]] to publish the current state of
revocations within the [[ref: revReg]] and by the [[ref: holder]] to produce
(if possible) a "proof of non-revocation" to show their issued credential has
not been revoked.

The process of generating the primes that populate the [[ref: tailsFile]] is as
follows:

::: todo
To Do: Document hashing of the tails file ([see also](https://github.com/hyperledger/indy-shared-rs/blob/d22373265f7c4cf93d59dd3c111251ef96d6a63d/indy-credx/src/services/tails.rs#L151)).
:::

::: todo
To Do: Document the process for generating the primes.
:::

Once generated, the array of primes is static, regardless of credential issuance
or revocation events. Once generated, the SHA256 (TO BE VERIFIED) hash of the
array of primes is calculated and returned to be inserted into the `tailsHash`
item of the [[ref: revReg]] object (as described in the [previous
section](#rev_reg_def-object-generation)). Typically, the array is streamed into a
file (hence, the term "Tails File") and published to a [[def: URL]] indicated by
the `tailsLocation` input parameter provided by the [[ref: issuer]].

The format of a [[ref: tailsFile]] is as follows:

::: todo
To Do: Define the format of the Tails File
:::

While not required, the Hyperledger Indy community has created a component, the "[Indy Tails
Server](https://github.com/bcgov/indy-tails-server)," which is basically a web
server for tails files. [[ref: holders]] get the `tailsLocation` during the
issuance process, download the [[ref: tailsFile]] (ideally) once and cache it
for use when generating proofs of non-revocation when creating a presentation
that uses its revocable verifiable credential. How the [[ref: tailsFile]] is
used is covered elsewhere in this specification:

- in the section about the [[ref: issuer]] publishing credential revocation
  state updates, and
- in the section about [[ref: holders]] creating a proof of non-revocation.

##### privateRevReg Object Generation

In addition to generating the [[ref: revReg]] object, a [[ref:
privateRevReg]] object is generated and securely stored by the [[ref:
issuer]]. The data model and definition of the items in the [[ref:
privateRevReg]] is as follows:

::: todo
To Do: Fill in the details about the privateRevReg
:::

#### Publishing the Revocation Registry Object

Once constructed, the [[ref: revReg]] is published by the [[ref: issuer]] in a
[[ref: Verifiable Data Registry]] using the issuer's [[ref: AnonCreds Objects
Method]]. For example, see [this
REV_REG](https://indyscan.io/tx/SOVRIN_MAINNET/domain/140386) that is published
on the Sovrin MainNet instance of Hyperledger Indy. The binary [[ref:
tailsFile]] associated with the [[ref: revReg]] can be downloaded from the
`tailsLocation` in the [[ref: revReg]] object.

#### Creating the Initial Revocation Registry Entry Object

Published [[ref: revRegEntry]] objects contain the state of the [[ref:
revReg]] at a given point in time such that [[ref: holders]] can generate a
proof of non-revocation (or not) about their specific credential and [[ref:
verifiers]] can verify that proof. An initial [[ref: revRegEntry]] is
generated and published immediately on creation of the [[ref: revReg]] so that
it can be used immediately by [[ref: holders]]. Over time, additional [[ref:
revRegEntry]] objects are generated and published as the revocation status of
one or more credentials within the [[ref: revReg]] change.

A secure process must be run to create the initial [[ref: revRegEntry]] object,
taking the following input parameters.

- `type`: the type of revocation registry being created. For Hyperledger Indy
  this is always "CL_ACCUM."
- `revRegId`: the ID of the [[ref: revReg]] for which the initial [[ref:
revRegEntry]] is to be generated.
  - The process uses this identifier to find the associated [[ref:
privateRevReg]] to access the information within that object.

The process collects from the identified [[ref: privateRevReg]] information to
calculate the cryptographic accumulator value for the initial [[ref:
revRegEntry]], including:

- `issuanceType`: an enumerated value that defines the initial state of
  credentials in the [[ref: revReg]]: revoked ("issuanceOnDemand") or
  non-revoked ("issuanceByDefault"). See the [warning and recommendation
  against the use of
  `issuanceOnDemand`](#recommend-not-using-issuanceondemand).
- `maxCredNum`: The capacity of the [[ref: revReg]], a count of the number of
  credentials that can be issued using the [[ref: revReg]].
- `tailsArray`: The contents of the [[ref: tailsFile]], the array of primes,
  one for each credential to be issued from the [[ref: revReg]].
- `privateKey`: The accumulator private key for the [[ref: revReg]].

With the collected information, the process the initial cryptographic
accumulator for the [[ref: revReg]]. The format of the identifier for the
[[ref: revRegEntry]] is dependent on the [[ref: AnonCreds Objects Method]]
used by the issuer.

In simple terms, the cryptographic accumulator at any given point in time is the
(modulo) product of the primes for each non-revoked credential in the [[ref:
revReg]]. Based on the value of `issuanceType`, all of the
credentials are initially either revoked, or unrevoked. If all of the credentials are
initially revoked, the accumulator value is `0`, if all are unrevoked, the
accumulator value has contributions from all of the entries in the array of
primes.

The accumulator is calculated using the following steps:

::: todo
To Do: Adding the algorithm for calculating the accumulator
:::

THe following is an example of an initial, published [[ref: revRegEntry]] object:

```json
{
  "revocDefType": "CL_ACCUM",
  "revocRegDefId": "Gs6cQcvrtWoZKsbBhD3dQJ:4:Gs6cQcvrtWoZKsbBhD3dQJ:3:CL:140389:mctc:CL_ACCUM:1-1024",
  "value": {
    "accum": "21 10B...33D"
  }
}
```

The items in the data model are:

- `revocDefType`: the input parameter `type`
- `revocRegDefId`: the identifier of the associated [[ref: revRegDef]]. The
  format of the identifier is dependent on the [[ref: AnonCreds Objects Method]]
  used by the issuer.
- `accum`: the calculated cryptographic accumulator reflecting the initial state
  of the [[ref: revReg]]

To see what [[ref: revRegEntry]] transactions look like on a [[ref: VDR]], this is [a
link](https://indyscan.io/tx/SOVRIN_MAINNET/domain/55326) to an initial [[ref:
revRegEntry]] where the credentials are initially all revoked, while this is
[a link](https://indyscan.io/tx/SOVRIN_MAINNET/domain/140392) to an initial
[[ref: revRegEntry]] where all of the credentials are unrevoked.

#### Publishing the Initial Initial Revocation Registry Entry Object

Once constructed, the initial [[ref: revRegEntry]] is published by the [[ref:
issuer]] in a [[ref: Verifiable Data Registry]] using their selected [[ref:
AnonCreds Objects Method]]. For example, see [this
revRegEntry](https://indyscan.io/tx/SOVRIN_MAINNET/domain/140392) that is
published on the Sovrin MainNet instance of Hyperledger Indy.

### Holder Create and Store Link Secret

To prepare to use AnonCreds credentials, the [[ref: holder]] must create a
[[ref: link secret]], a unique identifier that allows credentials issued to a
[[ref: holder]] to be bound to that [[ref: holder]] and presented without
revealing a unique identifier, thus avoiding correlation of credentials by
[[ref: verifier]]s. The [[ref: linkSecret]] is kept private by the [[ref:
holder]]. The [[ref: link secret]] is used during the credential issuance
process to bind the credential to the [[ref: holder]] and in the generation of a
presentation. For the latter, it allows the [[ref: holder]] to create a zero
knowledge proof that they were issued the credential by demonstrating knowledge
of the value of the [[ref: linkSecret]] without sharing it. The details of how
the [[ref: linkSecret]] is used to do this is provided in the issuance,
presentation generation and verification sections of this specification.

The [[ref: link secret]] is a sufficiently random unique identifier. For
example, in the Hyperledger Indy implementation, the [[ref: link secret]] is
produced by a call to the Rust
[uuid](https://docs.rs/uuid/0.5.1/uuid/index.html) Crate's `new_v4()` method to
achieve sufficient randomness.

Once generated, the [[ref: linkSecret]] is stored locally by the [[ref:
holder]] for use in subsequent issuance and presentation interactions. If lost,
the [[ref: holder]] will not be able to generate a proof that the credential was
issued to them. The [[ref: holder]] generates only a single [[ref:
linkSecret]], using it for all credentials the [[ref: holder]] is issued. This
allows for [[ref: verifier]]s to verify that all of the credentials used in
generating a presentation with attributes from multiple credentials were all
issued to the same [[ref: holder]] without requiring the [[ref: holder]] to
disclose the unique identifier ([[ref: linkSecret]]) that binds these
credentials together.

There is nothing to stop a [[ref: holder]] from generating multiple [[ref:
linkSecret]]s and contributing them to different credential issuance processes.
However, doing so prevents the [[ref: holder]] from producing a presentation
combining credentials issued to distinct [[ref: linkSecret]]s that can be
proven to have been issued to the same entity. It is up to the [[ref: verifier]]
to require and enforce the binding between multiple credentials used in a
presentation.
