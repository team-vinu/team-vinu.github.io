---
layout: single
title: Godot4.0(Beta3)でUV Scrolling ShaderをVisual Shaderで実装した話
author: anosatsuk124
tags: Godot
toc: true
---

## 今回実装したShader

<video width="800" src="/assets/videos/2022-11-16-anosatsuk124/uv-shader-overview.mp4" controls></video>

### 機能

- FPSを指定できる
- Emissionで色をOverlayできる
- 再生速度の調整ができる
- Alphaが効く

## 実装における要件と経緯

- FPSを指定できるようにする必要がある
    - 私たちが現在作成しているゲームでは3D空間で実装されているが、Post-ProcessingやFPSを指定することでドット絵風のグラフィックデザインを実現しているため
- ノイズやパーティクル用のテクスチャを使えるようにAlpha値に対応したい
    - テクスチャを描けるスキルに乏しく、有料アセットにも手を出しづらい
    - Godot用のアセットは少く、使いまわせる単純なアセットで実装したかった
- 上述の理由と同様で単色+Alpha画像に色をOverlayすることで色を変えたかった
- Visual Shaderで実装したかった
    - Godot4.0ではShader言語がGodot3.x以前と変っている(レンダリングエンジンがVulkanに変っており、Godot3.xのShaderを修正しただけでは動かないことがある)のでドキュメントに乏しく、実装することの困難さがあった
    - そもそもShader言語を触ったこともなく、グラフィックスの知識にも乏しい
- アニメーションとして実装するのは余計なCPU負荷や手間もかかるため控えたい

## 実装の解説

### FPSパラメータの実装

UVにはTimeをInputとして入力した後の処理が大変でした。おそらく色々と間違っている実装の可能性もありますので、グラフィックスなどに詳しい方はコメントで指摘お願いします。

まず時間をfloatの値で除算したものをSpeedを適用する(値に比例して速度が下がる)

↓

その値をFractで小数部分を取りだしたものに、FPS値を乗算する

↓

その値をFloorにかけて整数値にしたものをまたFPS値で除算したものをUVに加算する

という処理を行なってFPSとスピードの調整を行いました(が今改めて書き下すとFracのあとにFloorとかバカらしい気もします)。

![visual-shader1](/assets/images/2022-11-16-anosatsuk124/visual-shader1.png)

### Texture/Emission/Alphaの実装

こちらの処理は単純です。Texture2Dに先程のUVを接続し、Texture2DからAlbedoとAlphaに接続しているだけです。
本来はTexture2DからAlphaの伸ばすだけではいけないはず(はず)ですが、今回は基本単色の単純なテクスチャしか使う予定がないので問題はないはずです。

Emissionは単純に色をEmissionに接続しているだけです。

![visual-shader2](/assets/images/2022-11-16-anosatsuk124/visual-shader2.png)

## まとめ

今回Visual ShaderでShaderを実装してみて、Shaderは怖くない(怖いかもしれないけど)という感覚に陥いりました。

またレンダパイプラインなど未知の概念も多い(Godot4.0はCustom Rendering Pipelineに対応についてのIssueがMilestoneにありました)ですが、地道に勉強を重ねてShaderの気持ちへの理解を深めたいですね。

最後に(おそらく)Godot4.0の新機能であるVisual ShaderからShader言語のコードを生成できる機能を用いてShaderをコピペ用に貼っておくのでご自由にお使いください。`CC0 License`とします。


```
/*
Creative Commons Legal Code

CC0 1.0 Universal

    CREATIVE COMMONS CORPORATION IS NOT A LAW FIRM AND DOES NOT PROVIDE
    LEGAL SERVICES. DISTRIBUTION OF THIS DOCUMENT DOES NOT CREATE AN
    ATTORNEY-CLIENT RELATIONSHIP. CREATIVE COMMONS PROVIDES THIS
    INFORMATION ON AN "AS-IS" BASIS. CREATIVE COMMONS MAKES NO WARRANTIES
    REGARDING THE USE OF THIS DOCUMENT OR THE INFORMATION OR WORKS
    PROVIDED HEREUNDER, AND DISCLAIMS LIABILITY FOR DAMAGES RESULTING FROM
    THE USE OF THIS DOCUMENT OR THE INFORMATION OR WORKS PROVIDED
    HEREUNDER.
 */

shader_type spatial;
uniform float Speed;
uniform int FPS;
uniform sampler2D Albedo_Texture;
uniform vec4 Emission_Color : source_color;



void fragment() {
// Input:7
	float n_out7p0 = TIME;


// FloatParameter:12
	float n_out12p0 = Speed;


// FloatOp:9
	float n_out9p0 = n_out7p0 / n_out12p0;


// FloatFunc:8
	float n_out8p0 = fract(n_out9p0);


// IntParameter:16
	int n_out16p0 = FPS;


// FloatOp:14
	float n_out14p0 = n_out8p0 * float(n_out16p0);


// FloatFunc:13
	float n_out13p0 = floor(n_out14p0);


// FloatOp:15
	float n_out15p0 = n_out13p0 / float(n_out16p0);


// Input:10
	vec2 n_out10p0 = UV;


// VectorOp:11
	vec2 n_out11p0 = vec2(n_out15p0) + n_out10p0;



	vec4 n_out3p0;
// Texture2D:3
	n_out3p0 = texture(Albedo_Texture, n_out11p0);


// ColorParameter:5
	vec4 n_out5p0 = Emission_Color;


// Output:0
	ALBEDO = vec3(n_out3p0.xyz);
	ALPHA = n_out3p0.x;
	EMISSION = vec3(n_out5p0.xyz);


}
```