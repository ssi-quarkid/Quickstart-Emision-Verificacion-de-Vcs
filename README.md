# Quickstart

The objective of this Quickstart is to record the steps to perform a process of issuing and verifying credentials.

## Issuance and Verification (Code Format)

1. [Creation, publication, and resolution of DID](#install-and-create-your-did).
2. [Creation of Verifiable Credential, hereinafter VC](#creation-of-a-vc-verifiable-credential).
3. [Verification of a VC](#verification-of-a-vc).

## Issuance (QR Format)

1. [Creation and reading of QR code for Test Credential](#generation-of-test-credential-with-qr-code)

---

## Install and create your DID

_Prerequisites_

You must have installed:

```
NODEJS
NPM
TYPESCRIPT
TS-NODE
```

### 1. Create a directory

```
mkdir quarkid-app
```

```
cd quarkid-app
```

### 2. Install NPMJs packages

```
npm init
```

```
npm install @quarkid/did-registry @quarkid/did-core @quarkid/kms-client @quarkid/kms-core
```

### 3. Create the keys

To create a DID, you need to provide your public keys. You can use the KMS component to generate your public and private keys.

_Key Management Service_: KMS Storage and KMS Keys

## KMS Storage

KMS applies dependency inversion concepts, so it requires sending a SecureStorage in its constructor. On this occasion, we will use FileSystem to store the data and access it in a simple way.

### 1. Create a storage.ts file with the following code

```typescript
import { readFileSync, writeFileSync, existsSync } from "fs";
import { KMSStorage } from "@quarkid/kms-core";

export class FileSystemKMSSecureStorage implements KMSStorage {
  public readonly filepath: string;

  constructor(params: { filepath: string }) {
    this.filepath = params.filepath;
  }

  async add(key: string, data: any): Promise<void> {
    const map = this.getData();
    map.set(key, data);
    this.saveData(map);
  }

  async get(key: string): Promise<any> {
    return this.getData().get(key);
  }

  async getAll(): Promise<Map<string, any>> {
    return this.getData();
  }

  update(key: string, data: any) {
    const map = this.getData();
    map.set(key, data);
    this.saveData(map);
  }

  remove(key: string) {
    const map = this.getData();
    map.delete(key);
    this.saveData(map);
  }

  private getData(): Map<string, any> {
    if (!existsSync(this.filepath)) {
      return new Map();
    }

    const file = readFileSync(this.filepath, {
      encoding: "utf-8",
    });

    if (!file) {
      return new Map();
    }

    return new Map(Object.entries(JSON.parse(file)));
  }

  private saveData(data: Map<string, any>) {
    writeFileSync(this.filepath, JSON.stringify(Object.fromEntries(data)), {
      encoding: "utf-8",
    });
  }
}
```

## KMS Keys

### 1. Create a did.ts file.

### 2. Import the following dependencies:

```typescript
import { FileSystemKMSSecureStorage } from "./storage";
import { KMSClient } from "@quarkid/kms-client";
import { LANG, Suite } from "@quarkid/kms-core";
```

### 3. Create a function called createDID as shown below:

Create the keys _recoveryKey, updateKey, bbsBlsJwk, and didCommJwk_.

```typescript
export const createDID = async () => {
  const kms = new KMSClient({
    lang: LANG.en,
    storage: new FileSystemKMSSecureStorage({
      filepath: "file-system-storage",
    }),
  });

  const updateKey = await kms.create(Suite.ES256k);
  const recoveryKey = await kms.create(Suite.ES256k);

  const didComm = await kms.create(Suite.DIDComm);
  const bbsbls = await kms.create(Suite.Bbsbls2020);
};
```

> [!TIP]
> For more information about KMS, see the [available documentation](https://www.npmjs.com/package/@quarkid/kms-client).

---

### 4. Create a long DID

Once you have generated your public and private keys, the next step is to create a LONG DID.

With the DID creation service, you can create a LONG DID, which is a DID where its DID Document is embedded in the information it returns in a base64 format. It is a self-resolvable DID, meaning that by decoding the base64, you can obtain its DID Document.

In the did.ts file, import the dependencies and add the code shown below.

```typescript
import { ModenaUniversalRegistry } from "@quarkid/did-registry";
import { AssertionMethodPurpose, KeyAgreementPurpose } from "@quarkid/did-core";
```

```typescript
export const createDID = async () => {
  const kms = new KMSClient({
    lang: LANG.en,
    storage: new FileSystemKMSSecureStorage({
      filepath: "file-system-storage",
    }),
  });

  const updateKey = await kms.create(Suite.ES256k);
  const recoveryKey = await kms.create(Suite.ES256k);

  const didComm = await kms.create(Suite.DIDComm);
  const bbsbls = await kms.create(Suite.Bbsbls2020);

  const registry = new ModenaUniversalRegistry();

  const createDidResponse = await registry.createDID({
    updateKeys: [updateKey.publicKeyJWK],
    recoveryKeys: [recoveryKey.publicKeyJWK],
    verificationMethods: [
      {
        id: "bbsbls",
        type: "Bls12381G1Key2020",
        publicKeyJwk: bbsbls.publicKeyJWK,
        purpose: [new AssertionMethodPurpose()],
      },
      {
        id: "didComm",
        type: "X25519KeyAgreementKey2019",
        publicKeyJwk: didComm.publicKeyJWK,
        purpose: [new KeyAgreementPurpose()],
      },
    ],
  });
  console.log(JSON.stringify(createDidResponse.longDid));
};
```

Continue with the next step before executing your code:

### 5. DID Publication

Publish your DID on the blockchain. This requires a QuarkID API URL, which represents a QuarkID node running as a service.
You can [provide your own node](https://github.com/ssi-quarkid/Nodo-QuickStart/) or use the following:

### Node:

```typescript
export const QuarkIDEndpoint = "https://node-ssi.buenosaires.gob.ar";
```

### 6. Write the following code within the createDID function to publish.

```typescript
const result = await registry.publishDID({
  universalResolverURL: QuarkIDEndpoint,
  didMethod: "did:quarkid",
  createDIDResponse: createDidResponse,
});

console.log("result", result);
```

The result constant will return a canonicalId, a short DID, and a long DID that will allow you to later resolve it and obtain the DID Document.

Continue with the next step before executing your code.

### 7. Resolve

To resolve your DID, install the following package:

```
npm install @quarkid/did-resolver
```

Within the did.ts file, import the following dependency:

```typescript
import { DIDUniversalResolver } from "@quarkid/did-resolver";
import { DIDDocument } from "@quarkid/did-core";
```

Write the following function before the createDID function.

```typescript
export const resolveDid = (did: string) =>
  new Promise<DIDDocument>((resolve, reject) => {
    setTimeout(async () => {
      const universalResolver = new DIDUniversalResolver({
        universalResolverURL: QuarkIDEndpoint,
      });

      const didDocument = await universalResolver.resolveDID(did);
      console.log(didDocument);

      return didDocument;
    }, 65000);
  });
```

Add the following code to the createDID function:

```typescript
const didDocument = await resolveDid(result.did);
console.log("Did Document", didDocument);
```

### 8. Test your DID

To test your code, you'll need to call the createDID function.

```typescript
createDID();
```

If you followed the steps correctly, you should have a did.ts file like the following:

```typescript
import { FileSystemKMSSecureStorage } from "./storage";
import { KMSClient } from "@quarkid/kms-client";
import { LANG, Suite } from "@quarkid/kms-core";
import { ModenaUniversalRegistry } from "@quarkid/did-registry";
import { AssertionMethodPurpose, KeyAgreementPurpose } from "@quarkid/did-core";
import { DIDUniversalResolver } from "@quarkid/did-resolver";
import { DIDDocument } from "@quarkid/did-core";

export const QuarkIDEndpoint = "https://node-ssi.buenosaires.gob.ar";

export const resolveDid = (did: string) =>
  new Promise<DIDDocument>((resolve, reject) => {
    setTimeout(async () => {
      const universalResolver = new DIDUniversalResolver({
        universalResolverURL: QuarkIDEndpoint,
      });

      const didDocument = await universalResolver.resolveDID(did);

      return didDocument;
    }, 65000);
  });

export const createDID = async () => {
  const kms = new KMSClient({
    lang: LANG.en,
    storage: new FileSystemKMSSecureStorage({
      filepath: "file-system-storage",
    }),
  });

  const updateKey = await kms.create(Suite.ES256k);
  const recoveryKey = await kms.create(Suite.ES256k);

  const didComm = await kms.create(Suite.DIDComm);
  const bbsbls = await kms.create(Suite.Bbsbls2020);

  const registry = new ModenaUniversalRegistry();

  const createDidResponse = await registry.createDID({
    updateKeys: [updateKey.publicKeyJWK],
    recoveryKeys: [recoveryKey.publicKeyJWK],
    verificationMethods: [
      {
        id: "bbsbls",
        type: "Bls12381G1Key2020",
        publicKeyJwk: bbsbls.publicKeyJWK,
        purpose: [new AssertionMethodPurpose()],
      },
      {
        id: "didComm",
        type: "X25519KeyAgreementKey2019",
        publicKeyJwk: didComm.publicKeyJWK,
        purpose: [new KeyAgreementPurpose()],
      },
    ],
  });

  console.log(JSON.stringify(createDidResponse.longDid));

  const result = await registry.publishDID({
    universalResolverURL: QuarkIDEndpoint,
    didMethod: "did:quarkid",
    createDIDResponse: createDidResponse,
  });

  console.log("result", result);
  const didDocument = await resolveDid(result.did);
  console.log("Did Document", didDocument);
};

createDID();
```

The next step is to execute the createDID function. To do this, open a terminal and use ts-node or npx to run your code.

```
ts-node did.ts
```

```
npx ts-node did.ts
```

> [!NOTE]
> In your terminal, you will be able to observe the result of the console.log statements in your code. The resolution of the DID Document takes a few seconds to impact, as the DID is being published on the blockchain and this process can take time.

---

## Creation of a VC (Verifiable Credential)

The format of a VC consists of a JSON Object that contains identifiers and metadata to describe the properties of the credential, such as its Issuer, validity period, representative image, a public key to use for verification purposes, revocation mechanisms, among others. The metadata can be signed by the issuer, so its authenticity can be proven through cryptographic verification.
Verifiable credentials and verifiable presentations MUST include an @context property.

In the particular case of QuarkID, the URLs provided in this section must be processable as JSON-LD.

JSON-LD, which stands for "JSON Linked Data", is a way of representing structured data using JSON syntax while incorporating Linked Data concepts to allow semantic interoperability on the web.

### 1. Install

```
npm install @quarkid/vc-core
```

### 2. Create a credential

To create your first credential, you must have created a DID and its keys stored in the KMS storage.
With that task completed, you can proceed. Create a credential.ts file and import the following dependency.

```typescript
import { VerifiableCredentialService } from "@quarkid/vc-core";
```

### 3. Write the following code and replace what's in ISSUER with your DID:

```typescript
const credential = async () => {
  const vcService = new VerifiableCredentialService();

  const credential = await vcService.createCredential({
    context: [
      "https://w3id.org/vaccination/v1",
      "https://w3id.org/security/v2",
      "https://w3id.org/security/bbs/v1",
    ],
    vcInfo: {
      issuer: "REPLACE WITH YOUR DID",
      expirationDate: new Date("2026/05/05"),
      id: "123456789",
      types: ["VaccinationCertificate"],
    },
    data: {
      type: "VaccinationEvent",
      batchNumber: "1183738569",
      administeringCentre: "MoH",
      healthProfessional: "MoH",
      countryOfVaccination: "NZ",
      recipient: {
        type: "VaccineRecipient",
        givenName: "JOHN",
        familyName: "SMITH",
        gender: "Male",
        birthDate: "1958-07-17",
      },
      vaccine: {
        type: "Vaccine",
        disease: "COVID-19",
        atcCode: "J07BX03",
        medicinalProductName: "COVID-19 Vaccine Moderna",
        marketingAuthorizationHolder: "Moderna Biotech",
      },
    },
    mappingRules: null,
  });

  console.log("Credential", credential);
};
```

Continue with the next step before executing your code.

### 4. Sign your credential

To be able to sign your credential, you must instantiate the KMS to request the keys with which you will sign your credential.

Import the following dependencies in the credential.ts file.

```typescript
import { KMSClient } from "@quarkid/kms-client";
import { LANG, Suite } from "@quarkid/kms-core";
import { AssertionMethodPurpose } from "@quarkid/did-core";
import { FileSystemKMSSecureStorage } from "./storage";
```

Add the following code to the credential function after what you've already written.

```typescript
const kms = new KMSClient({
  lang: LANG.es,
  storage: new FileSystemKMSSecureStorage({
    filepath: "file-system-storage",
  }),
});

const bbsbls2020 = await kms.getPublicKeysBySuiteType(Suite.Bbsbls2020);
```

We use the bbsbls2020 key algorithm to sign a credential as it allows us to use selective disclosure and zero-knowledge proof.

The KMS provides a signVC method that allows us to sign a credential.

Add the following code to the credential function, after what you've written previously, but replace the string "YOUR DID" with the DID you created in the previous steps.

```typescript
const vc = await kms.signVC(
  Suite.Bbsbls2020,
  bbsbls2020[0],
  credential,
  "YOUR DID",
  "YOUR DID" + "#bbsbls",
  new AssertionMethodPurpose()
);
console.log("Verifiable Credential Signed", vc);
```

### 5. Test your code

To test your code, you'll need to call the credential function.

```typescript
credential();
```

If you followed the steps correctly, you should have a credential.ts file like the following:

```typescript
import { VerifiableCredentialService } from "@quarkid/vc-core";
import { KMSClient } from "@quarkid/kms-client";
import { LANG, Suite } from "@quarkid/kms-core";
import { AssertionMethodPurpose } from "@quarkid/did-core";
import { FileSystemKMSSecureStorage } from "./storage";

const credential = async () => {
  const vcService = new VerifiableCredentialService();

  const credential = await vcService.createCredential({
    context: [
      "https://w3id.org/vaccination/v1",
      "https://w3id.org/security/v2",
      "https://w3id.org/security/bbs/v1",
    ],
    vcInfo: {
      issuer: "did:quarkid:EiBA3ihJrI5fSsdpZWd3H_-0Wr4rEL8muoDOsuDQDhe2FQ",
      expirationDate: new Date("2026/05/05"),
      id: "123456789",
      types: ["VaccinationCertificate"],
    },
    data: {
      type: "VaccinationEvent",
      batchNumber: "1183738569",
      administeringCentre: "MoH",
      healthProfessional: "MoH",
      countryOfVaccination: "NZ",
      recipient: {
        type: "VaccineRecipient",
        givenName: "JOHN",
        familyName: "SMITH",
        gender: "Male",
        birthDate: "1958-07-17",
      },
      vaccine: {
        type: "Vaccine",
        disease: "COVID-19",
        atcCode: "J07BX03",
        medicinalProductName: "COVID-19 Vaccine Moderna",
        marketingAuthorizationHolder: "Moderna Biotech",
      },
    },
    mappingRules: null,
  });

  console.log("Credential", credential);

  const kms = new KMSClient({
    lang: LANG.es,
    storage: new FileSystemKMSSecureStorage({
      filepath: "file-system-storage",
    }),
  });

  const bbsbls2020 = await kms.getPublicKeysBySuiteType(Suite.Bbsbls2020);

  const vc = await kms.signVC(
    Suite.Bbsbls2020,
    bbsbls2020[0],
    credential,
    "did:quarkid:EiBA3ihJrI5fSsdpZWd3H_-0Wr4rEL8muoDOsuDQDhe2FQ",
    "did:quarkid:EiBA3ihJrI5fSsdpZWd3H_-0Wr4rEL8muoDOsuDQDhe2FQ" + "#bbsbls",
    new AssertionMethodPurpose()
  );
  console.log("Verifiable Credential Signed", vc);
};
credential();
```

> [!NOTE]
> Remember that you should always test with the DID created in the Creating a DID section, this is because to sign the credential it looks for the keys stored in the storage you created.

The next step is to execute the credential function. To do this, open a terminal and use ts-node or npx to run your code.

```
ts-node credential.ts
npx ts-node credential.ts
```

In your terminal, you will see the result of the console.log statements in your code, that is, you will see a created credential and then the same credential but signed.

## Verification of a VC

### 1. Install the following NPMJs package:

```
npm install @quarkid/vc-verifier @quarkid/did-resolver
```

### 2. Verify

To verify the credential, we will use the VcVerifierService. We need to pass this service a callback to resolve the DID, which will be the same function used in the Resolve section.

In the credential function you created in the previous step, add the following code:

```typescript
import { VCVerifierService } from "@quarkid/vc-verifier";
import { DIDUniversalResolver } from "@quarkid/did-resolver";
```

```typescript
const QuarkIDEndpoint = "https://node-ssi.buenosaires.gob.ar";

const service = new VCVerifierService({
  didDocumentResolver: async (did: string) => {
    const universalResolver = new DIDUniversalResolver({
      universalResolverURL: QuarkIDEndpoint,
    });

    const didDocument = await universalResolver.resolveDID(did);

    return didDocument;
  },
});
const result = await service.verify(vc, new AssertionMethodPurpose());
console.log("result", result);
```

We resolve the DID Document and call the verification function, which receives the credential and the purpose, which in this case is Assertion Method, which is the one used for credential verification.

The result variable returns true if it verifies correctly, or false with an error result if the verification failed.

### 3. Test

```
ts-node credential.ts
```

or

```
npx ts-node credential.ts
```

## Generation of Test Credential with QR code

1. Clone the [Demo](https://github.com/ssi-quarkid/Quickstart-Emision-Verificacion-de-Vcs/tree/main/demo) file

2. Remove the following files from the solution:

- issuer-secure-storage-ws.json
- issuer-storage-ws.json
- waci-protocol-ws.json

Since a new DID will be generated, they will be automatically regenerated when running the project.

### Prerequisites:

You need to have ngrok installed. To do this, run npm i ngrok -g in a new Terminal,
different from the one you're going to use to run the project.

You must download the QuarkID wallet on a mobile device [iOS](https://apps.apple.com/ar/app/quarkid/id6450680088) or [Android](https://play.google.com/store/apps/details?id=com.quarkid).

### Steps

1. In the console where you're running ngrok, you should execute the following:

```
ngrok http 3010
```

2. You will see in the console:

```
Full request capture now available in your browser: https://ngrok.com/r/ti

Session Status                online
Account                       X@gmail.com (Plan: Free)
Version                       3.9.0
Region                        South America (sa)
Latency                       33ms
Web Interface                 http://127.0.0.1:4040
Forwarding                    https://248c-161-22-25-244.ngrok-free.app -> http://localhost:3010
Connections                   ttl     opn     rt1     rt5     p50     p90
```

3. Copy the URL shown in "Forwarding", in this example it's "https://248c-161-22-25-244.ngrok-free.app"
   and paste it in the "app.module.ts" file on line 47 where it says "serviceEndpoint"

4. Save the changes

5. Run in another console that is not the one used for ngrok:

```
yarn
```

to download the dependencies.

6. Then run:

```
yarn start
```

You can navigate to the URL http://localhost:3010/invitation-message and you will see:

```
{"invitationId":"135204a7-91d2-4ff2-9352-9cc6d4f91272","invitationContent":"didcomm://?_oob=eyJ0eXBlIjoiaHR0cHM6Ly9kaWRjb21tLm9yZy9vdXQtb2YtYmFuZC8yLjAvaW52aXRhdGlvbiIsImlkIjoiMTM1MjA0YTctOTFkMi00ZmYyLTkzNTItOWNjNmQ0ZjkxMjcyIiwiZnJvbSI6ImRpZDpxdWFya2lkOkVpQ25pekFscVNWOXEzUHFEbGZUTVpzcV9LRUQ1Qm50OG1nelJIYmlsMEc4RmciLCJib2R5Ijp7ImdvYWxfY29kZSI6InN0cmVhbWxpbmVkLXZjIiwiYWNjZXB0IjpbImRpZGNvbW0vdjIiXX19"}
```

- The content of "invitationContent" (what's inside the quotes after the ":")
- Is the code used to draw the QR
- This code can be pasted into any QR generation website, we suggest [QR Code Generator](https://www.the-qrcode-generator.com/)

7. Select the Plain Text option, paste the content, and the QR is drawn to be scanned with the Production Quarkid wallet, available in the Operating Systems' Stores.
