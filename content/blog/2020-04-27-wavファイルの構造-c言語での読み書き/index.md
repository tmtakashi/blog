---
title: WAVファイルの構造&C言語での読み書き
date: 2020-04-27T11:10:20.161Z
description: wavファイルについて
---
この記事ではWAVファイルの構造と、C言語でのWAVファイルの扱い方（の一例）について解説します。

## WAVファイルの内容について

WAV ファイルは RIFF チャンクと呼ばれるブロック構造によって記述されています。 この RIFF チャンクの中には「fmt チャンク」と「data チャンク」が含まれており、前者は標本化周波数、量子化ビット数などのファイルのプロパティ情報、後者は音の波形データそのものが格納されています。

#### RIFFチャンクの先頭

| パラメータ     | バイト数 | 内容            |
|-----------|------|---------------|
| chunkID   | 4    | "RIFF"        |
| chunkSize | 4    | 音データのサイズ + 16 |
| formType  | 4    | "WAVE"        |

#### fmtチャンク

| パラメータ          | バイト数 | 内容                                                                 |
|----------------|------|--------------------------------------------------------------------|
| chunkID        | 4    | "fmt "                                                             |
| chunkSize      | 4    | 16                                                                 |
| waveFormatType | 2    | 音データの形式。PCMの場合1                                                    |
| channel        | 2    | モノラルの場合1、ステレオの場合2                                                  |
| samplesPerSec  | 4    | 標本化周波数                                                             |
| bytesPerSec    | 4    | 1秒間の音データを記録するのに必要なデータ量(byte)。blockSize * samplesPerSec             |
| blockSize      | 2    | (byte / sample)  *channel。 16bitステレオの場合 2 \[byte / sample]*  2 = 4 |
| bitsPerSample  | 2    | bitを単位とする量子化精度。8bitの場合は8, 16bitの場合は16                              |

#### dataチャンク

| パラメータ     | バイト数             | 内容       |
|-----------|------------------|----------|
| chunkID   | 4                | "data"   |
| chunkSize | 4                | 音データのサイズ |
| data      | (1 or 2) * サンプル数 | 音データ     |

dataパラメータには、量子化ビット数が8bitの場合最小値が0、オフセットが128、最大値が256の音データが記録され、 量子化ビット数が16bitの場合最小値が-32768、オフセットが0、最大値が32767の音データが記録されます。
![bit range](/img/bit_range.png)

モノラルの場合は時間が進むにつれて音データが順番に記録されていきます。 ステレオの場合は左チャンネルのデータと右チャンネルのデータが交互に記録されていきます。

![](/img/wave_block.svg)

