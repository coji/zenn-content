---
title: Google Gemini APIで複数話者テキスト音声合成（TTS）を実現！会話を自然な声で生成
emoji: 🎤
type: tech
topics:
  - generative-ai
  - google-gemini
  - text-to-speech
  - typescript
published: false # 下書きとして公開する場合は true に変更
---

## はじめに

近年、AI技術の進化は目覚ましく、特にテキスト音声合成（Text-to-Speech: TTS）は、その品質と表現力が飛躍的に向上しています。これまでは単一話者の音声合成が主流でしたが、Google Gemini APIを使うことで、なんと**複数話者の会話を、それぞれの話者に異なる声色を割り当てて自然に生成**できるようになりました。

本記事では、Google Gemini APIの新しいTTS機能 `gemini-2.5-flash-preview-tts` モデルとTypeScriptを使用して、複数話者での会話を音声ファイルとして出力する実装方法を解説します。ポッドキャスト、オーディオブック、キャラクター会話のプロトタイプなど、様々なユースケースでの活用が期待できます。

この記事を読めば、以下のことができるようになります。

* Google Gemini APIで複数話者TTSを行うための基本的な設定とコードを理解できる。
* 提供されたサンプルコードを動かし、自分の会話を音声化できる。
* 話者の声色をカスタマイズする方法を知ることができる。

## Google Gemini APIとは？

Google Gemini APIは、Googleが開発した強力なマルチモーダルAIモデル「Gemini」にアクセスするためのAPIです。テキスト、画像、音声など、多様な形式の情報を理解し、生成することができます。今回注目するのは、その中でも特に新しい「音声生成（Text-to-Speech）」機能です。

Gemini APIのTTS機能は、単一話者だけでなく、最大2話者までの複数話者の会話を合成できるのが特徴です。また、プロンプトで話者のスタイル、アクセント、ペース、トーンを制御できる「Controllable TTS」という機能も備わっています。 [1]

本記事で使用するモデルは `gemini-2.5-flash-preview-tts` で、これは高速で効率的な音声合成に適しています。 [1]

## 複数話者音声合成の魅力

なぜ複数話者音声合成が便利なのでしょうか？

1. **リアルな会話の再現**: 複数のキャラクターが登場するシナリオや会話において、それぞれの話者に異なる声色を割り当てることで、より自然で臨場感のある音声コンテンツを作成できます。
2. **コンテンツ制作の効率化**: ポッドキャストやオーディオブックの試作、研修用コンテンツ、ゲームのセリフなど、手動で録音する手間を省き、迅速に音声コンテンツを生成できます。
3. **表現の多様性**: 会話の参加者ごとに異なる声色や話し方を設定することで、より豊かな表現が可能になります。

## プロジェクトのセットアップ

今回使用するサンプルコードのリポジトリは以下の構造を持っています。

```

├── .gitignore
├── main.ts
├── output.wav
├── package.json
├── pnpm-lock.yaml
├── pnpm-workspace.yaml
└── tsconfig.json

```

このリポジトリをベースに、必要な環境構築を進めましょう。

### 1. APIキーの取得

Google Gemini APIを利用するには、Google Cloud ProjectでAPIキーを発行する必要があります。

1. **Google Cloud Console**にアクセスし、新しいプロジェクトを作成するか、既存のプロジェクトを選択します。
2. ナビゲーションメニューから「APIとサービス」>「ライブラリ」を選択し、「**Generative Language API**」を検索して有効にします。
3. 「APIとサービス」>「認証情報」に進み、「認証情報を作成」>「APIキー」を選択して新しいAPIキーを発行します。
4. 発行されたAPIキーを安全な場所にメモしておきましょう。

### 2. リポジトリのクローンと依存関係のインストール

プロジェクトをクローンし、依存関係をインストールします。pnpm を使用しているので、事前にインストールしておきましょう。

```bash
# リポジトリをクローン
git clone https://github.com/coji/tts-test.git
cd tts-test

# 依存関係をインストール
pnpm install
```

### 3. `.env` ファイルの設定

取得したAPIキーを環境変数として設定します。プロジェクトのルートディレクトリに `.env` ファイルを作成し、以下のように記述してください。

```dotenv
# .env
GOOGLE_GENERATIVE_AI_API_KEY=YOUR_API_KEY_HERE
```

`YOUR_API_KEY_HERE` の部分を、先ほど取得した実際のAPIキーに置き換えてください。`.gitignore` に `.env` が含まれているため、誤ってコミットされる心配はありません。

## コードの解説 (`main.ts`)

