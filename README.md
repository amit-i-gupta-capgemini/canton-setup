# Canton beginner guide

This document explains a simple local Canton setup using the sample Daml project in this workspace.

## Prerequisites

Before starting, make sure you have the following installed:

- Daml SDK / DPM version 3.5.11
- Java 17 or newer
- A terminal such as PowerShell or Command Prompt

## Sample project used in this walkthrough

I created a sample Daml project using:

```bash
dpm new DamlFirstApp
```

The default template created a basic Daml module in [DamlFirstApp/main/daml/Main.daml](../../DamlFirstApp/main/daml/Main.daml).

After that, I built the project with:

```bash
dpm build
```

This generated the DAR file:

- [DamlFirstApp/main/.daml/dist/DamlFirstApp-main-0.0.1.dar](../../DamlFirstApp/main/.daml/dist/DamlFirstApp-main-0.0.1.dar)

That DAR file is then used by the Canton bootstrap script to upload the Daml package into the local network.

---

## 1. Setting up a node

A Canton node is made of several components working together:

- Sequencer: orders transactions and creates shared ledger history
- Mediator: coordinates multi-party workflows
- Participant: hosts ledger state for parties and exposes APIs

In this example, the setup is defined inline below.

---

```hocon
// File: examples/example- 1 participant/canton.conf
canton {
  sequencers {
    globalSequencer {
      public-api { address = "0.0.0.0", port = 5001 }
      admin-api  { address = "0.0.0.0", port = 5002 }
      storage.type = memory
    }
  }
  mediators {
    globalMediator {
      admin-api { address = "0.0.0.0", port = 5202 }
      storage.type = memory
    }
  }
  participants {
    participantNode1 {
      ledger-api      { address = "0.0.0.0", port = 5011 }
      admin-api       { address = "0.0.0.0", port = 5012 }
      http-ledger-api { address = "0.0.0.0", port = 7575 }
      storage.type = memory
    }
  }
}
```

// File: examples/example- 1 participant/init.canton
```scala
import com.digitalasset.canton.config.RequireTypes.PositiveInt
import com.digitalasset.canton.admin.api.client.data.StaticSynchronizerParameters
import com.digitalasset.canton.version.ProtocolVersion

// 1. Boot up the node infrastructure
nodes.local.start()

// 2. Initialize the blind coordination network (Sequencer + Mediator)
bootstrap.synchronizer(
  synchronizerName = "global",
  sequencers = Seq(globalSequencer),
  mediators = Seq(globalMediator),
  synchronizerOwners = Seq(globalSequencer, globalMediator),
  synchronizerThreshold = PositiveInt.one,
  staticSynchronizerParameters = StaticSynchronizerParameters.defaultsWithoutKMS(ProtocolVersion.forSynchronizer)
)

// 3. Connect the physical participant node to the network grid
participantNode1.synchronizers.connect_local(globalSequencer, alias = "global")

// 4. Upload your compiled Daml 3.x smart contract DAR file
println("--- Uploading DamlFirstApp-main DAR Package ---")
participantNode1.dars.upload("DamlFirstApp/main/.daml/dist/DamlFirstApp-main-0.0.1.dar")

// 5. Allocate the logical network identities required by Main.daml
println("--- Provisioning Ledger Parties ---")
val issuerParty = participantNode1.parties.enable("AssetIssuer", synchronizer = Some("global"), synchronizeParticipants = Seq(participantNode1))
val aliceParty  = participantNode1.parties.enable("Alice", synchronizer = Some("global"), synchronizeParticipants = Seq(participantNode1))

// 6. Onboard individual User Accounts with explicit Ledger API capabilities
println("--- Onboarding User Accounts & Granting Rights ---")

// Create Issuer User Account
participantNode1.ledger_api.users.create(
  id = "issuer_user",
  primaryParty = Some(issuerParty),
  actAs = Set(issuerParty),
  readAs = Set(issuerParty)
)

// Create Alice User Account
participantNode1.ledger_api.users.create(
  id = "alice_user",
  primaryParty = Some(aliceParty),
  actAs = Set(aliceParty),
  readAs = Set(aliceParty)
)

println("=============================================")
println("===          SANDBOX SYSTEM READY         ===")
println("=============================================")
println(s"participantNode1: parties=${participantNode1.parties.list().map(_.party.toProtoPrimitive)}")
println(s"Onboarded Users : ${participantNode1.ledger_api.users.list().users.map(_.id)}")
```

