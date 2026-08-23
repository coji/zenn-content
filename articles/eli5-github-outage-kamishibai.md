---
title: "GitHubが8時間止まった日を、5歳でもわかる紙芝居にした"
emoji: "🚪"
type: "idea"
topics: ["claudecode", "claude", "github", "sre", "ai"]
published: false
---

## これはなに？

Anthropicの社内で使われているという **ELI5 Skill**（Explain Like I'm 5、「5歳児に説明するように」）がXで話題になっていました。
中身を見にいくと、本文はたった1行です。

> Explain like I'm someone who knows nothing about this topic, using a HTML artifact with big pictures and few words.

「大きい絵と少ない言葉のHTMLで説明して」。それだけ。

ちょうど1週間前の2026年8月17日、GitHubは約8時間止まる大規模障害を起こしました。
公式ポストモーテムも出ていますが、autoscaling、sidecar、リトライストームと、正直ひと目では頭に入ってきません。
そこでこの1行スキルで、障害の顛末を紙芝居にしてみました。
この記事はそのスクショを並べただけです。スクロールするだけで、あの日なにが起きたかわかります。

## 紙芝居「GitHubが8時間とまった日」

![タイトル: GitHubが8時間とまった日](/images/eli5-github-outage/scene1.png)

![お客さんがいままででいちばん来た](/images/eli5-github-outage/scene2.png)

![入口の係さんが手いっぱい](/images/eli5-github-outage/scene3.png)

![たすけロボットは反対を見ていた。だからたすけが来なかった](/images/eli5-github-outage/scene4.png)

![ドアがぜんぶつまって、お店ごとストップ](/images/eli5-github-outage/scene5.png)

![「もういっかい！」でもっとたいへんに](/images/eli5-github-outage/scene6.png)

![いったんしめて、深呼吸。8時間後、お店はなおった](/images/eli5-github-outage/scene7.png)

おしまい。

## ほんとうは何が起きたか

たとえ話と実際の対応はこうなっています。

| 紙芝居 | 実際 |
| --- | --- |
| お客さんがいちばん来た | トラフィックが過去最高（月間コミットは4月の14億から8月は29億へ倍増） |
| 入口の係さんが手いっぱい | Central USデータセンターのIstio sidecarが同時処理数の上限に到達 |
| たすけロボットは反対を見ていた | autoscalerが本体サービスの負荷だけを監視していて、sidecarの限界を見ていなかった |
| ドアがぜんぶつまった | 詰まりがHAProxy 4ノードのフロー上限に波及し、認証が劣化して全面障害に |
| 「もういっかい！」 | VS Codeの潜在リトライバグでCopilotのトークン要求が約10倍（7-9K→70-100K RPS）に増幅 |
| いったんしめて深呼吸 | HAProxyを一時停止すると即座に広範囲が回復。403で受付を抑えて段階的に再開 |

障害の核心が「autoscalerが、いちばん詰まっていた場所を監視対象にしていなかった」ことだと、紙芝居のほうが先に腹落ちしませんか。
私はこの図解を作ってはじめて、ポストモーテムの文章がすっと読めるようになりました。

詳細は公式ポストモーテムをどうぞ。

https://github.blog/news-insights/company-news/the-august-17-outage-and-the-work-ahead/

## つくりかた

やったことは2つだけです。

まず、ELI5 Skillの1行プロンプトに、ポストモーテムの内容を渡して図解HTMLを作らせました。
スキルの原文は[anthropics/claude-plugins-community](https://github.com/anthropics/claude-plugins-community/blob/main/eli5/skills/eli5/SKILL.md)にあります。
Claude Codeなら `claude plugin install eli5@claude-community` で入ります。
本文が1行のプロンプトなので、ChatGPTやCursorに貼ってもそのまま動きます。

```text
Explain like I'm someone who knows nothing about this topic, using a HTML artifact with big pictures and few words.

Topic: <説明してほしいトピック>
```

コツは、big pictures and few wordsの精神どおり「1画面1枚の絵と一言だけ」に寄せることです。

次に、できたHTMLをそのままURLにして共有しました。
私がひとりで開発しているArtifact Shareを使うと、コマンド1発で済みます。

```bash
npx --yes @artifactshare/cli share ./github-outage-eli5.html --visibility link --no-link-expiry
```

できあがったページがこちらです。
スクロールで紙芝居が進み、末尾にELI5プロンプトのコピーボタンを置いてあります。

https://artifactshare.com/a/l2z2809bsp
