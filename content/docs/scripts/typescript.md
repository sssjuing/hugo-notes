---
title: TypeScript 范型计算
date: "2025-02-22"
---

---

```ts
// 获取数组中的元素类型
type ElementOf<T> = T extends Array<infer U> ? U : never;

// T,U 都是 key-value 类型, 从 T 的 key 集合中剔除与 U 的 key 相同的 key
type Without<T, U> = Pick<T, Exclude<keyof T, keyof U>>;

// T,U 都是 key-value 类型, 返回 T 与 U 的 key 集合的交集构成的 key-value 类型
type PartialPick<T, U extends keyof T> = { [P in U]?: T[P] };

//
type WithNull<T> = T | null;
```

### Overwrite

```ts
type Overwrite<T, U> = Pick<T, Exclude<keyof T, keyof U>> & U;
```

T,U 都是 key-value 类型, 将 U 的属性添加到 T, 同时使用 U 的属性覆盖 T 的同名属性。例如：

```ts
type A = { a: number; b: string; c: boolean };
type B = { b: string; c: number; d: boolean };
```

则 Overwrite<A,B> 的类型为：

```ts
{
  a: number; // A 的属性
  b: string; // A 的属性和 B 的属性相同
  c: number; // B 覆盖了 A 的同名属性
  d: boolean; // B 的属性添加到 A
}
```

### Common

```ts
type Common<T, U> = { [K in keyof T & keyof U]: T[K] | U[K] };
```

T,U 都是 key-value 类型, 抽取 T,U 中都存在的 key 组成新的类型, 值类型合并为联合类型。例如：

```ts
type A = { a: number; b: string; c: boolean };
type B = { b: string; c: number; d: boolean };
```

则 Common<A,B> 的类型为：

```ts
{
  b: string; // 因为 A 和 B 中 b 的类型都是 string
  c: boolean | number; // 因为 A 中 c 的类型是 boolean，B 中 c 的类型是 number
}
```