// File: DamlFirstApp/main/daml/Main.daml
```daml
module Main where

type AssetId = ContractId Asset

template Asset
  with
    issuer : Party
    owner  : Party
    name   : Text
  where
    ensure name /= ""
    signatory issuer
    observer owner
    choice Give : AssetId
      with
        newOwner : Party
      controller owner
      do create this with
           owner = newOwner
```
---

### Start the local network

Run the following command from the project root:

```powershell
java -jar bin\canton.jar -c "examples\example- 1 participant\canton.conf" --bootstrap "examples\example- 1 participant\init.canton"
```

### What this command does

The bootstrap script:
1. starts the local node infrastructure
2. creates the synchronizer named "global"
3. connects the participant to the synchronizer
4. uploads the Daml DAR file
5. creates ledger parties
6. creates sample users

---

## 2. How user onboarding works

In Canton, onboarding usually happens in two layers:

1. Party onboarding
   - A party is a logical identity on the ledger
   - Example: "Alice" or "AssetIssuer"

2. User onboarding
   - A user is an API account that can submit commands
   - The user is linked to a party and granted rights

The sample bootstrap script creates:
- the party "AssetIssuer"
- the party "Alice"
- the users "issuer_user" and "alice_user"

It also grants the required rights to let those users act on behalf of their parties.

---

## 3. Entry points into the Canton network

There are three main ways to interact with the network.

### A. Ledger API (gRPC, port 5011)

This is the main programming interface for backend systems and service-based integrations.

#### Infra setup
- Make sure your participant node in [examples/example- 1 participant/canton.conf](canton.conf) exposes the ledger API on port 5011.
- Start Canton with the bootstrap configuration:

```powershell
java -jar bin\canton.jar -c "examples\example- 1 participant\canton.conf" --bootstrap "examples\example- 1 participant\init.canton"
```

- After the node is up, upload your Daml package to the participant using:

```scala
participantNode1.dars.upload("DamlFirstApp/main/.daml/dist/DamlFirstApp-main-0.0.1.dar")
```

#### Client setup
- This is the best option for backend services such as Spring Boot applications.
- Typical Java client libraries include the Daml bindings and gRPC dependencies.
- Example Maven dependencies:

```xml
<dependency>
  <groupId>com.daml</groupId>
  <artifactId>bindings-java</artifactId>
  <version>3.5.11</version>
</dependency>
<dependency>
  <groupId>io.grpc</groupId>
  <artifactId>grpc-netty-shaded</artifactId>
</dependency>
```

- Example Java usage:

```java
DamlLedgerClient client = DamlLedgerClient.newBuilder("localhost", 5011).build();
client.connect();
```

### B. HTTP Ledger JSON API (REST, port 7575)

This is the easiest entry point for web apps, mobile apps, Postman, or Swagger-based integrations.

#### Infra setup
- The sample configuration already exposes the HTTP API on port 7575 through the participant node in [examples/example- 1 participant/canton.conf](canton.conf).
- When Canton starts, this endpoint becomes available automatically.
- You can browse the generated API docs at:

```text
http://localhost:7575/docs/openapi
```

#### Client setup
- You can use Postman, Swagger, or simple curl commands to call the REST endpoints.
- Example:

```bash
curl http://localhost:7575/v2/state/ledger-end
```

### C. Admin API (gRPC, port 5012)

This is used by operators and administrators for node management and operational tasks.

#### Infra setup
- The participant node exposes the admin API on port 5012 in [examples/example- 1 participant/canton.conf](canton.conf).
- This API is mainly for administration and is often restricted in production using firewalls or private networks.