それでは、核となる `main.ts` ファイルのコードを見ていきましょう。

```typescript
// main.ts
import { GoogleGenAI, type SpeechConfig } from '@google/genai';
import * as wav from 'wav'; // WAVファイル書き出し用ライブラリ

const GEMINI_API_KEY = process.env.GOOGLE_GENERATIVE_AI_API_KEY; // 環境変数からAPIキーを読み込み

const ai = new GoogleGenAI({ apiKey: GEMINI_API_KEY }); // GoogleGenAIクライアントの初期化

async function main() {
  const prompt = `TTS the following conversation between Joe and Jane:
Joe: やあ、元気？
Jane: うん、元気だよ!
Joe: 今日は何をしているの？
Jane: 今日は友達と遊びに行く予定だよ。`; // 音声合成する会話プロンプト

  const response = await ai.models.generateContent({
    model: 'gemini-2.5-flash-preview-tts', // 使用するTTSモデル
    contents: prompt, // プロンプトをモデルに渡す
    config: {
      responseModalities: ['AUDIO'], // レスポンスの形式を「AUDIO」に指定
      speechConfig: {
        multiSpeakerVoiceConfig: { // 複数話者設定
          speakerVoiceConfigs: [ // 各話者の音声設定
            {
              speaker: 'Joe', // プロンプト内の話者名と一致させる
              voiceConfig: {
                prebuiltVoiceConfig: {
                  voiceName: 'Kore', // Joeに割り当てる声色
                },
              },
            },
            {
              speaker: 'Jane', // プロンプト内の話者名と一致させる
              voiceConfig: {
                prebuiltVoiceConfig: {
                  voiceName: 'Puck', // Janeに割り当てる声色
                },
              },
            },
          ],
        },
      } satisfies SpeechConfig, // 型アサーション
    },
  });
  const audio = response.candidates?.?.content?.parts?.?.inlineData?.data; // 生成された音声データを取得
  if (audio) {
    const audioBuffer = Buffer.from(audio, 'base64'); // base64エンコードされたデータをBufferに変換
    await waveFile('output.wav', audioBuffer); // WAVファイルとして保存
  }
}

// BufferをWAVファイルとして保存するヘルパー関数
const waveFile = async (
  filename: string,
  pcm: Buffer,
  channels = 1,
  sampleRate = 24000
) => {
  return new Promise<void>((resolve, reject) => {
    const wavWriter = new wav.FileWriter(filename, {
      channels,
      bitDepth: 16,
      sampleRate,
    });

    wavWriter.write(pcm);
    wavWriter.end();

    wavWriter.on('finish', () => {
      console.log(`Wrote ${filename}`);
      resolve();
    });

    wavWriter.on('error', (err) => {
      console.error(`Error writing ${filename}:`, err);
      reject(err);
    });
  });
};

main();
```

### 主要なポイント

* **`GoogleGenAI` の初期化**:

    ```typescript
    import { GoogleGenAI } from '@google/genai';
    const ai = new GoogleGenAI({ apiKey: GEMINI_API_KEY });
    ```

    `@google/genai` ライブラリを使用してクライアントを初期化します。APIキーは環境変数から安全に読み込んでいます。
* **`generateContent` メソッドの呼び出し**:

    ```typescript
    const response = await ai.models.generateContent({
      model: 'gemini-2.5-flash-preview-tts',
      contents: prompt,
      config: { /* ... */ }
    });
    ```

    Gemini APIの `generateContent` メソッドを呼び出します。ここで重要なのは `model` に `gemini-2.5-flash-preview-tts` を指定することです。このモデルは音声生成に特化しています。
* **`responseModalities: ['AUDIO']`**:
    レスポンスの形式を音声（オーディオ）として受け取ることを明示的に指定します。
