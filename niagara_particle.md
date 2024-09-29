# Niagaraでマテリアルパラメータを動的に指定する方法

## 概要
UE5サンプルのSampleTopDownでは、地面をクリックするとキャラクターをその地点に移動することができる。  
クリックした際にその地点に矢印が表示されるのだが、矢印が時間経過とともに動くアニメーションがついている。(Cursorと命名されている)  
このアニメーションはNiagaraから専用マテリアルのパラメータに値を渡すことで実現されていた。  
その方法を調査して、簡単な矢印エフェクトを再現してみる。  
![image](https://github.com/user-attachments/assets/1055f2b7-5758-48ca-8544-33e0596d67a8)


## 調査
Cursorは下記の要素で構成されている  
![image](https://github.com/user-attachments/assets/a9d7b776-6cc6-4562-8d69-f74be9fe18be)

### T_Arrow : 矢印のテクスチャ
UVアニメーションのため空白領域がある

### SM_CursorMesh : 矢印テクスチャを貼るメッシュ
- メッシュ:
  - ![image](https://github.com/user-attachments/assets/51ed8c20-a15a-4eb0-94ed-f26c73331c0e)
- 3枚の曲面の板は下記のUV範囲が設定されていて、テクスチャを貼った際にちょうど矢印が見えないようになっている。
  - ![image](https://github.com/user-attachments/assets/9b808680-2e81-4025-94ce-4cb7b9a99340)
- UV.yにオフセットが適用されると、下記のように矢印が見えるようになりアニメーションも行える
  - ![image](https://github.com/user-attachments/assets/0ecc1ff0-0919-4952-a00f-bc9681d1860f)
   - ![image](https://github.com/user-attachments/assets/27555603-6332-4982-8571-d3c609f707e2)

### M_Cursor : UVパラメータを持ったマテリアル
![image](https://github.com/user-attachments/assets/143d2194-4044-4d2d-9258-633f1a4e5182)
- 公開パラメータ`TexCoordY`がメッシュの頂点UV座標(TexCoord)に加算されて矢印テクスチャがSampleするマテリアル
  - 公開パラメータは「右クリック」→「パラメータ」→「Vector Parameter」で作れる様子
- 色は固定値が入っている

### FX_Cursor : パーティクル設定
（一番調査に難儀したのはこのパーティクル設定で、どの設定が矢印エフェクトのために指定され既定値ではなくなってるのか見つけるのが大変だった。）  

パーティクルの寿命は、0.7秒に指定されている。
![image](https://github.com/user-attachments/assets/1ef3fc54-ff0a-45c4-9dd9-ef151b153ffe)

最も重要なのはここ。  
メッシュを使うのでメッシュレンダラーとSM_CursorMeshが指定されている。（板ポリの場合はSpriteレンダラーになる）  
「バインディング」→「Material Parameter」→「Attribute Binding」で、先程のマテリアルの公開パラメータ「TexCoord」に「NormalizedLoopAge」(1.0に正規化されたパーティクルの年齢)が指定されている。  
この設定によって、UV座標の加算値が0.0から1.0に変化し矢印や動いているように見えるアニメーションが実現されている。  
![image](https://github.com/user-attachments/assets/298a06b6-4f6b-4229-9110-e6b6f9ff57e0)


## トライアル.1 : 色も時間経過とともに変化させるには？
色をカラフルに変化させるため、パーティクルの年齢を都度更新しながら最新の値を渡して反映させたい。  
このような状況ではテクスチャ座標と同じように、「Attribute Binding」を使うらしい。  

1. M_Cursorに色相を設定する用の公開パラメータ`ColorHue`を作成する
  - ノードは下記のように繋いで、`HSV=(ColorHue, 1.0, 1.0)`となるようにした
  - ![image](https://github.com/user-attachments/assets/fdca6e73-1623-4046-8760-81d5513e4e1e)

2. FX_CursorからM_CursorにNormalizedLoopAgeが渡るようにする
  - ![image](https://github.com/user-attachments/assets/640565ed-c878-4c3b-86ef-43ef18497271)

3. できた

## トライアル.2 : パーティクル毎に決めたランダムな色を与えるには？
今度はパーティクルの寿命中に都度更新する必要はなく、パーティクル生成時に色を1回決めてその値を使い続ければ良い。  
その値はどう決めてどうやったら利用できるのか？  
色や座標はパーティクル専用の属性があるようなので、それを利用する。  

1. FX_Cursorからマテリアルの`ParticleColor`をどう設定するか決める
  - ![image](https://github.com/user-attachments/assets/86adaaf7-d94d-4bb7-8594-5b02c9df401d)

2. M_Cursorで今までの色ノードの代わりに`ParticleColor`を繋ぐ
  - ![image](https://github.com/user-attachments/assets/e5467fcd-cab3-4aa1-bee0-bfde7dd6250a)

3. できた

## トライアル.3 : なんか代わりのエフェクトを作ってみる