### バイナリをのぞいてみる
WAVファイルのバイナリをVSCodeの拡張機能、[hexdump for VSCode](https://marketplace.visualstudio.com/items?itemName=slevesque.vscode-hexdump)を使って見てみましょう。
拡張機能をインストールしたあと、`*.wav`ファイルをファイルエクスプローラーで右クリックし、"Show hexdump for file"をクリックすると以下のようにファイルの内容が16進数表記のバイト列として表示されます。
右側に表示されている文字列は各バイトをASCIIで解釈したものになっています。
```
  Offset: 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 	
00000000: 52 49 46 46 A4 3E 00 00 57 41 56 45 66 6D 74 20    RIFF$>..WAVEfmt.
00000010: 10 00 00 00 01 00 01 00 40 1F 00 00 80 3E 00 00    ........@....>..
00000020: 02 00 10 00 64 61 74 61 80 3E 00 00 00 00 3F 0C    ....data.>....?.
00000030: A1 16 90 1D 00 20 90 1D A1 16 3F 0C 00 00 C1 F3    !.......!.?...As
```
WAVファイルのバイトオーダーは各chunkIDがビッグエンディアン、それ以外はリトルエンディアンで記述されています。

ビッグエンディアンは、例えば`0x1234ABCD`という4 byteのデータがあったときにバイト毎に上位側から`12 34 AB CD`と並べ、それに対しリトルエンディアンは下位側から`CD AB 34 12`と並べます。

いくつか具体的なプロパティを読んでみましょう。

-　チャンネル数は23, 24バイト目の位置(0x00000016~0x00000017番地）を見ると`01 00`と書かれているので、順序を戻すと
0x0001=1、よって1チャンネル（モノラル）。
- 標本化周波数は25〜28バイト目の位置(0x00000018~0x0000001B番地)を見ると`40 1F`と書かれているので、順序を戻すと0x1F40=8000、よって8000Hz
- 音データのサイズは41〜44バイト目の位置(0x00000028~0x0000002B番地)を見ると`80 3E 00 00`と書かれているので、順序を戻すと
0x3E80=16000, よって音データのサイズは16000（標本化周波数8000Hz, 量子化ビット数16bit(2byte)なので秒に換算すると16000 / 800 / 2 = 2秒）

バイト列を目で読むだけでWAVファイルの基本的な性質が読み取ることができますね。

### C言語でのWAVファイルの読み書き

C言語でWAVファイルの読み書きを実装してみます。ここでは必要な情報だけを保持できるように、
以下のような構造体`MONO_PCM`にWAVファイルのプロパティを移すことを考えます。
`MONO_PCM`構造体では、計算精度の向上のため、音データを-1.0 ~ 1.0の`double`型として扱います。

```c
typedef struct
{
  int fs; // 標本化周波数
  int bits; // 量子化ビット数
  int length; // 音データの長さ
  double *s; // 音データ
} MONO_PCM
```

以下の`wave.h`にWAVファイルから`MONO_PCM`構造体にデータを読み込む`mono_read_wave`関数と、`MONO_PCM`構造体にあるデータからWAVファイルを生成する`mono_write_wave`関数を示します。
なお、これらの関数は量子化ビット数16bitの音源を想定しています。
（個人的にわかりにくかったところにはコメントを入れています）

```c
/* wave.h */
#include <stdio.h>
#include <stdlib.h>

void mono_wave_read(MONO_PCM *pcm, char *file_name)
{
    FILE *fp; 
    int n;
    char riff_chunk_ID[4];
    long riff_chunk_size;
    char riff_form_type[4];
    char fmt_chunk_ID[4];
    long fmt_chunk_size;
    short fmt_wave_format_type;
    short fmt_channel;
    long fmt_samples_per_sec;
    long fmt_bytes_per_sec;
    short fmt_block_size;
    short fmt_bits_per_sample;
    char data_chunk_ID[4];
    long data_chunk_size;
    short data;

    // file_nameにつながるストリームを作り、それを管理するFILEへのポインタ返す
    fp = fopen(file_name, "rb");

    // size_t fread(void *buf, size_t size, size_t nmemb, FILE *stream)
    // streamのストリームから最大size * nmembバイトのデータを読み込み,
    // bufの指すメモリ領域に書き込む
    // fread, fwriteは実行するたびにファイルオフセット（どこまで読み書きしたか）が更新される
    fread(riff_chunk_ID, 1, 4, fp);
    fread(&riff_chunk_size, 4, 1, fp);
    fread(riff_form_type, 1, 4, fp);
    fread(fmt_chunk_ID, 1, 4, fp);
    fread(&fmt_chunk_size, 4, 1, fp);
    fread(&fmt_wave_format_type, 2, 1, fp);
    fread(&fmt_channel, 2, 1, fp);
    fread(&fmt_samples_per_sec, 4, 1, fp);
    fread(&fmt_bytes_per_sec, 4, 1, fp);
    fread(&fmt_block_size, 2, 1, fp);
    fread(&fmt_bits_per_sample, 2, 1, fp);
    fread(data_chunk_ID, 1, 4, fp);
    fread(&data_chunk_size, 4, 1, fp);

    /* メンバー変数に標本化周波数、量子化ビット数、信号の長さを書き込む */
    pcm->fs = (int)fmt_samples_per_sec;
    pcm->bits = (int)fmt_bits_per_sample;
    pcm->length = (int)data_chunk_size / 2;
    /* メモリのヒープ領域に信号用のメモリ領域を確保する */ 
    pcm->s = (double *)calloc(pcm->length, sizeof(double));

    for (n = 0; n < pcm->length; n++)
    {
        /* 音データの読み込み */
        fread(&data, 2, 1, fp);
        /* -1 ~ 1の範囲に正規化 */
        pcm->s[n] = (double)data / 32768.0;
    }
    
    /* ストリームを閉じる */
    fclose(fp);
}

void mono_wave_write(MONO_PCM *pcm, char *file_name)
{
    FILE *fp;
    int n;
    char riff_chunk_ID[4];
    long riff_chunk_size;
    char riff_form_type[4];
    char fmt_chunk_ID[4];
    long fmt_chunk_size;
    short fmt_wave_format_type;
    short fmt_channel;
    long fmt_samples_per_sec;
    long fmt_bytes_per_sec;
    short fmt_block_size;
    short fmt_bits_per_sample;
    char data_chunk_ID[4];
    long data_chunk_size;
    short data;
    double s;

    riff_chunk_ID[0] = 'R';
    riff_chunk_ID[1] = 'I';
    riff_chunk_ID[2] = 'F';
    riff_chunk_ID[3] = 'F';
    riff_chunk_size = 36 + pcm->length * 2;
    riff_form_type[0] = 'W';
    riff_form_type[1] = 'A';
    riff_form_type[2] = 'V';
    riff_form_type[3] = 'E';

    fmt_chunk_ID[0] = 'f';
    fmt_chunk_ID[1] = 'm';
    fmt_chunk_ID[2] = 't';
    fmt_chunk_ID[3] = ' ';
    fmt_chunk_size = 16;
    fmt_wave_format_type = 1;
    fmt_channel = 1;
    fmt_samples_per_sec = (long)pcm->fs;
    fmt_bytes_per_sec = (long)(pcm->fs * pcm->bits / 8);
    fmt_block_size = (short)(pcm->bits / 8);
    fmt_bits_per_sample = (short)pcm->bits;

    data_chunk_ID[0] = 'd';
    data_chunk_ID[1] = 'a';
    data_chunk_ID[2] = 't';
    data_chunk_ID[3] = 'a';
    data_chunk_size = (long)pcm->length * 2;

    fp = fopen(file_name, "wb");

    fwrite(riff_chunk_ID, 1, 4, fp);
    fwrite(&riff_chunk_size, 4, 1, fp);
    fwrite(riff_form_type, 1, 4, fp);
    fwrite(fmt_chunk_ID, 1, 4, fp);
    fwrite(&fmt_chunk_size, 4, 1, fp);
    fwrite(&fmt_wave_format_type, 2, 1, fp);
    fwrite(&fmt_channel, 2, 1, fp);
    fwrite(&fmt_samples_per_sec, 4, 1, fp);
    fwrite(&fmt_bytes_per_sec, 4, 1, fp);
    fwrite(&fmt_block_size, 2, 1, fp);
    fwrite(&fmt_bits_per_sample, 2, 1, fp);
    fwrite(data_chunk_ID, 1, 4, fp);
    fwrite(&data_chunk_size, 4, 1, fp);

    for (n = 0; n < pcm->length; n++)
    {
        // pcm->s[n] + 1.0 => 0.0 ~ 2.0の範囲
        // (pcm->s[n] + 1.0) / 2.0 => 0.0 ~ 1.0の範囲
        // (pcm->s[n] + 1.0) / 2.0 * 65536.0 => 0.0 ~ 65536.0の範囲
        s = (pcm->s[n] + 1.0) / 2.0 * 65536.0;

        // クリッピング
        if (s > 65535.0)
        {
            s = 65535.0;
        }
        else if (s < 0.0)
        {
            s = 0.0;
        }

        // 四捨五入 & -32768 ~ 32767の範囲に戻す
        // 例えばs=30000.3の場合、(short)(s + 0.5)は(short)(30000.8)=> 30000
        // s=30000.8の場合、(short)(s + 0.5)は(short)(30001.3) => 30001
        data = (short)(s + 0.5) - 32768;

        fwrite(&data, 2, 1, fp);
    }

    fclose(fp);
}
```

#### 読み込み

この`wave.h`を使って実際にWAVファイルの各種プロパティを`MONO_PCM`構造体経由で読み込んでみます。

```c
// read_wave.c 

#include <stdio.h>
#include "wave.h"

int main(int argc, char *argv[])
{
    if (argc < 2)
    {
        fprintf(stderr, "%s: file name not given\n", argv[0]);
        exit(1);
    }
    MONO_PCM pcm;
    mono_wave_read(&pcm, argv[1]);
    printf("Sampling frequency: %d\n", pcm.fs);
    printf("Quantization bit rate: %d\n", pcm.bits);
    printf("Signal length: %d\n", pcm.length);
}
```

以上のコードをコンパイルして、適当なWAVファイルを引数にとって実行します。
```
gcc -o read_wave read_wave.c
./read_wave sine.wav
```
すると
```
Sampling frequency: 8000
Quantization bit rate: 16
Signal length: 8000
```
のような結果が出力され、WAVファイルの基本的なプロパティが読み込めていることが分かります。
肝心の音データですが、パイプ接続用のAPIである`popen()`関数を使って、gnuplotでグラフを書いてみます。

```c
// plot_wav.c

#include <stdio.h>
#include <stdlib.h>
#include <math.h>
#include <unistd.h>
#include "wave.h"

#define WAIT 100000000

int main(int argc, char *argv[])
{
    if (argc < 2)
    {
        fprintf(stderr, "%s: file name not given\n", argv[0]);
        exit(1);
    }

    MONO_PCM pcm;
    FILE *GPLT;
    int n;

    mono_wave_read(&pcm, argv[1]);
    // gnuplot用のパイプ接続されたストリームを開く
    GPLT = popen("gnuplot", "w");
    // 1行ずつデータを読み込むために '-' をつかう
    // w l => with line
    fprintf(GPLT, "plot '-' w l\n ");
    // すべて表示すると見づらいので表示範囲を狭める
    for (n = 0; n < pcm.length / 10; n++)
    {
        fprintf(GPLT, "%d %f\n", n, pcm.s[n]);
    }
    // 'e'で始まる行に出会うとデータ読み込みが終了
    fprintf(GPLT, "e\n");
    // ストリームに溜まっているデータをwrite
    fflush(GPLT);
    // 即時に終了するのを阻止
    usleep(WAIT);
    pclose(GPLT);
}
```

以上のコードを以下のようにコンパイルして実行すると、gnuplotのウィンドウが起動してグラフが表示されます。
```
gcc -o plot_wav plot_wav.c
./plot_wav sine.wav
```
![](/img/sine.svg)

よって、音データの取得もできていることが分かります。

#### 書き込み

次に、`MONO_PCM`構造体と`mono_wave_write`を使って純音のWAV音源を生成してみます。
以下の`sine.c`では、サンプリング周波数8000Hz、量子化ビット数16bit、周波数250Hzの純音のWAVファイル音源を書き出します。

```c
// sine.c

#include <stdio.h>
#include <stdlib.h>
#include <math.h>
#include "wave.h"

int main()
{
    MONO_PCM pcm1;
    int n;
    double A, f0;

    pcm1.fs = 8000;
    pcm1.bits = 16;
    pcm1.length = 8000;
    // メモリ領域を動的確保
    pcm1.s = calloc(pcm1.length, sizeof(double));

    A = 0.25;
    f0 = 250.0;

    for (n = 0; n < pcm1.length; n++)
    {
        pcm1.s[n] = A * sin(2.0 * M_PI * f0 * n / pcm1.fs);
    }
    mono_wave_write(&pcm1, "sine.wav");
    // メモリ領域を開放
    free(pcm1.s);

    return 0;
}
```

これをコンパイルして実行すると、純音のWAVファイル音源が作成されます。