* **`speechConfig` と `multiSpeakerVoiceConfig`**:
    複数話者音声合成の肝となる設定です。
    `multiSpeakerVoiceConfig` 内の `speakerVoiceConfigs` 配列に、各話者の設定をオブジェクトとして渡します。
  * `speaker`: これは**プロンプト内で使用する話者名**と完全に一致させる必要があります。コード例では `Joe` と `Jane` がこれに当たります。
  * `voiceConfig.prebuiltVoiceConfig.voiceName`: 各話者に割り当てる声色を指定します。Google Gemini APIは多くの[プリビルド音声](https://ai.google.dev/gemini-api/docs/speech-generation#voice-options)を提供しており、例えば以下のようなものが利用可能です。 [1]
    * `Zephyr` -- Bright
    * `Puck` -- Upbeat
    * `Charon` -- Informative
    * `Kore` -- Firm
    * `Fenrir` -- Excitable
    * `Leda` -- Youthful
    * `Orus` -- Firm
    * `Aoede` -- Breezy
    * `Callirhoe` -- Easy-going
    * `Autonoe` -- Bright
    * `Enceladus` -- Breathy
    * `Iapetus` -- Clear
    * `Umbriel` -- Easy-going
    * `Algieba` -- Smooth
    * `Despina` -- Smooth
    * `Erinome` -- Clear
    * `Algenib` -- Gravelly
    * `Rasalgethi` -- Informative
    * `Laomedeia` -- Upbeat
    * `Achernar` -- Soft
    * `Alnilam` -- Firm
    * `Schedar` -- Even
    * `Gacrux` -- Mature
    * `Pulcherrima` -- Forward
    * `Achird` -- Friendly
    * `Zubenelgenubi` -- Casual
    * `Vindemiatrix` -- Gentle
    * `Sadachbia` -- Lively
    * `Sadaltager` -- Knowledgeable
    * `Sulafar` -- Warm

        これらの声色は [AI Studio](https://aistudio.google.com/) で試聴することもできます。 [1]
* **WAVファイルへの書き出し**:
    生成された音声データはBase64エンコードされた形式で返されるため、`Buffer.from(audio, 'base64')` でデコードし、`wav` ライブラリの `FileWriter` を使って `output.wav` として保存しています。

## 実行と結果

プロジェクトの準備が整ったら、以下のコマンドで実行できます。

```bash
pnpm start
```

実行が完了すると、プロジェクトのルートディレクトリに `output.wav` という音声ファイルが生成されます。

### 生成された音声の例

このコード例で生成される音声は、以下の会話に対応しています。

**会話テキスト:**

```
Joe: やあ、元気？
Jane: うん、元気だよ!
Joe: 今日は何をしているの？
Jane: 今日は友達と遊びに行く予定だよ。
```

**音声ファイル:**

<iframe src="https://raw.githubusercontent.com/coji/tts-test/main/output.wav"></iframe>

（[GitHub上のoutput.wav](https://raw.githubusercontent.com/coji/tts-test/main/output.wav) でも直接聞くことができます。）

このように、JoeとJaneのセリフがそれぞれ異なる声色で合成されていることがわかります。

## 発展的な内容とカスタマイズ

このサンプルをベースに、さらに様々なカスタマイズが可能です。

* **会話の内容の変更**:
    `main.ts` ファイルの `prompt` 定数を自由に変更し、色々な会話を試してみてください。

    ```typescript
    const prompt = `TTS the following conversation between SpeakerA and SpeakerB:

SpeakerA: こんにちは！
SpeakerB: こんにちは、今日は良い天気ですね。`;
    ```
    ただし、`speaker` の設定とプロンプト内の話者名を一致させるのを忘れないでください。

* **声色の変更**:
    `speakerVoiceConfigs` 内の `voiceName` を、[前述のリスト](https://ai.google.dev/gemini-api/docs/speech-generation#voice-options) から別のものに変更してみましょう。会話の雰囲気に合わせて最適な声色を見つけることができます。

* **会話スタイルの制御**:
    プロンプトに話し方の指示を追加することで、声のスタイルをより細かく制御できます。 [1]
    例:

    ```typescript
    const prompt = `Make Joe sound tired and bored, and Jane sound excited and happy:

Joe: やあ、元気？
Jane: うん、元気だよ!`;
    ```
    話者の声色とプロンプトの指示を組み合わせることで、さらに表現豊かな音声を生成できます。

* **ストリーミング出力**:
    今回のサンプルでは一度に全ての音声データを取得してファイルに保存していますが、Gemini APIはストリーミング出力もサポートしています。これにより、音声生成が開始され次第、リアルタイムで音声を再生することが可能になります。 [1]

## まとめ

本記事では、Google Gemini APIの強力なTTS機能を使って、複数話者の会話を自然な声で合成する方法を解説しました。`gemini-2.5-flash-preview-tts` モデルと `multiSpeakerVoiceConfig` を活用することで、ポッドキャスト、オーディオブック、キャラクターボイスなど、多様な音声コンテンツの可能性が広がります。

ぜひご自身のプロジェクトでこの技術を試してみてください。

---
[1] Speech generation (Text-to-speech) | Gemini API | Google AI for Developers. `https://ai.google.dev/gemini-api/docs/speech-generation#controllable`
