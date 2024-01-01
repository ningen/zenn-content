---
title: "zodの実装を少しだけ読んでみた"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "zod"]
published: true
publication_name: minma
---

# はじめに

最近、gitlabのapiを叩くのにdeno+zodでお試ししてみたが、かなり開発者体験が良かった。

OSSにコントリビュートしてみたい気持ちが少しあるのと、仕組みも若干気になったため、最小のコードがどのように動くのか読んでみた。

# 注意

- 今回読むに当たって必要がない部分は、断りなく省略しています。

## 今回読む対象のサンプルコード

```tsx
import { z } from "zod"

const boolSchema = z.boolean();
const boolResult = boolSchema.parse(true);

console.log(boolResult);
```

## z.boolean()は実際には何を呼んでいるのか

import される`z`は、index.tsで以下のように記載されている。

```tsx
import * as z from "./external";
export * from "./external";
export { z };
export default z;
```

実態はexternal.tsにありそうなため、external.tsを確認する。

```tsx
export * from "./errors";
export * from "./helpers/parseUtil";
export * from "./helpers/typeAliases";
export * from "./helpers/util";
export * from "./types";
export * from "./ZodError";
```

types.tsに以下の記述があり実態は、types.tの`ZodBoolean.create()`になっていることがわかる。

```tsx
const booleanType = ZodBoolean.create;
// ...

export {
	// ...
  booleanType as boolean,
	// ...
};
```

`Zodboolean`は以下の様な実装になっている。

```tsx
export interface ZodBooleanDef extends ZodTypeDef {
  typeName: ZodFirstPartyTypeKind.ZodBoolean;
  coerce: boolean;
}

export class ZodBoolean extends ZodType<boolean, ZodBooleanDef> {
  // _parse()が存在するが、後述するため一旦省略

  static create = (
    params?: RawCreateParams & { coerce?: boolean }
  ): ZodBoolean => {
    return new ZodBoolean({
      typeName: ZodFirstPartyTypeKind.ZodBoolean,
      coerce: params?.coerce || false,
      ...processCreateParams(params),
    });
  };
}
```

ここまでで、以下のことが分かる。

- `z.boolean()`の実態は`ZodBoolean.create()`である。
    - typeNameにZodFirstPartyTypeKind.ZodBoolean(実態は定数の`”ZodBoolean”`)
    - coerceには、params.coerce or falseを渡している
- `ZodBoolean`は`ZodType`を継承している。

## parseしたときに何が起きているか

schemaを定義したときの動作はわかったため、次は`boolSchema.parse`を呼んだときの処理を追ってみる。

`ZodBoolean`側に`parse`が存在せず`ZodType`クラスに存在する。

```tsx
abstract _parse(input: ParseInput): ParseReturnType<Output>;

_parseSync(input: ParseInput): SyncParseReturnType<Output> {
    const result = this._parse(input);
    if (isAsync(result)) {
      throw new Error("Synchronous parse encountered promise.");
    }
    return result;
 }

safeParse(
    data: unknown,
    params?: Partial<ParseParams>
  ): SafeParseReturnType<Input, Output> {
    const ctx: ParseContext = {
      common: {
        issues: [],
        async: params?.async ?? false,
        contextualErrorMap: params?.errorMap,
      },
      path: params?.path || [],
      schemaErrorMap: this._def.errorMap,
      parent: null,
      data,
      parsedType: getParsedType(data),
    };
    const result = this._parseSync({ data, path: ctx.path, parent: ctx });

    return handleResult(ctx, result);
 }

parse(data: unknown, params?: Partial<ParseParams>): Output {
    const result = this.safeParse(data, params);
    if (result.success) return result.data;
    throw result.error;
}

*// ...ZodType外*
const handleResult = <Input, Output>(
  ctx: ParseContext,
  result: SyncParseReturnType<Output>
):
  | { success: true; data: Output }
  | { success: false; error: ZodError<Input> } => {
  if (isValid(result)) {
    return { success: true, data: result.value };
  } else {
    if (!ctx.common.issues.length) {
      throw new Error("Validation failed but no issues detected.");
    }

    return {
      success: false,
      get error() {
        if ((this as any)._error) return (this as any)._error as Error;
        const error = new ZodError(ctx.common.issues);
        (this as any)._error = error;
        return (this as any)._error;
      },
    };
  }
};
```

以下の処理の流れであることが分かる。

1. 受け取ったデータと、パラメーターをそのまま`safeParse()`に渡す。
    1. `safeParse()`はparseに関する文脈を作成した上で、`_parseSync()`を呼ぶ。
        1. 子クラス側で実装された、`_parse()` を呼ぶ。
            1. 返り値が非同期処理である場合は、エラーを投げる。
            2. そうでない場合は、`_parse()`の返り値をそのまま返す。
    2. `_parseSync()` の結果を`handleResult()`に渡す。
        1. `isValid()` の場合は`success: true`にして、値をwrapして返す。
        2. `isValid()`がfalseなのにも関わらず、問題(`issues`)が存在する場合はエラーをthrowする。
        3. 違う場合は、エラーを生成してreturnする。
    3. `handleResult()` の結果を`parse()`に返す。
    4. safeParse()のsuccessがtrueなら、データを返す。
    5. dで値が帰っていない場合(`success: false`)は、エラーをthrowする。

では、`ZodBoolean`で実装されている`_parse()`を確認してみる。

```tsx
// parseUtils.ts内
export type INVALID = { status: "aborted" };
export const INVALID: INVALID = Object.freeze({
  status: "aborted",
});

export type OK<T> = { status: "valid"; value: T };
export const OK = <T>(value: T): OK<T> => ({ status: "valid", value });

// utils.ts内
export const getParsedType = (data: any): ZodParsedType => {
  const t = typeof data;

  switch (t) {
    // 他のケースは省略
    case "boolean":
      return ZodParsedType.boolean;
}
// type.ts内

// 親クラス内の処理
export class ZodType {
	_getType(input: ParseInput): string {
    return getParsedType(input.data);
  }
}
export class ZodBoolean extends ZodType<boolean, ZodBooleanDef> {
  _parse(input: ParseInput): ParseReturnType<boolean> {
    if (this._def.coerce) {
      input.data = Boolean(input.data);
    }
    const parsedType = this._getType(input);

    if (parsedType !== ZodParsedType.boolean) {
      const ctx = this._getOrReturnCtx(input);
      addIssueToContext(ctx, {
        code: ZodIssueCode.invalid_type,
        expected: ZodParsedType.boolean,
        received: ctx.parsedType,
      });
      return INVALID;
    }
    return OK(input.data);
  }

  // create()は関係ないため、省略
}
```

以下の処理の流れであることが分かる。

1. coerceがtrueの場合は、渡された値を変換する。
2. `_getType()`を呼ぶ。
    1. `getParsedType()` を呼ぶ。
        1. `typeof [input.data](http://input.data)` がbooleanの場合、`ZodParsedType.boolean`を返す。
    2. `getParsedType()`の返り値をそのまま`_parse()`に返す。
3. 返り値が`ZodParsedType.boolean`ではない場合、問題などをcontextに詰めて、INVALIDを返す。
4. `status: valid` として、値を返す。

あとの処理は前述した通りのため、このような流れで最初に提示したコードが動いていることがわかった。

## 感想

zodはprod buildで他に依存しているものがなく、かなりとっつきやすかった。

cloneしてライブラリ側を変えてテストすることも容易で、最初に見るにはかなりいいOSSだと思いました。
