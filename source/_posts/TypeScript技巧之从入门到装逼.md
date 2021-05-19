---
title: TypeScript技巧之从入门到装逼
date: 2019-8-23 14:55:08
categories:
  - TypeScript
tags:
  - Web
	- TypeScript
---

## 预备动作

> 阅读本篇之前你可能需要一定的TypeScript基础，如果还没开始上手建议从这里开始

> 以下是传送门
> 1. [TypeScript Document 官方文档](http://www.typescriptlang.org/docs/home.html)
> 2. [TypeScript HandBook 小手册](https://zhongsp.gitbooks.io/typescript-handbook/content/)
> 3. [TypeScript Playground 旧版练习场](http://www.typescriptlang.org/play/index.html)
> 4. [TypeScript Playground 新版练习场 (比上面那个高级，写作期间是beta版)](http://www.typescriptlang.org/v2/en/play)

## 正片开始
> 主要介绍 TypeScript 的常用和不常用的各种技巧，包括用这些技巧创建一些高级类型函数，会附上 Playground 的链接。

### 关键词
> 用于更加细粒度的处理接口和类型别名**

> - @scope *runtime* 运行时
> - @scope *lexical* 静态分析/词法分析

**类型断言 / 类型守卫 / 类型约束**
> 可以获取，确定，约束变量类型

#### `typeof` @scope *runtime*
> 获取变量的类型，常用于一些需要确认变量的类型然后进行不同操作的场景

```typescript
type AOrB = { s: string; } | string;
const val: AOrB = { s: 'a' };
if (typeof val === 'object') {
  console.log(val.s);
} else {
  console.log(val);
}
```

[Playground Link](http://www.typescriptlang.org/v2/en/play#code/C4TwDgpgBAgg8gJwEJQLxQN5QM4C4fAICWAdgOYDcUAvlAD4HHkUCwAUAMYD2J2wUANwCGAG3zxkaTDnwByIbJqs2RAGZQAFKEhd1wkWlTpZXAEYArCB2CyAlJnZQo3XlxEQAdCK5kN+j9i2yrQQItjQGI7OPNhunt6++kHs1EA)

#### `extends` @scope *lexical*
> 类型约束，通常用在泛型传参时约束参数范围

```typescript
interface Gen {
    s: string;
}

interface Others {
    d: number;
}

function BaseFunc<P extends Gen>(p: P) {
    console.log(p);
}

const s1: Gen = {
    s: 'something',
};

const s2: Others = {
    d: 1,
};

BaseFunc(s1); // good

BaseFunc(s2); // bad, error
```

[Playground Link](http://www.typescriptlang.org/v2/en/play?target=7#code/JYOwLgpgTgZghgYwgAgOIRMg3gWAFDKHIDOAXCWFKAOYDc+AvvvqJLIigPJgAW0x2fEWQATciACuAWwBG0eniZ58MCSARhgAe0wAhOMQgAxNQgA8ABWQQAHpBAiB6EAD4AFAAdyFgJSCCRAg6xFoANhAAdKFa1J4+Ckr4QSDEYCQAjOTOyAC8-sJkyADkIVIQvDRFADSMCknBacQATOTcfFACebgBhGLI6TWKdXj6hibqbsTp8cgA9LPI1FpaIswjBsamk00z88gycCJV1lBQWlBAA)


#### `instanceof` @scope *runtime*
> 确认目标是是否在指定对象的原型链上

```typescript
class Animal {
    name: string;

    constructor(name: string) {
        this.name = name;
    }
}

class Dog extends Animal {
    wang: number;

    constructor(name: (typeof Animal)['name'], wang: number) {
        super(name);
        this.wang = wang;
    }

    wangwang() {
        console.log('wangwang');
    }
}

class Cat extends Animal {
    miao: number;

    constructor(name: (typeof Animal)['name'], miao: number) {
        super(name);
        this.miao = miao;
    }

    miaomiao() {
        console.log('miaomiao');
    }
}

const dog: Dog = new Dog('dog', 1);

const cat: Cat = new Cat('cat', 1);

function doS<C extends Animal>(cls: C) {
    /** bad */
    cls.wangwang(); // error
    cls.miaomiao(); // error
    /** good */
    /** 函数已经限制了 cls 的约束范围 */
    if (cls instanceof Dog) {
        /** 通过 instanceof 类型守卫 进一步确认类型 */
        cls.wangwang();
    } else if (cls instanceof Cat)  {
        /** 同上 */
        cls.miaomiao();
    }
}
```

[Playground Link](http://www.typescriptlang.org/v2/en/play?target=7#code/MYGwhgzhAECCB2BLAtmE0DeBYAUNf08YyApgFzQQAuATovAOYDcuuB0wA9vNTQK7AqnGgAoipCr3oMAlJjbsCVABaIIAOnEloAXkLESLPAQC+uMzlyhIMACKcG0EgA8qJeABMYCFGnnH8AHcwRgp4PmQAIxIaIwV8Lh5aASFRLQoRKgBPAAcSTgAzOCRUEBkAbQByLUqAXQAaaGDQwgjomjlsAMUIPjy0gxkjRSVVDWbHPQnh01ZuiYmRTvjFRIhOEBJ1EAcRSoWQhkqhlYsLK3AoaABhMConV3cvYt90LvZkRDBOMLaYuO6a2SgmEYgMGWyeUKL1KFWqBjqjU+31+URiy267F6-TBpBOmNGanUyM4umgJJm+HO3RJJKW-hGCW4602212lVpX04x0p0DOcyB0A8Dgo9kmhBIgWgYr2wqOjQAjPirMz7sA7hRbvc9PBJTc7nt1VRKorlTgCnx4IJENwhZwAMoAHmuDzcnm8JTQAD4RKAIJqMewAPQAKhD0EiYA80BDQZWfvUB0YSyY0CDQacNBownjIA0nO+KbTGZi2ZoK1D4YYnE40djFbD0EAv4qAB1NAE+6gHm-QAKaYA2JUAYXIcPPQQAhboAyv0A+uaAYGDAC9qMbj3UQRV9Q-o1BCwHyRTFgcZleggCwEwDj8dBV1R15voIBvH0A0eqACO1ANbK0EA2-GAADlAKbWgDsPQAl0Te5yt2AmSYMCmpxOHm2iLtAy4wKe57QlqcgMrujaADAqgBQcn+BIJHmxJcnS+LsGcQA)

<!-- more -->

#### `is` @scope *lexical*
> 利用函数返回值确认类型

```typescript
enum CUSTOM_ENUM {
    A = 'a',
    B = 'b',
}

interface Base {
    type: CUSTOM_ENUM;
}

interface A extends Base {
    type: CUSTOM_ENUM.A;
    a?: number;
}

interface B extends Base {
    type: CUSTOM_ENUM.B;
    b?: number;
}

function isA(val: Base): val is A {
    return val?.type === CUSTOM_ENUM.A;
}

/** 假设你不知道是啥类型 */
const a: any = {
    type: CUSTOM_ENUM.A,
    a: 1,
};
const b: any = 1;


console.log(a.a); // 恰好没有出错
if (isA(a)) {
    console.log(a.a) // good
}

console.log(b.a); // 潜在的运行时错误
if (isA(b)) {
    console.log(b.a) // 避免了出错
}
```

[Playground Link](http://www.typescriptlang.org/v2/en/play?target=7#code/KYOwrgtgBAwgqgZQCoHkCyB9AogOTmqAbwFgAoKCqAQSgF4oByAQwYBozKoAhOxgIzZkAvmTIBLEABdgAJwBmTAMbBuTAM4qS5SpICeAB2AAuWIlSZc+ANzDRpCdPlKVNYAA9pIACZrVGohw6BsamyOjYeGgAdFQ22hRMAPwm4BB8snEipOJSsgrK3FDunj5+moEUeoYm8GEWkVFccZx8yVCp6TKZdnJgIIqSYgD2IFBialQAFABuTAA2JlzqwACUJrNzY740WpwywJJgMqMbiVFVKrRXoeYR+DHd2aQA9ABUr1CA4gqAfdGABvKAsHKAU-dAMoJgHozQCmqoBvH0A0epQV7PMiKEZqSRQJgmJggXS8XZBao3cKWaJUdjxNEmACMpKEcSRIBRUD4GKxvApcTsdLUQzmwCicyGAHNJkwokwVlYoM9nlBAA4GgF9NQCFNoBIc0AX4qATFTxHIoJNxlMxSsAmTOdzefyhSKxZLpQKhkMvLYnsaeXzBZM+KLxVaoIAde0AFOqAELdAAvxgBkIwBvpmrAPfRmu1urdKwNuIoTtNrvdlqlUEA-gmAWUVAGFy6uEQA)


#### `as` @scope *lexical*
> 类型断言

```typescript
function double(s: number) {
    return Number(s) * 2;
}

const a: unknown = '1';

console.log(double(a as number));

```

[Playground Link](http://www.typescriptlang.org/v2/en/play?target=7#code/GYVwdgxgLglg9mABAEziARgGwKYAoDOAXImCALbrYBOAlIgN4CwAUIm4ldlCFUgHLlKVAnQBUiAEwBuFgF8WLCAnxREAQ2LgA1mDgB3JAF5EAcgCMJmc0XK4OAHSY4Ac1yoMOXGvX4Sg6jQ0VkA)


#### `in` @scope *runtime*
> es标准中的 `in` 标识符

```typescript
interface B {
    b: string;
}
interface A {
    a: string;
}


function foo(x: A | B) {
    // good
    if ('a' in x) {
        return x.a;
    }
    return x.b;
}

const a: A = {
    a: 'a',
}

const b: B = {
    b: 'b',
}

console.log(foo(a));
```

[Playground Link](http://www.typescriptlang.org/v2/en/play?target=7#code/JYOwLgpgTgZghgYwgAgELIN4FgBQz-IBGAXMgM5hSgDmA3LgL66iSyIoCCmuBycpFKiDqNcYnDACuIBGGAB7EMhjz5ACgAepLgB80ASm54CAehPJqqgCY8CwGMjUByOE+ShkGw9mO98UCDBJKCUNADo4el9kJmiAoJDPMMIo2NwERQo+bWQAXiNefmQXJwAaURx0zLAiUnR8n14SYsIyiqqQMnkAGwgw7vlqNRV1OH19WiA)


#### `!` @scope *runtime*
> 可以绕过严格空判断，TypeScript 会认为该属性一定存在

```typescript
interface A {
  s?: {
    t: string;
  };
}
const a: A = {
  s: {
    t: 'something',
  },
};
console.log(a.s.t); // error，s 可能为空
console.log(a.s!.t); // 不会报错，s 被认为一定存在
```

[Playground Link](http://www.typescriptlang.org/v2/en/play#code/JYOwLgpgTgZghgYwgAgILIN4FgBQzkDOA-AFya775hkFhSgDmA3BcgL4s5u4ID2ItZHDLoAvOTyEy2SVTIByArwC2EMAAtG8gDSs2urpz4DeAGwgA6U7wYAKOBYIWwASibIA9B+TQovKIAw-wTIgPfKgL8BgFxygF5ePPxK5lY29o4AhM5unt6AsHKAWPKApUaAmKlByIDVEYAl0RGAAHKAWdqAGtqAFOpAA)

---

**类型索引**
> 获取 interface / type 中的某个属性

#### `T[K]` @scope *lexical*

```typescript
interface PersonStruct {
    name: string;
    age: number;
}

class Person implements PersonStruct {
    // 通过 T[K] 的模式设置属性类型
    name: PersonStruct['name'];
    age: PersonStruct['age'];

    constructor({ name, age }: PersonStruct) {
        this.name = name;
        this.age = age;
    }
}
```

[Playground Link](http://www.typescriptlang.org/v2/en/play?target=7#code/JYOwLgpgTgZghgYwgAgArQM4HsQGUxQCuCYyA3gLABQytyIcAthAFzIYGgDmA3NXcjhdW9QowBG0PlQC+1aggA2cDBjSYcyYIwAOiiM3Br0UbHgLFSlGnQD0t5ICwEwOPxyACoBtANIBdZIBC3QEIrQHh9QD7owDt-QD0dQHIDQG8fQGj1fjoGZjYTM3wiEg8AchSIHJ9pASERdJxMy1zSwukk2gQcDiywLCgACjJ6JggAGkFhZBk0jXMWgEpyeoFkMAALYAwAOnzkAF5u5mKZ2nnFpdL1gYht2jlZaiA)

---

**类型属性枚举**
> 列举 interface 中的子属性

#### `keyof` @scope *lexical*
> 枚举 interface 中的成员


```typescript
interface PersonStruct {
    name: string;
    age: number;
}

type PersonKeys = keyof PersonStruct; // name | age
```

[Playground Link](http://www.typescriptlang.org/v2/en/play?target=7#code/JYOwLgpgTgZghgYwgAgArQM4HsQGUxQCuCYyA3gLABQytyIcAthAFzIYGgDmA3NXcjhdW9QowBG0PlQC+1amACeABxToo2EAGkIijMgC8yANa6sMNJhz4iJHsgD0D+kxQAfQcOpA)

---

**属性赋值**
> 把枚举的到的属性赋值到目标上

#### `in` @scope *lexical*
> 这个 `in` 与类型守卫中的 `in` 的含义是不一样的
> 静态分析中的 `in` 是赋值

```typescript
interface PersonStruct {
    name: string;
    age: number;
}

type Mapper = {
    [k in keyof PersonStruct]: string; // { name: string; age: string; }
}
```

[Playground Link](http://www.typescriptlang.org/v2/en/play?target=7#code/JYOwLgpgTgZghgYwgAgArQM4HsQGUxQCuCYyA3gLABQytyIcAthAFzIYGgDmA3NXcjhdW9QowBG0PlQC+1amACeABxQBZOMtVRkAXnL86AbQDWyUMhMRFWGGkw58REgF02HKNx7IA9D-L0TCIeXoLC7pwgvMhyskA)

---

**属性状态**
> 描述一个属性的状态

#### `readonly` @scope *lexical*
> 被标记的属性无法被修改/赋值

```typescript
interface PersonStruct {
    readonly name: string;
    readonly age: number;
}

const person: PersonStruct = {
    name: 'xxx',
    age: 19,
}

person.age = 1; // error
person.name = 'yyy'; // error

class Person {
    readonly name: string;
    readonly age: number;

    constructor({ name, age }: PersonStruct) {
        this.name = name;
        this.age = age;
    }
}

const p = new Person({ name: 'xxx', age: 198 });
p.age = 1; // error
p.name = 'yyy'; // error
```

[Playground Link](http://www.typescriptlang.org/v2/en/play?target=7#code/JYOwLgpgTgZghgYwgAgArQM4HsQGUxQCuCYyA3gLABQytyUEcAJjgDYCeyIcAthAFzIMBUAHMA3NTr1GLEB2RxRAroR4AjaJKoBfatQQ5hyAA6Ycg9FGx4CxUgF5yUutz6CA5AA8fHgDQutEoqAIwAnAG6+lRm1jgAdMHITiHiyAD06cjQUFhQ1LE28W4oTh7sFR5pmdlQuflUBqxwGBho5iDONHQMzGycJYLCUGLa0r1yCsGCIGqaUNqByIYgw-Z5ABRkXLwQforKyDqWHfhEJACUXdLSYAAWwBjFu8k7fGM3tPePiYdOwR9aHooo0qCtjCZXiAIAB3dpxEBbN4qby+fbTZDhAAcRwu2hMv1KmOqWRyeQKzz4r3KlRJtXq1CAA)

#### `?` @scope *lexical*
> 属性状态-可选属性

```typescript
interface PersonStruct {
    name: string;
    age?: number;
}

// good, age可以有也可以没有
const person: PersonStruct = {
    name: 'xxx',
}
```

[Playground Link](http://www.typescriptlang.org/v2/en/play?target=7&ssl=1&ssc=1&pln=10&pc=1#code/JYOwLgpgTgZghgYwgAgArQM4HsQGUxQCuCYyA3gLABQytyIcAthAFzIYGgDmA3NXcjhcIAfjYhCjAEbQ+VAL7VqAemXIuWLABMANIOGB75UCncoEhzQPpyxwIU2J6ghwdkAB0w426KNjwFipALzl+OgZmNgByAA9I0J1qRSogA)

#### `-` @scope *lexical*
> 移除属性状态的标记

```typescript
interface PersonStruct {
    readonly name: string;
    age?: number;
}

type Remove = {
    -readonly [k in keyof PersonStruct]-?: PersonStruct[k]; // { name: string; age: number }
}
```

[Playground Link](http://www.typescriptlang.org/v2/en/play?target=7#code/JYOwLgpgTgZghgYwgAgArQM4HsQGUxQCuCYyA3gLABQytyUEcAJjgDYCeyIcAthAFzIMBUAHMA3NTrI4oiAH5BIQjwBG0SVQC+1amHYAHFACUIPLADcUAXnJS6AWgbM2nANoBrZKGQeI7LBg0TBx8IhIAXQdFYKhsPAJiME8I8WQAenTyLl4BIREQCRk5JRV1KGQdbWogA)

---

**类型推导**
> 提供一个类型（不需要具体实现），通过推导的方式得到其类型或其子属性类型

#### `infer` @scope *lexical*
> 推导出某个类型中的某个属性类型

```typescript
interface PersonStruct {
    name?: string;
    age: number;
}

type PersonList = PersonStruct[];

type ListItemType = PersonList extends (infer T)[] ? T : unknown; // PersonStruct
```

[Playground Link](http://www.typescriptlang.org/v2/en/play?target=7&ssl=1&ssc=25&pln=1&pc=1#code/JYOwLgpgTgZghgYwgAgArQM4HsQGUxQCuCYyA3gLABQytyIcAthAPwBcyGBoA5gNzU6yODwgcQhRgCNoAqgF9q1MAE8ADinRRsIADLAuyALxpMOfERIBtALpzl6lPq4BJSIwAqj46e05npBAAHpAgACYYyAAUoDDQyB4AlLbILAnIHIQgANYgWADuIHzIAPQlvjoWxGDUQA)

---

### 优雅的处理全局变量
> 优雅解决window下挂载的自定义变量

```typescript
/** typings/window.d.ts */
declare module _global {
    export const someVar: number;
}

interface Window {
    _global: typeof _global;
}


/** src/you path/somefile.d.ts */
console.log(window._global.someVar); // good

console.log(_global.someVar); // good
```

[Playground Link](http://www.typescriptlang.org/v2/en/play?target=7#code/PQKhAIBcE8AcEsB2BzAzsA7kgJgewwHTYGSrgjACwAUNgKYDGANgIYBOd4AtrtgK5NOAfWRNcAIxZNwAbxrgF4OgA9YuNpHANciVJtS4udAGrsAXOER8u4umwDcNAL40aSSHYBmLBpwDqOPiy8ooiYpJMFjCwdLie4GESUo7ULtSu1KAQqGwMwNC4fOCwLJAAFsAGRp7wgkQkZBQ02rq4dWLIABRYiHiEiREEVSbsAJT24MDA4Mi4vBktBu24XQNSQ4YjbOOT07O8QA)

---

### 优雅的处理第三方库

```typescript
// 有些第三方库可能没有 d.ts

/** typings/react.d.ts */
declare namespace SomeThirdModule {
    export const someFunc: () => void;
}

/** src/you path/somefile.d.tsx */

SomeThirdModule.someFunc(); // good
```

[Playground Link](http://www.typescriptlang.org/v2/en/play?target=7#code/PTAEkhzRsuUGm9Eg5RO00Ml6h75UL8BhCm3KAJgOgC4DOAsAFBnABUlo+AngA4CWAdgOaHABOApgIYBjfLjxFQlYGWw8BAGz69QLPgFsehBoJ6gAygHs1AFQAWTLtgCye7AFdZ2gN5lQL0DwAeDPV3ygBelkJfQgMeADEbFgEALlAACgBKUABeAD5QADc9JmwAbjIAXzIKalBCLgFgOj0bUE18Y2AQtQAzJnsRAkJ3cUlyUn0jU3MrWw7m8MiBRNzQEFA2PWsyIA)

---

### 优雅的处理非模块文件
> 图片和样式文件会提示非 module

```typescript
/** typings/ext.d.ts */

declare module '*.scss' {
    const value: { [className: string]: string };
    export default value;
}

/**
 * @description svg 模块
 */
declare module '*.svg' {
    import * as React from 'react';

    const src: React.SVGProps<SVGSVGElement>;
    export default src;
}
```

[Playground Link](http://www.typescriptlang.org/v2/en/play?target=7#code/PQKhAIBcE8AcEsB2BzAzsApgD0gOgCa6SrgjACwAUFfhgMYA2AhgE4bgC2A9vgK4PsA5CFyo6qVIPABvKuHng6XRKkjgAbkwa8MALhngA2oyYSAckw57wqlkmQBdfbfvgAvgG45C7LC4s1WgAzJn41TW0ML0o3KipQEDkIAAFaMTtYSHhlG3VkcEBCK0B1dSSKSloTNk4efiERVDypWUoFcHgOPwDScFNwACUMJjo1IJYuDnBBNiHIQWjveSUVNVQWOn0BmdwAZQA1AHEABTHYVAAePf3LgFEBK0RIAD5o1t9-QIwQsJs16LcgA)

---

### TypeScript 内置类型函数 列举（不完全）

#### `Pick`


#### `Record`


#### `Partial`


#### `Required`


#### `Readonly`


#### `Exclude`


#### `Extract`


#### `Omit`


#### `NonNullable`


#### `Parameters`


#### `ConstructorParameters`


#### `ReturnType`


#### `InstanceType`


#### `ThisType`