#### Client setup
- Administrators commonly use the Canton shell or remote admin commands.
- Example command for remote-style admin access:

```powershell
java -jar bin\canton.jar -C canton.remote-participants.participantNode1.admin-api.port=5012 -C canton.remote-participants.participantNode1.ledger-api.port=5011
```

---

## 4. Useful first steps for a beginner

Here are a few simple commands you can try after startup:

### Check network health
```scala
health.status
```

### Check participant status
```scala
participantNode1.health.status
```

### List parties
```scala
participantNode1.parties.list()
```

### List users
```scala
participantNode1.ledger_api.users.list()
```

### Check the current ledger end
```bash
curl http://localhost:7575/v2/state/ledger-end
```

### Fetch active contracts
1. First get the current ledger offset:

```bash
curl http://localhost:7575/v2/state/ledger-end
```

2. Then pass that offset to the active-contracts API:

```bash
curl -X POST http://localhost:7575/v2/state/active-contracts?limit=7946&stream_idle_timeout_ms=7946 \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "activeAtOffset": 22,
    "eventFormat": {
      "filtersByParty": {
        "Alice::12200723c484600f013f3d47a6e693eb0440dadc5007f7158cf965726121be07edda": {}
      }
    }
  }'
```

> Replace the value `22` with the offset returned by the ledger-end API.

### Create a contract
1. First, find the party details for your onboarded users. The `primaryParty` from the user response is the value to use in the contract payload.

```bash
curl http://localhost:7575/v2/users/alice_user \
  -H "Accept: application/json"
```

Example response:

```json
{
  "user": {
    "id": "alice_user",
    "primaryParty": "Alice::1220aac02ac2a2745d409409b8c17357ed467d80d0946d495a4083fce09b1f2fe14b",
    "isDeactivated": false
  }
}
```

2. Find the template ID from the uploaded DAR package in the Canton console:

```scala
participantNode1.dars.list()
```

You will see the uploaded DAR. In this example, the relevant template ID is based on the uploaded DAR package:

```text
98c1dc87eae3463fd1148d9008d283086b38f82d1f821fa4e99b1c550fc974a7
```

3. Submit a create-command request:

```bash
curl --location 'http://localhost:7575/v2/commands/submit-and-wait' \
  --header 'Content-Type: application/json' \
  --header 'Accept: application/json' \
  --data '
  {
    "commandId": "create-asset-002",
    "userId": "issuer_user",
    "actAs": [
      "AssetIssuer::12200723c484600f013f3d47a6e693eb0440dadc5007f7158cf965726121be07edda"
    ],
    "commands": [
      {
        "CreateCommand": {
          "templateId": "98c1dc87eae3463fd1148d9008d283086b38f82d1f821fa4e99b1c550fc974a7:Main:Asset",
          "createArguments": {
            "issuer": "AssetIssuer::12200723c484600f013f3d47a6e693eb0440dadc5007f7158cf965726121be07edda",
            "owner": "Alice::12200723c484600f013f3d47a6e693eb0440dadc5007f7158cf965726121be07edda",
            "name": "Silver Asset"
          }
        }
      }
    ]
  }
  '
```

### View user rights
```scala
participantNode1.ledger_api.users.rights.list(id = "alice_user")
```

### Grant user rights
```scala
participantNode1.ledger_api.users.rights.grant(
  id = "alice_user",
  actAs = Set(aliceParty),
  readAs = Set(aliceParty)
)
```

### Disable a party
```scala
val aliceParty = participantNode1.parties.list(filterParty = "Alice").head.party
participantNode1.parties.disable(aliceParty)
```

> Note: a party can only be disabled when it is no longer a stakeholder in any active unarchived contract.

### Delete a user
```scala
participantNode1.ledger_api.users.delete(id = "alice_user")
```

### Deactivate a user
```scala
participantNode1.ledger_api.users.update(
  id = "alice_user",
  user => user.copy(isDeactivated = true)
)
```
