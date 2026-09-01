# Money

Amounts are `numeric(19,4)` decimal **strings** everywhere in this module, and
all arithmetic runs on scaled `bigint` integers. Minor units (cents) exist only
at the SDK boundary. This is not fussiness: a marketplace splits one number
across N sellers and then reverses parts of it later, so a half-cent of float
drift becomes an unreversible transfer.

## The rules

| Rule | Why |
|---|---|
| Store `numeric(19,4)`, never `float`/`double` | Binary floats cannot represent `0.10` |
| Serialize as a **string** in JSON, currency in a sibling field | `120.00` survives the wire; `120` and `120.0000001` do not |
| Scale is fixed at 4 dp internally | Rates (`0.0290`) and amounts share one scale |
| Never mix currencies in one operation | There is no FX here; a mismatch is a programmer error |
| Quantize to minor units *before* anything provider-bound | Stripe takes integers and rejects sub-cent precision |

## The module

Complete and self-contained. `ES2017` target, so `BigInt(0)` rather than the `0n`
literal — drop the wrappers if you target ES2020+.

```ts
// lib/money.ts
const SCALE = 4;
const SCALE_FACTOR = BigInt(10000); // 10 ** SCALE
const B0 = BigInt(0);
const B1 = BigInt(1);
const B2 = BigInt(2);

export type Money = { amount: string; currency: string };

export class MoneyError extends Error {}

function fail(detail: string): never {
  throw new MoneyError(detail);
}

/** Parse a `numeric(19,4)` string (or number) into scaled bigint units. */
export function toUnits(amount: string | number): bigint {
  const s = typeof amount === "number" ? amount.toString() : amount.trim();
  if (!/^-?\d+(\.\d+)?$/.test(s)) fail(`invalid money amount: ${JSON.stringify(amount)}`);
  const negative = s.startsWith("-");
  const [intPart = "0", fracPartRaw = ""] = (negative ? s.slice(1) : s).split(".");
  if (fracPartRaw.length > SCALE) fail(`amount ${s} exceeds ${SCALE} decimal places`);
  const frac = `${fracPartRaw}0000`.slice(0, SCALE);
  const units = BigInt(intPart) * SCALE_FACTOR + BigInt(frac);
  return negative ? -units : units;
}

/** Format scaled bigint units back to a canonical `numeric(19,4)` string. */
export function fromUnits(units: bigint): string {
  const negative = units < B0;
  const abs = negative ? -units : units;
  const intPart = abs / SCALE_FACTOR;
  const frac = (abs % SCALE_FACTOR).toString().padStart(SCALE, "0");
  return `${negative ? "-" : ""}${intPart.toString()}.${frac}`;
}

export const ZERO = "0.0000";

export function normalizeAmount(amount: string | number): string {
  return fromUnits(toUnits(amount));
}
export function isZero(a: string): boolean {
  return toUnits(a) === B0;
}
export function add(a: string, b: string): string {
  return fromUnits(toUnits(a) + toUnits(b));
}
export function sub(a: string, b: string): string {
  return fromUnits(toUnits(a) - toUnits(b));
}
export function negate(a: string): string {
  return fromUnits(-toUnits(a));
}
export function sum(amounts: string[]): string {
  return fromUnits(amounts.reduce((acc, a) => acc + toUnits(a), B0));
}
export function mulQty(amount: string, qty: number): string {
  if (!Number.isInteger(qty) || qty < 0) fail(`invalid quantity: ${qty}`);
  return fromUnits(toUnits(amount) * BigInt(qty));
}
export function gte(a: string, b: string): boolean {
  return toUnits(a) >= toUnits(b);
}
export function gt(a: string, b: string): boolean {
  return toUnits(a) > toUnits(b);
}
export function lte(a: string, b: string): boolean {
  return toUnits(a) <= toUnits(b);
}
export function lt(a: string, b: string): boolean {
  return toUnits(a) < toUnits(b);
}
export function eq(a: string, b: string): boolean {
  return toUnits(a) === toUnits(b);
}
export function min(a: string, b: string): string {
  return lte(a, b) ? normalizeAmount(a) : normalizeAmount(b);
}
export function max(a: string, b: string): string {
  return gte(a, b) ? normalizeAmount(a) : normalizeAmount(b);
}

/**
 * Multiply by a rate (`"0.10"` commission, `"0.0290"` fee projection), rounding
 * half-up **on the magnitude** so -0.5 and +0.5 round away from zero alike.
 */
export function mulRate(amount: string, rate: string | number): string {
  const rateUnits = toUnits(typeof rate === "number" ? rate.toString() : rate);
  const product = toUnits(amount) * rateUnits; // now scaled x10^8
  const negative = product < B0;
  const abs = negative ? -product : product;
  const q = abs / SCALE_FACTOR;
  const r = abs % SCALE_FACTOR;
  const rounded = r * B2 >= SCALE_FACTOR ? q + B1 : q;
  return fromUnits(negative ? -rounded : rounded);
}

/** Major-unit string → provider minor units. Rejects sub-cent precision. */
export function toMinorUnits(amount: string, fractionDigits = 2): number {
  const units = toUnits(amount);
  const divisor = SCALE_FACTOR / BigInt(10) ** BigInt(fractionDigits);
  if (units % divisor !== B0) {
    fail(`amount ${amount} has sub-minor-unit precision for ${fractionDigits}-dp currency`);
  }
  return Number(units / divisor);
}

export function fromMinorUnits(minor: number, fractionDigits = 2): string {
  const divisor = SCALE_FACTOR / BigInt(10) ** BigInt(fractionDigits);
  return fromUnits(BigInt(Math.round(minor)) * divisor);
}

/**
 * Quantize to the currency's minor unit. Every rate-derived ledger leg must pass
 * through this at the point it is created — `toMinorUnits` throws later
 * otherwise, deep inside a transfer, where the failure is expensive.
 * `down` truncates toward zero, for figures that must never round up (a payout
 * must not exceed the balance that justified it).
 */
export function roundToMinorUnits(
  amount: string,
  fractionDigits = 2,
  mode: "half_up" | "down" = "half_up",
): string {
  const units = toUnits(amount);
  const grain = SCALE_FACTOR / BigInt(10) ** BigInt(fractionDigits);
  const negative = units < B0;
  const abs = negative ? -units : units;
  const q = abs / grain;
  const r = abs % grain;
  const rounded = mode === "half_up" && r * B2 >= grain ? q + B1 : q;
  return fromUnits((negative ? -rounded : rounded) * grain);
}

/** Exactly `fractionDigits` decimal places, for SDKs that take decimal strings. */
export function toDecimalString(amount: string, fractionDigits = 2): string {
  const minor = toMinorUnits(amount, fractionDigits);
  if (fractionDigits === 0) return String(minor);
  const negative = minor < 0;
  const digits = Math.abs(minor).toString().padStart(fractionDigits + 1, "0");
  const cut = digits.length - fractionDigits;
  return `${negative ? "-" : ""}${digits.slice(0, cut)}.${digits.slice(cut)}`;
}

export function assertSameCurrency(a: Money, b: Money): void {
  if (a.currency !== b.currency) fail(`currency mismatch: ${a.currency} vs ${b.currency}`);
}
```

