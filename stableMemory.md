# How to implement stable memory in azle

## In this example i'm deploying a token 
```ts
// token related
let tokenOwners = new StableBTreeMap<Principal, nat32>(1, 1000, 100);
let symbol = 'TGS';
let tokenName = 'Tiny game studio';
const fossetRecord = new StableBTreeMap<Principal, nat32>(2, 1000, 100);
const activeoupons = new Map<string, nat32>([
  ['1000', 1000],
  ['10', 10],
]);
```

## Please node the use of Option for stable memory to cypher content
```ts
$query;
export function getBalance(principal: Principal): number {
  const current = tokenOwners.get(principal);
  return match(current, {
    Some: (val) => val,
    None: () => 0,
  });
}
```

## Init function gets called once at creation
```ts
$init;
export function init(): void {
  tokenOwners.insert(ic.id(), 1_000_000_000);
  console.log('token created');
}
```

## Return token name etc
```ts
$query;
export function getTokenName(): string {
  return tokenName;
}

$query;
export function getTokenSymbol(): string {
  return symbol;
}

```

## Fosset and transfer
```ts
$update;
export function fosset(coupon: string): string {
  if (!activeoupons.has(coupon)) return 'invalid coupon';
  const value = activeoupons.get(coupon)!;
  const from = ic.id();
  const res = transfer_hidden(from, ic.caller(), value);
  return res;
}

$update;
export function gimmeGimme(): string {
  if (fossetRecord.containsKey(ic.caller())) return 'Already claimed';
  const value = activeoupons.get('10')!;
  const from = ic.id();
  fossetRecord.insert(ic.caller(), value);
  const res = transfer_hidden(from, ic.caller(), value);
  return res;
}

$update;
export function transfer(to: Principal, amount: nat32): string {
  const from = ic.caller();
  if (!tokenOwners.containsKey(from)) return 'insufficient balance';
  if (getBalance(from) < amount) return 'insufficient balance';
  if (!tokenOwners.containsKey(to)) tokenOwners.insert(to, 0);
  const fromBal = getBalance(from) - amount;
  const toBal = getBalance(to) + amount;
  tokenOwners.insert(from, fromBal);
  tokenOwners.insert(to, toBal);
  return 'success';
}

function transfer_hidden(
  from: Principal,
  to: Principal,
  amount: nat32,
): string {
  if (!tokenOwners.containsKey(from)) return 'insufficient balance';
  if (getBalance(from) < amount) return 'insufficient balance';
  if (!tokenOwners.containsKey(to)) tokenOwners.insert(to, 0);
  const fromBal = getBalance(from) - amount;
  const toBal = getBalance(to) + amount;
  tokenOwners.insert(from, fromBal);
  tokenOwners.insert(to, toBal);
  return 'success';
}
```