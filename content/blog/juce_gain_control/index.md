---
title: "JUCE入門: レベルメータの実装"
date: 2020-04-29T07:15:50.256Z
description: "JUCE入門: レベルメータの実装"
---

今回は、オーディオアプリケーション開発用C++ライブラリである[JUCE](https://juce.com/)上で[この動画](https://www.youtube.com/watch?v=Bw_OkHNpj1M&t=2321s)を参考に、以下の画像にあるような簡単なレベルメータを実装していきます。

![final slider](/img/juce_gain_control/slider_final.png)

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

そして、"Debug Executable"のチェックを外します。

![select logic](/img/juce_gain_control/select_logic.png)

左上のビルドボタン（もしくはCMD+R）でプロジェクトをビルドすると、自動的に選択したDAWが立ち上がります。DAW側からプロジェクト名と同じ名前のプラグインが呼び出せ、以下のようなウィンドウが立ち上がれば成功です。

![select plugin](/img/juce_gain_control/select_plugin.png)
![plugin window](/img/juce_gain_control/plugin_window.png)

## スライダーの配置

まずは操作系であるスライダーを実装していきます。
`PluginProcessor.h`にある`GainTutorialAudioProcessorEditor`に`Slider`クラスのprivateメンバ変数`mGainSlider`を追加します。

```cpp
// PluginEditor.h

class GainTutorialAudioProcessorEditor  : public AudioProcessorEditor
{
public:
    GainTutorialAudioProcessorEditor (GainTutorialAudioProcessor&);
    ~GainTutorialAudioProcessorEditor();

    //==============================================================================
    void paint (Graphics&) override;
    void resized() override;

private:
    Slider mGainSlider;

    GainTutorialAudioProcessor& processor;

    JUCE_DECLARE_NON_COPYABLE_WITH_LEAK_DETECTOR (GainTutorialAudioProcessorEditor)
};
```

次にスライダーの見た目を作る処理を行う、[`Slider`](https://docs.juce.com/master/classSlider.html)クラスのメンバー関数を`GainTutorialAudioProcessorEditor`のコンストラクタ内で呼び出します。

具体的には、[`setSliderStyle()`](https://docs.juce.com/master/classSlider.html#a6b6917dd3753c7552778977733f0b9ef)というメンバー関数でスライダーの種類を決めます。

この関数は列挙体[`Slider::SliderStyle`](https://docs.juce.com/master/classSlider.html#af1caee82552143dd9ff0fc9f0cdc0888)の列挙値を引数に取ります。

今回は縦型のスライダーが欲しいので、列挙値`LinearVertical`を選択します。

```cpp
// PluginEditor.cpp

GainTutorialAudioProcessorEditor::GainTutorialAudioProcessorEditor (GainTutorialAudioProcessor& p)
    : AudioProcessorEditor (&p), processor (p)
{
    mGainSlider.setSliderStyle (Slider::SliderStyle::LinearVertical)

    setSize (400, 300);
}
```

次に、スライダーの範囲と間隔を`setRange()`メンバー関数、初期値を`setValue()`メンバー関数で設定します。

今回は最小値0、最大値1、間隔0.01に設定し、初期値は0.5とします。

```cpp
// PluginEditor.cpp

GainTutorialAudioProcessorEditor::GainTutorialAudioProcessorEditor (GainTutorialAudioProcessor& p)
    : AudioProcessorEditor (&p), processor (p)
{
    mGainSlider.setSliderStyle(Slider::SliderStyle::LinearVertical);
    mGainSlider.setRange(0.0f, 1.0f, 0.01f);
    mGainSlider.setValue(0.5f);
    
    setSize (400, 300);
}
```

デフォルトでは、スライダーの設定値を表示するテキストボックスが左側に表示されるので、
[`setTextBox()`](https://docs.juce.com/master/classSlider.html#a5bc748a21e72fe14153bc9fe5ac03e77)メンバー関数でスライダーの下部に表示されるようにします。

`setTextBox()`の第1引数に列挙体[`Slider::TextEntryBoxPosition
`](https://docs.juce.com/master/classSlider.html#ab6d7dff67151c029b9cb53fc40b4412f)の列挙値を指定します。
今回はスライダーの下部に欲しいので`TextBoxBelow`を選択します。

また、第2引数`isReadOnly`を`true`に設定して読み取り専用にし、第3, 第4引数にはテキストボックスの幅と高さを設定します。

```cpp
// PluginEditor.cpp

GainTutorialAudioProcessorEditor::GainTutorialAudioProcessorEditor (GainTutorialAudioProcessor& p)
    : AudioProcessorEditor (&p), processor (p)
{
    mGainSlider.setSliderStyle(Slider::SliderStyle::LinearVertical);
    mGainSlider.setTextBoxStyle(Slider::TextEntryBoxPosition::TextBoxBelow, true, 50, 20);
    mGainSlider.setRange(0.0f, 1.0f, 0.01f);
    mGainSlider.setValue(0.5f);
    
    setSize (400, 300);
}
```

そして、`addAndMakeVisible()`関数を使って`GainTutorialAudioProcessorEditor`に`mGainSlider`を子コンポーネントとして追加します。

ついでにウィンドウを縦長のスライダーに合わせて縦長にしておきます。

```cpp
// PluginEditor.cpp

GainTutorialAudioProcessorEditor::GainTutorialAudioProcessorEditor (GainTutorialAudioProcessor& p)
    : AudioProcessorEditor (&p), processor (p)
{
    mGainSlider.setSliderStyle(Slider::SliderStyle::LinearVertical);
    mGainSlider.setTextBoxStyle(Slider::TextEntryBoxPosition::TextBoxBelow, true, 50, 20);
    mGainSlider.setRange(0.0f, 1.0f, 0.01f);
    mGainSlider.setValue(0.5f);
    addAndMakeVisible(mGainSlider);
    
    setSize (200, 300);
}
```

このスライダーコンポーネントを実際に表示させるには、コンポーネントの表示位置およびサイズを`Component`クラス（`GainTutorialAudioProcessor`の基底クラス。[JUCE Documentation](https://docs.juce.com/master/classAudioProcessorEditor.html)のクラス図を参照）のメンバー関数、[`resized()`](https://docs.juce.com/master/classComponent.html#ad896183a68d71daf5816982d1fefd960)をオーバーライドして指定します。


この`resized()`メンバー関数はコンポーネントのサイズが変更されたときに呼び出される関数です。

`resized()`は[`setSize()`](https://docs.juce.com/master/classComponent.html#af7e0443344448fcbbefc2b3dd985e43f)関数がコンストラクタで呼ばれるときにも同時に呼ばれるので、プラグインを起動したときにも`resized()`が呼ばれてスライダーの表示位置とサイズが設定されます。

今回のはスライダーコンポーネントのサイズを幅100px、高さ150pxに指定します。

第1、第2引数にはそれぞれコンポーネントの左上の頂点のx、y座標を指定します（原点は親コンポーネントの左上の
頂点）。
スライダーコンポーネントの位置がウィンドウに対して中央に来るようにしたいので、以下の図のようなロジックで座標を決めます。

親コンポーネントの横幅と縦幅はそれぞれ`getWidth()`関数と`getHeight()`関数で取得できるので、それらを元に座標を計算します。

また、デフォルトで`paint()`メンバー関数に記述されている`Hello World!`を表示するコードも削除して、背景が真っ黒になるようにします。

![setBounds](/img/juce_gain_control/setBounds.svg)

```cpp
// PluginEditor.cpp

void GainTutorialAudioProcessorEditor::paint (Graphics& g)
{
    g.fillAll(Colours::black);
}

void GainTutorialAudioProcessorEditor::resized()
{
    mGainSlider.setBounds(getWidth() / 2 - 50, getHeight() / 2 - 75, 100, 150);
}
```

この時点でビルドして、DAWでプラグインを起動するとスライダーが真ん中に配置されているような見た目になるはずです。

![slider](/img/juce_gain_control/slider.png)

## 音量を変えてみる

スライダーと信号処理の紐付けを行う前に、簡単な信号処理でJUCEでの信号の扱い方を見ていきます。

JUCEでの信号処理は`PluginProcessor.(h/cpp)`に記述されている`AudioProcessor`クラスの派生クラス（今回は`GainTutorialAudioProcessor`）が担っています。

その中でも特に重要なのが、`processBlock()`メンバー関数で、信号処理のほとんどをこの関数内に書き込んでいくことになります。

`processBlock()`の中身は、コメントを除くと現時点で以下のようになっていると思います。

```cpp
// PluginProcessor.cpp

void GainTutorialAudioProcessor::processBlock (AudioBuffer<float>& buffer, MidiBuffer& midiMessages)
{
    ScopedNoDenormals noDenormals;
    auto totalNumInputChannels  = getTotalNumInputChannels();
    auto totalNumOutputChannels = getTotalNumOutputChannels();

    for (auto i = totalNumInputChannels; i < totalNumOutputChannels; ++i)
        buffer.clear (i, 0, buffer.getNumSamples());

    for (int channel = 0; channel < totalNumInputChannels; ++channel)
    {
        auto* channelData = buffer.getWritePointer (channel);
    }
}
```

今回はMIDIを扱わないので、`processBlock()`の第2引数である`midiMessages`については無視します。

関数内の最初のコード`ScopedNoDenormals noDenormals;`は一時的に[非正規化数](https://ja.wikipedia.org/wiki/%E9%9D%9E%E6%AD%A3%E8%A6%8F%E5%8C%96%E6%95%B0)（0にごく近い数）をCPU上で無効にする処理を行います。

次のコードでは、デバイスの入力チャンネル数と出力チャンネル数をそれぞれ取得します。

```cpp
auto totalNumInputChannels  = getTotalNumInputChannels();
auto totalNumOutputChannels = getTotalNumOutputChannels();
```

その次のforループでは、出力チャンネル数が入力チャンネル数よりも大きい場合に、余った出力チャンネルから余分な音がでないように、そのチャンネルのバッファ内のサンプルをすべて0にします。
```cpp
for (auto i = totalNumInputChannels; i < totalNumOutputChannels; ++i)
    buffer.clear (i, 0, buffer.getNumSamples());
```
例えば、入力チャンネル数が1で出力チャンネル数が2の場合、上のfor文は1ループ実行され、1つ目の出力チャンネルのバッファ内のすべてのサンプルが0になります。

一方、入力チャンネル数と出力チャンネル数が等しい場合には`i < totalNumOutputChannels`が`true`になることがないので、上のforループが実行されることはありません。

最後のforループの中で、実際に信号処理を記述していきます。

入力チャンネルごとに、対応する出力チャンネルの書き込み先のバッファの先頭アドレスを取得し、`channelData`ポインタに格納します。

`channelData`に格納されたアドレスを起点として、バッファ内のサンプル値を書き換えていきます。

```cpp
for (int channel = 0; channel < totalNumInputChannels; ++channel)
{
    auto* channelData = buffer.getWritePointer (channel);
}
```

まず簡単に、音量を0.1倍する処理を行ってみます。バッファ内の全サンプルに対して、0.1を乗算する処理を書きます。

```cpp
for (int channel = 0; channel < totalNumInputChannels; ++channel)
{
    auto* channelData = buffer.getWritePointer (channel);
    for (int sample = 0; sample < buffer.getNumSamples(); ++sample)
        channelData[sample] *= 0.1;
}
```

この状態でプロジェクトをビルドしてDAW上で適当な音源が流れている状態でプラグインをオンオフすると、音量の大小が切り替わることが確認できます。

しかし、このままだと音量調整ができないので、乗算する数をスライダーの値に対応するような変数に置き換える必要があります。

## スライダーに音量調整機能をもたせる


まず、音量の値を格納する変数を定義します。`GainTutorialAudioProcessor`クラスのpublicメンバー変数`mGain`を追加します。
privateではなくpublicであるのは、`mGainSlider`を保持する`GainTutorialAudioProcessorEditor`からアクセスされるからです。

`mGain`の初期値は、スライダーの初期値と同様に0.5に設定します。

```cpp
// PluginProcessor.h

class GainTutorialAudioProcessor  : public AudioProcessor
{
public:
    // .......
    
    float mGain { 0.5 };

private:
    // .......
};
```

`processBlock()`内の乗算処理も、`mGain`で置き換えておきます。

```cpp
for (int channel = 0; channel < totalNumInputChannels; ++channel)
{
    auto* channelData = buffer.getWritePointer (channel);
    for (int sample = 0; sample < buffer.getNumSamples(); ++sample)
        channelData[sample] *= mGain;
}
```

スライダーを使って`GainTutorialAudioProcessor`のメンバー変数である`mGain`の値を変更する橋渡し役になるのが、
`GainTutorialAudioProcessorEditor`のメンバー変数である`processor`です。

```cpp
class GainTutorialAudioProcessorEditor  : public AudioProcessorEditor,
{
public:
    // .........

private:
    // .........
    GainTutorialAudioProcessor& processor;

    JUCE_DECLARE_NON_COPYABLE_WITH_LEAK_DETECTOR (GainTutorialAudioProcessorEditor)
};
```

`processor`は`GainTutorialAudioProcessor`クラスの参照であり、`processor.mGain`のように`GainTutorialAudioProcessor`クラスの
メンバー変数を`GainTutorialAudioProcessorEditor`クラスから`processor`を介してアクセスできます。

よって、以下のような擬似コードの機能を持つコードをPluginEditor側で記述できれば、スライダーの値を`mGain`に反映できます。

```
processor.mGain = mGainSlider.getValue();
```



ここで、`GainTutorialAudioProcessorEditor`にスライダーの状態変化を検知するイベントリスナーとしての機能を持たせます。

以下のように、[`Slider::Listener`](https://docs.juce.com/master/classSlider_1_1Listener.html)クラスを継承させます。

```cpp
// PluginEditor.h

class GainTutorialAudioProcessorEditor  : public AudioProcessorEditor,
                                        public Slider::Listener
{
public:
    // .........
```

`Slider::Listener`クラスは純粋仮想関数`sliderValueChanged(Slider *slider)`を持つため、これをオーバーライドする必要があります。

以下のように、値が変化したスライダーが上で作成した`mGainSlider`だった場合に、そのスライダーの値を`mGain`に格納するコードを記述します。

```cpp
// ProcessorEditor.h

class GainTutorialAudioProcessorEditor  : public AudioProcessorEditor,
                                        public Slider::Listener
{
public:
    // ...... 
    void sliderValueChanged (Slider *slider) override;
```


```cpp
// ProcessorEditor.cpp

// .....
void GainTutorialAudioProcessorEditor::sliderValueChanged(Slider *slider)
{
    if (slider == &mGainSlider)
    {
        processor.mGain = mGainSlider.getValue();
    }
}
```

最後に、`mGainSlider`の変化を`GainTutorialAudioProcessorEditor`クラスに監視させるコードをコンストラクタ内に記述します。

```cpp
// ProcessorEditor.cpp

GainTutorialAudioProcessorEditor::GainTutorialAudioProcessorEditor (GainTutorialAudioProcessor& p)
    : AudioProcessorEditor (&p), processor (p)
{
    // ....
    mGainSlider.addListener(this);
    addAndMakeVisible(mGainSlider);
    
    setSize (200, 300);
}
```

ここで再度ビルドしてDAW側でプラグインを起動し、音源を再生しながらスライダーを動かしたときに音量が変化すれば、`mGainSlider`と`mGain`の紐付けは成功です。

## スライダーの対数スケール化

ウェーバー・フェヒナーの法則およびスティーブンスの法則により、人間が知覚する音の大きさは音の強さの対数、あるいは指数部が1より小さい指数に比例します。

実際、現状のスライダーを動かしてみると、1.0近い値を取る位置にあるほど変化を感じづらくなっていると思います。

この変化の感じ方を一定にするために、例えばデフォルト設定のLogic Pro Xでは以下のようなdBFSを単位としたレベルメータが付属しています。

![level meter](/img/juce_gain_control/level_meter.png)

JUCEでは`Decibels`クラスを用いれば、簡単にdB値からゲインに変換することができます。

まずは、スライダーの範囲と初期値をdBFSのスケールに合うように変更します。

```cpp
// PluginEditor.cpp
GainTutorialAudioProcessorEditor::GainTutorialAudioProcessorEditor (GainTutorialAudioProcessor& p)
    : AudioProcessorEditor (&p), processor (p)
{
    // ............
    mGainSlider.setRange(-60.0f, 0.0f, 0.01f);
    mGainSlider.setValue(-12.0f);
    // ............
}
```

そして`processBlock()`側で`Decibels::decibelsToGain()`関数を使ってスライダーのdB値をゲイン値に変換します。

```cpp
// PluginProcessor.cpp

void GainTutorialAudioProcessor::processBlock (AudioBuffer<float>& buffer, MidiBuffer& midiMessages)
{
    // ..............
    for (int channel = 0; channel < totalNumInputChannels; ++channel)
    {
        auto* channelData = buffer.getWritePointer (channel);
        for (int sample = 0; sample < buffer.getNumSamples(); ++sample)
            channelData[sample] *= Decibels::decibelsToGain(mGain);
    }
}
```

再度ビルドしてするとスライダーのスケールが変わり、スライダーのかかり方も変化していることが分かります。

![final slider](/img/juce_gain_control/slider_final.png)