## `distribute` — the one everyone gets wrong

Splitting one amount across N weights must sum back to **exactly** the original.
Naive per-part rounding leaks a cent, and that cent later makes a reversal exceed
its transfer, which Stripe rejects outright.

Largest-remainder method, allocating in whole **minor** units because the parts
feed provider APIs:

```ts
export function distribute(total: string, weights: number[], fractionDigits = 2): string[] {
  const n = weights.length;
  if (n === 0) return [];
  const grain = SCALE_FACTOR / BigInt(10) ** BigInt(fractionDigits);
  const totalUnits = toUnits(total);
  const totalGrains = totalUnits / grain; // truncates toward zero
  const residue = totalUnits % grain; // sub-minor dust on the input itself

  const weightBig = weights.map((w) => {
    if (!Number.isFinite(w) || w < 0) fail(`invalid weight: ${w}`);
    return BigInt(Math.round(w * 1_000_000)); // 6-dp weight precision
  });
  let weightSum = weightBig.reduce((acc, w) => acc + w, B0);
  let effective = weightBig;
  if (weightSum === B0) {
    // All-zero weights split evenly rather than dividing by zero.
    effective = weights.map(() => B1);
    weightSum = BigInt(n);
  }

  const base: bigint[] = [];
  const remainders: { idx: number; rem: bigint }[] = [];
  let allocated = B0;
  for (let i = 0; i < n; i++) {
    const numerator = totalGrains * (effective[i] as bigint);
    const q = numerator / weightSum;
    base.push(q);
    allocated += q;
    remainders.push({ idx: i, rem: numerator % weightSum });
  }

  // Hand the leftover grains to the largest remainders, one at a time. `step`
  // is signed so a negative total (a reversal) distributes correctly too.
  let leftover = totalGrains - allocated;
  remainders.sort((a, b) => (b.rem === a.rem ? 0 : b.rem > a.rem ? 1 : -1));
  const step = leftover >= B0 ? B1 : -B1;
  let k = 0;
  while (leftover !== B0) {
    const target = (remainders[k % n] as { idx: number }).idx;
    base[target] = (base[target] as bigint) + step;
    leftover -= step;
    k++;
  }

  const parts = base.map((g) => g * grain);
  // Sub-minor dust rides on the top-ranked part so the sum stays exact.
  if (residue !== B0) {
    const top = (remainders[0] as { idx: number }).idx;
    parts[top] = (parts[top] as bigint) + residue;
  }
  return parts.map(fromUnits);
}
```

