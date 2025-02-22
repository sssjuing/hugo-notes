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

// T,U 都是 key-value 类型, 将 U 的属性添加到 T, 同时使用 U 的属性覆盖 T 的同名属性
type Overwrite<T, U> = Pick<T, Exclude<keyof T, keyof U>> & U;
```
