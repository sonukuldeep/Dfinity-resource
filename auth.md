# How to set up auth

## To run Internet identity cansiter locally insert this to dfx.json
remember to remove this when deploying to mainnet
```ts
    "internet_identity": {
      "type": "custom",
      "candid": "https://github.com/dfinity/internet-identity/releases/latest/download/internet_identity.did",
      "wasm": "https://github.com/dfinity/internet-identity/releases/latest/download/internet_identity_dev.wasm.gz",
      "remote": {
        "id": {
          "ic": "rdmx6-jaaaa-aaaaa-aaadq-cai"
        }
      },
      "frontend": {}
    }
```

## The imports you will need
```ts
import { AuthClient } from "@dfinity/auth-client";
import { Actor, Identity, ActorSubclass, HttpAgent } from "@dfinity/agent";
import { canisterId, createActor, idlFactory } from "../declarations/backend";
import { _SERVICE } from "../declarations/backend/backend.did";
```

## Default option which comes in handy
```ts
export const defaultOptions = {
  createOptions: {
    idleOptions: {
      // Set to true if you do not want idle functionality
      disableIdle: true,
    },
  },
  loginOptions: {
    identityProvider: `http://localhost:4943?canisterId=INTERNET_IDENTITY_CANISTER_ID_HERE#authorize`,
    // identityProvider: 'https://identity.ic0.app/#authorize',
    // Maximum authorization expiration is 8 days
    maxTimeToLive: BigInt(60 * 1000 * 1000 * 1000), // 1min
  },
};
```

## Auth client. V-important
```ts
export const authClient = await AuthClient.create(defaultOptions.createOptions);
```

## Sample login implementation
```ts
function callLogin() {
  authClient.login({
    ...defaultOptions.loginOptions,
    onSuccess: async () => {
      // set state or something
      // call function that need signed in user after authClient.login is success
      handleAuthenticated();
    },
    onError(error) {
      console.log(error);
    },
  });
}
```

## Sample auth handler that should be called after login
```ts
function handleAuthenticated() {
  const identity = authClient.getIdentity() as unknown as Identity;
  const whoami_actor = createActor(canisterId as string, {
    agentOptions: {
      identity,
    },
  });
  // Invalidate identity then render login when user goes idle
  authClient.idleManager?.registerCallback(() => {
    Actor.agentOf(whoami_actor)?.invalidateIdentity?.();
    //call login after login invalidate
    callLogin();
  });

  //   authenticated canister calls
  //   const response = await actor.whoami();
  renderLoggedIn(whoami_actor);
}
```

## Use authAgent to login logout etc
```ts
export const renderLoggedIn = (actor: ActorSubclass<_SERVICE>) => {
  (document.getElementById("whoamiButton") as HTMLButtonElement).onclick =
    async () => {
      try {
        const response = await actor.whoami();
        (document.getElementById("whoami") as HTMLInputElement).value =
          response.toString();
      } catch (error) {
        console.error(error);
      }
    };

  (document.getElementById("logout") as HTMLButtonElement).onclick =
    async () => {
      await authClient.logout();
      //call login after logout
      callLogin();
    };
};
```

## Another way to create acthenticated actor
On successful authClient.login, principal will no longer yield anonymous id "2vxsx-fae"<br/>
the actor can also be used to call other functions defined in the canister
```ts
// You can also do this to create actor and access canisters functions
const identity = authClient.getIdentity();
const principal = identity.getPrincipal().toString();
const actor = Actor.createActor(idlFactory, {
  agent: new HttpAgent({
    identity,
  }),
  canisterId,
});
```