Used for: splitting a gateway fee across sellers by net weight, prorating a
partial refund across transfers, splitting shipping and tax across line items.

## Tests worth keeping

These encode the properties the module claims. Port them with the code.

```ts
import { describe, expect, it } from "vitest";
import { distribute, mulRate, roundToMinorUnits, sum, toMinorUnits } from "./money";

describe("distribute", () => {
  it("always sums back to the total", () => {
    for (const [total, weights] of [
      ["100.0000", [1, 1, 1]],
      ["0.0300", [1, 1, 1]],
      ["999.9900", [7, 3, 11, 2]],
      ["-50.0000", [5, 5, 1]],
    ] as const) {
      expect(sum(distribute(total, [...weights]))).toBe(total);
    }
  });

  it("splits evenly when every weight is zero", () => {
    expect(distribute("9.0000", [0, 0, 0])).toEqual(["3.0000", "3.0000", "3.0000"]);
  });

  it("produces only minor-unit-representable parts", () => {
    for (const part of distribute("10.0000", [1, 1, 1])) {
      expect(() => toMinorUnits(part)).not.toThrow();
    }
  });

  it("does not mutate the input weights", () => {
    const weights = [3, 1];
    distribute("4.0000", weights);
    expect(weights).toEqual([3, 1]);
  });
});

describe("rounding", () => {
  // mulRate rounds at the 4-dp internal scale, NOT at the minor unit. Quantize
  // separately with roundToMinorUnits before anything provider-bound.
  it("rounds a rate product half-up at 4 dp, symmetrically about zero", () => {
    expect(mulRate("1.5000", "0.0001")).toBe("0.0002"); // exactly .00015 -> up
    expect(mulRate("-1.5000", "0.0001")).toBe("-0.0002"); // magnitude, not floor
    expect(mulRate("1.4999", "0.0001")).toBe("0.0001"); // just below -> down
  });

  it("leaves an exactly-representable product alone", () => {
    expect(mulRate("0.0500", "0.5")).toBe("0.0250");
  });

  it("computes a commission, then quantizes it for the provider", () => {
    expect(mulRate("199.9900", "0.1500")).toBe("29.9985"); // 4 dp, exact
    expect(roundToMinorUnits(mulRate("199.9900", "0.1500"))).toBe("30.0000"); // half-up
    expect(roundToMinorUnits(mulRate("199.9900", "0.1500"), 2, "down")).toBe("29.9900");
  });

  it("truncates toward zero in `down` mode", () => {
    expect(roundToMinorUnits("1.9999", 2, "down")).toBe("1.9900");
  });

  it("rejects sub-cent precision at the SDK edge", () => {
    expect(() => toMinorUnits("1.0050")).toThrow();
  });
});
```

## Related

[architecture.md](architecture.md) for why no FX,
[reconciliation.md](reconciliation.md) for where `distribute` is used in anger.
