---
title: "Github API を呼び出す JavaScript Action using TypeScript のテストを書く Tips"
emoji: "💡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "githubactions", "jest"]
published: true
---

[Create a JavaScript Action using TypeScript](https://github.com/actions/typescript-action) を元に TypeScript で JavaScript Action を書いた時に、テストの方法が元のリポジトリだけでは分からなかったので記録する。

# テストする対象のコード
https://github.com/nokamoto/merge-conflict-action

```typescript
import * as github from "@actions/github";

export async function listPulls(){
  const octokit = github.getOctokit(token);

  return octokit.rest.pulls
    .list({
      owner: owner,
      repo: repo,
    })
}
```

`@actions/github` で Rest API を呼び出している部分をモックしたい。

調べた感じ [octokit/fixtures](https://github.com/octokit/fixtures) も使えるかもしれなかったが、オーバーキルに見えたので [jest](https://github.com/facebook/jest) で出来ないかを試していた。

# モック化するテストコード
```typescript
  const setup = (list: jest.Mock) => {
    const getOctokit = jest.fn().mockImplementation(() => {
      const { GitHub } = jest.requireActual("@actions/github/lib/utils");
      return {
        ...GitHub,
        rest: {
          pulls: { list: list },
        },
      };
    });

    jest.spyOn(github, "getOctokit").mockImplementation(getOctokit);

    return [getOctokit];
  };
```

`getOctokit` を　`spyOn` でモック化する。

```typescript
const { GitHub } = jest.requireActual("@actions/github/lib/utils");
```

`GitHub` を与えていないと tsc がコンパイルエラーを吐いていたので試行錯誤していた。

# モックを使うテストコード
```typescript
    const list = jest.fn().mockImplementation(() =>
      Promise.resolve({
        data: [],
      })
    );

    const [getOctokit] = setup(list);
```

正常系のレスポンスをテストする場合は `Promise.resolve` を返す実装を与える。

```typescript
    const list = jest
      .fn()
      .mockImplementation(() => Promise.reject(new Error("failed")));

    const [getOctokit] = setup(list);
```

異常系のレスポンスをテストする場合は `Promise.reject` を返す実装を与える。

# 気になったこと

`Promise.resolve` で返す型が適当なので実行時エラーが起きるのが気になる。ただし、型で制約が掛かりすぎてもテストを書くのが面倒になると思うので、仕方ないかもしれない。
