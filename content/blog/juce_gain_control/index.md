---
title: "JUCE入門: ゲインコントロールの実装"
date: 2020-04-29T07:15:50.256Z
description: "JUCE入門: ゲインコントロールの実装"
---

今回は、オーディオアプリケーション開発ライブラリである[JUCE](https://juce.com/)上で[この動画](https://www.youtube.com/watch?v=Bw_OkHNpj1M&t=2321s)を参考に簡単なゲインコントロールを実装していきます。

## プロジェクトの作成

まず、今回のプロジェクトの雛形を作成します。Projucerを起動して、"Create New Project"の画面から"Audio Plug-In"を選択し、次の画面で適当な名前（`gainTutorial`など）をつけて"Create..."をクリックします。

![create new project](/img/juce_gain_control/new_project.png)
![give name](/img/juce_gain_control/give_name.png)

すると、以下のような画面になるので、XcodeのアイコンをクリックしてXcodeを起動します。
![launch xcode](/img/juce_gain_control/launch_xcode.png)

Xcodeを起動すると以下のような画面が出ます。
左側のProject Managerを見てみましょう。`gainTutorial/Source`の下に4つのファイル、`PluginProcessor.(h/cpp)`、`PluginEditor.(h/cpp)`があります。
前者は主に信号処理のコードを記述し、後者は主にGUIのコードを記述します。
![xcode startup](/img/juce_gain_control/xcode_startup.png)

## DAWを用いたデバッグ環境の構築

DAWを使ってプラグインの動作確認を行えるように環境構築を行います。
Xcodeのメニューバーから"Product > Scheme > Edit Scheme"を選択します。
![xcode startup](/img/juce_gain_control/edit_scheme.png)

"Excecutable"から"Other..."を選択し、デバッグに使いたいDAWの実行ファイルを選択します。今回はLogic Pro Xを選択しました。
Debug Executable"のチェックを外します。

![select logic](/img/juce_gain_control/select_logic.png)

左上のビルドボタン（もしくはCMD+R）でプロジェクトをビルドすると、自動的に選択したDAWが立ち上がります。DAW側からプロジェクト名と同じ名前のプラグインが呼び出せ、以下のようなウィンドウが立ち上がれば成功です。

![select plugin](/img/juce_gain_control/select_plugin.png)
![plugin window](/img/juce_gain_control/plugin_window.png)

