# 蓄電システム 通信仕様(実装例)

**目次**
- [蓄電システム 通信仕様(実装例)](#蓄電システム-通信仕様実装例)
  - [**1. 用語・略語**](#1-用語略語)
  - [**2. 構成**](#2-構成)
  - [**3. 通信仕様**](#3-通信仕様)
  - [**4. 通信シーケンス**](#4-通信シーケンス)
  - [**5. レジスタマップ**](#5-レジスタマップ)
  - [**6. コマンドフォーマット**](#6-コマンドフォーマット)
    - [6.1 Request コマンドフォーマット](#61-request-コマンドフォーマット)
    - [6.2 Response コマンドフォーマット](#62-response-コマンドフォーマット)
    - [6.3 Error コマンドフォーマット](#63-error-コマンドフォーマット)
    - [6.4 Exeption code](#64-exeption-code)
    - [6.5 CRC 算出処理について](#65-crc-算出処理について)

## **1. 用語・略語**

| **用語**         | **説明**|
|:--|:--|
| apis-main        | Sony CSL が開発した電力相互融通ソフトウェアの名称である|
| PP2P             | Physical Peer to Peer の略である<br>ブロックチェーン等による台帳管理による電力取引ではなく<br>物理的に特定の相手間での電力交換ができるという意味を込めてP2Pと分けた表現にしている|
|ユニット           |DCグリッド上に接続された１つのノードが構成する<br> Linux-board(apis-main)、蓄電システム、DC/DC converter を含めたシステムの単位|
|クラスタ           |apis-mainは、コミュニケーションライン上に存在する複数のapis-mainとクラスタを構築し、<br>同一クラスタのユニットと電力相互融通を実施する|


---

## **2. 構成**
本通信仕様は、Sony CSL が開発した自律分散制御の電力相互融通ソフトウェア [APIS](https://github.com/SonyCSL/APIS) と実際の蓄電システムを通信する際に、Device Driverとして、[dcdc_batt_comm](https://github.com/SonyCSL/apis-dcdc_batt_comm) を使用する場合の蓄電システムの通信仕様例である。

<div align="center">
<img src="media/dcdc_batt_comm_battery-image001.PNG" alt="システム構成" width="600" height="375">
</div>

<br>

---

## **3. 通信仕様**

|項目|仕様|備考|
|:-:|:--|:--|
|インターフェース|RS485||
|伝送速度|115200bps|9600, 19200, 38400, 57600, 115200bps選択|
|データ長|8bit||
|パリティ|None||
|スタートビット|1bit||
|ストップビット|1bit||
|フロー制御|None||
|通信プロトコル|Modbus RTU||
|ビット転送順序|LSB|下位ビットから送信|
|バイトオーダー|ビックエンディアン|上位バイトから送信|
|誤り検出|CRC||

<br>

---

## **4. 通信シーケンス**
以下に、Request/Response の通信シーケンスを示す。Device Driver 側がマスタ、蓄電システム側がスレーブの関係となる。

|Function code|送信方向|データ名称|名称/用途|
|:-:|:-:|:-:|:--|
|0x04|Device Driver → 蓄電システム|Request|Input Register の読込要求|
|0x04|蓄電システム → Device Driver|Response|指定 Input Register の内容を応答|
|0x84|蓄電システム → Device Driver|Error|Request 電文内容不正時の応答|

<br>

---

## **5. レジスタマップ**
Input Register の電池状態の Modbus アドレスを以下に示す。

|Modbus Address|Hex. Addr. Offset ※|Data名称|Bytes|R/W属性|Data 型|範囲|
|:-:|:-:|:-:|:-:|:-:|:-:|:--|
|30030|001D|RSOC(残容量率)|2|R|unit16|0.0～100.0%|
|30031|001E|融通許可/不許可|2|R|bit field|bit0:動作状態<br>(0:許可, 1:不許可)<br>bit1-15:RESERVED|

※Request コマンドの Start Address に設定

<br>

---

## **6. コマンドフォーマット**
基本のコマンドフォーマットを以下に示す。

|Byte数|-|1|1|N|2|-|
|:-:|:-:|:-:|:-:|:-:|:-:|:--|
|コマンドフォーマット|Start|Address|Function|Data|CRC Check|End|
|項目|-|Slave Address|Function Code||CRC|-|

<br>

### 6.1 Request コマンドフォーマット

<table>
<thead>
<tr class="head">
<th>Byte</th>
<th>コマンド<br>フォーマット名</th>
<th>データ名</th>
<th>Size</th>
<th>備考</th>
</tr>
</thead>
<tbody>
<tr class="even">
<td>0</td>
<td>Address</td>
<td>Slave Address</td>
<td>1</td>
<td>マスタ側で指定</td>
</tr>
<tr class="odd">
<td>1</td>
<td>Function</td>
<td>Function Code</td>
<td>1</td>
<td>0x04:Read Input Registers</td>
</tr>
<tr class="even">
<td>2</td>
<td rowspan="4">Data</td>
<td rowspan="2">Start Address</td>
<td rowspan="2">2</td>
<td rowspan="2">先頭アドレス Hex. Addr. Offset の値を設定</td>
</tr>
<tr class="odd">
<td>3</td>

</tr>
<tr class="even">
<td>4</td>

<td rowspan="2">Quantity of Input Registers</td>
<td rowspan="2">2</td>
<td rowspan="2">レジスタ(word=2byte)数(N)
</td>
</tr>
<tr class="odd">
<td>5</td>

</tr>
<tr class="even">
<td>6</td>
<td rowspan="2">CRC Check</td>
<td rowspan="2">CRC</td>
<td rowspan="2">2</td>
<td rowspan="2">0~(Nx2)+2byteの CRC 計算結果<br>CRC-Lo, CRC-Hi の順で設定</td>
</tr>
<tr class="odd">
<td>7</td>

</tr>
</tbody>
</table>


<br>

### 6.2 Response コマンドフォーマット

<table>
<thead>
<tr class="head">
<th>Byte</th>
<th>コマンド<br>フォーマット名</th>
<th>データ名</th>
<th>Size</th>
<th>備考</th>
</tr>
</thead>
<tbody>
<tr class="even">
<td>0</td>
<td>Address	</td>
<td>Slave Address</td>
<td>1</td>
<td>Request をそのまま設定</td>
</tr>
<tr class="odd">
<td>1</td>
<td>Function</td>
<td>Function Code</td>
<td>1</td>
<td>0x04:Read Input Registers</td>
</tr>
<tr class="even">
<td>2</td>
<td rowspan="8">Data</td>
<td>Byte Count</td>
<td>1</td>
<td>2xN</td>
</tr>
<tr class="odd">
<td>3</td>

<td rowspan="2">Input Register#1</td>
<td rowspan="2">2</td>
<td rowspan="2">要求レジスタ#1 内容</td>
</tr>
<tr class="even">
<td>4</td>


</tr>
<tr class="odd">
<td>5</td>

<td rowspan="2">Input Register#2</td>
<td rowspan="2">2</td>
<td rowspan="2">要求レジスタ#2 内容</td>
</tr>
<tr class="even">
<td>6</td>

</tr>
<tr class="odd">
<td>:</td>

<td>:</td>
<td>:</td>
<td>:</td>
</tr>
<tr class="even">
<td>(Nx2)+1</td>

<td rowspan="2">Input Register#N</td>
<td rowspan="2">2</td>
<td rowspan="2">要求レジスタ#N 内容</td>
</tr>
<tr class="odd">
<td>(Nx2)+2</td>

</tr>
<tr class="even">
<td>(Nx2)+3</td>
<td rowspan="2">CRC Check</td>
<td rowspan="2">CRC</td>
<td rowspan="2">2</td>
<td rowspan="2">0~(Nx2)+2byteの CRC 計算結果<br>CRC-Lo, CRC-Hi の順で設定</td>
</tr>
<tr class="odd">
<td>(Nx2)+4</td>
</tr>
</tbody>
</table>

<br>

### 6.3 Error コマンドフォーマット

<table>
<thead>
<tr class="head">
<th>Byte</th>
<th>コマンド<br>フォーマット名</th>
<th>データ名</th>
<th>Size</th>
<th>備考</th>
</tr>
</thead>
<tbody>
<tr class="even">
<td>0</td>
<td>Address</td>
<td>Slave Address</td>
<td>1</td>
<td>Request をそのまま設定</td>
</tr>
<tr class="odd">
<td>1</td>
<td>Function</td>
<td>Error Code</td>
<td>1</td>
<td>0x80+0x04</td>
</tr>
<tr class="even">
<td>2</td>
<td>Data</td>
<td>Exception Code</td>
<td>1</td>
<td>01 or 02 or 03 or 04</td>
</tr>
<tr class="odd">
<td>3</td>
<td rowspan="2">CRC Check</td>
<td rowspan="2">CRC</td>
<td rowspan="2">2</td>
<td rowspan="2">0~2byte の CRC 計算結果<br>CRC-Lo, CRC-Hi の順で設定</td>
</tr>
<tr class="even">
<td>4</td>
</tr>
</tbody>
</table>

<br>

### 6.4 Exeption code
例外コードのについて以下に示す。

|Code|名称|範囲|意味|
|:-:|:-:|:--|:--|
|01|ILLEGAL FUNCTION|正常 0x04<br>異常 上記以外|ファンクションコードが不正(0x04 以外)クエリで受け取ったファンクションコードは、サーバに対して許容されるアクションではありません。これは、ファンクションコードが新しいデバイスにのみ適用され、選択されたユニットには実装されていない可能性があります。また、このタイプの要求を処理するためにサーバが誤った状態にあることも示されます。たとえば、構成されておらず、レジスタ値を戻すように要求されているためです。|
|02|ILLEGAL DATA ADDRESS|正常<br>開始アドレス:0000～270E<br>レジスタ数:1～125<br><br>異常<br>上記以外または<br>開始アドレスと<br>レジスタ数で領域を超えた場合|開始アドレスとレジスタ数が不正クエリで受信したデータアドレスは、サーバの許容アドレスではありません。 具体的には、参照番号と転送長の組み合わせが無効です。 100 個のレジスタを持つコントローラの場合、PDUは最初のレジスタを0、最後のものを99 として扱います。開始レジスタのアドレスが96でレジスタの数が4 で要求が送信された場合、この要求は正常に動作します。 96 の開始レジスタアドレスと5のレジスタ数で要求が提出された場合、この要求は例外コード0x02 "Illegal Data Address"で失敗します。 それはレジスタ96,97,98,99 および100 上で動作しようとし、アドレス100 を有するレジスタは存在しないからである。|
|03|ILLEGAL DATA VALUE|正常 1～125<br>異常 上記以外|レジスタ数(1～125)が範囲外クエリデータフィールドに含まれる値は、サーバの許容値ではありません。 暗黙的な長さが正しくないなど、複雑な要求の残りの構造に障害があることを示します。MODBUS プロトコルは特定のレジスタの特定の値の意義を認識していないため、レジスタに格納するために提出されたデータ項目がアプリケーションプログラムの期待値外の値を持つことを特に意味しません。|
|04|SERVER DEVICE FAILURE|-|サーバの障害サーバが要求されたアクションを実行しようとしていたときに、回復不能なエラーが発生しました。|


<br>

### 6.5 CRC 算出処理について
以下にCRC16 算出処理を示す。

**CRC16 算出ロジック**
```
unsigned short CRC16 ( puchMsg, usDataLen ) /* The function returns the CRC as a unsigned short type*/
unsigned char *puchMsg ;                    /* message to calculate CRC upon */
unsigned short usDataLen ;                  /* quantity of bytes in message */
{
    unsigned char uchCRCHi = 0xFF ;             /* high byte of CRC initialized */
    unsigned char uchCRCLo = 0xFF ;             /* low byte of CRC initialized */
    unsigned uIndex ;                           /* will index into CRC lookup table */
    while (usDataLen--)                         /* pass through message buffer */
    {
        uIndex = uchCRCLo ^ *puchMsg++ ;            /* calculate the CRC */
        uchCRCLo = uchCRCHi ^ auchCRCHi[uIndex] ;
        uchCRCHi = auchCRCLo[uIndex] ;
    }
    return (uchCRCHi << 8 | uchCRCLo) ;
}
```

**上位バイトテーブル**
```
/* Table of CRC values for high–order byte */
static unsigned char auchCRCHi[] = {
    0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81,
    0x40, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0,
    0x80, 0x41, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40, 0x01,
    0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x01, 0xC0, 0x80, 0x41,
    0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81,
    0x40, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x01, 0xC0,
    0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x01,
    0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40,
    0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81,
    0x40, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0,
    0x80, 0x41, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40, 0x01,
    0xC0, 0x80, 0x41, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41,
    0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81,
    0x40, 0x01, 0xC0, 0x80, 0x41, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0,
    0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x01,
    0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41,
    0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81,
    0x40
} ;
```

**下位バイトテーブル**
```
/* Table of CRC values for low–order byte */
static char auchCRCLo[] = {
    0x00, 0xC0, 0xC1, 0x01, 0xC3, 0x03, 0x02, 0xC2, 0xC6, 0x06, 0x07, 0xC7, 0x05, 0xC5, 0xC4,
    0x04, 0xCC, 0x0C, 0x0D, 0xCD, 0x0F, 0xCF, 0xCE, 0x0E, 0x0A, 0xCA, 0xCB, 0x0B, 0xC9, 0x09,
    0x08, 0xC8, 0xD8, 0x18, 0x19, 0xD9, 0x1B, 0xDB, 0xDA, 0x1A, 0x1E, 0xDE, 0xDF, 0x1F, 0xDD,
    0x1D, 0x1C, 0xDC, 0x14, 0xD4, 0xD5, 0x15, 0xD7, 0x17, 0x16, 0xD6, 0xD2, 0x12, 0x13, 0xD3,
    0x11, 0xD1, 0xD0, 0x10, 0xF0, 0x30, 0x31, 0xF1, 0x33, 0xF3, 0xF2, 0x32, 0x36, 0xF6, 0xF7,
    0x37, 0xF5, 0x35, 0x34, 0xF4, 0x3C, 0xFC, 0xFD, 0x3D, 0xFF, 0x3F, 0x3E, 0xFE, 0xFA, 0x3A,
    0x3B, 0xFB, 0x39, 0xF9, 0xF8, 0x38, 0x28, 0xE8, 0xE9, 0x29, 0xEB, 0x2B, 0x2A, 0xEA, 0xEE,
    0x2E, 0x2F, 0xEF, 0x2D, 0xED, 0xEC, 0x2C, 0xE4, 0x24, 0x25, 0xE5, 0x27, 0xE7, 0xE6, 0x26,
    0x22, 0xE2, 0xE3, 0x23, 0xE1, 0x21, 0x20, 0xE0, 0xA0, 0x60, 0x61, 0xA1, 0x63, 0xA3, 0xA2,
    0x62, 0x66, 0xA6, 0xA7, 0x67, 0xA5, 0x65, 0x64, 0xA4, 0x6C, 0xAC, 0xAD, 0x6D, 0xAF, 0x6F,
    0x6E, 0xAE, 0xAA, 0x6A, 0x6B, 0xAB, 0x69, 0xA9, 0xA8, 0x68, 0x78, 0xB8, 0xB9, 0x79, 0xBB,
    0x7B, 0x7A, 0xBA, 0xBE, 0x7E, 0x7F, 0xBF, 0x7D, 0xBD, 0xBC, 0x7C, 0xB4, 0x74, 0x75, 0xB5,
    0x77, 0xB7, 0xB6, 0x76, 0x72, 0xB2, 0xB3, 0x73, 0xB1, 0x71, 0x70, 0xB0, 0x50, 0x90, 0x91,
    0x51, 0x93, 0x53, 0x52, 0x92, 0x96, 0x56, 0x57, 0x97, 0x55, 0x95, 0x94, 0x54, 0x9C, 0x5C,
    0x5D, 0x9D, 0x5F, 0x9F, 0x9E, 0x5E, 0x5A, 0x9A, 0x9B, 0x5B, 0x99, 0x59, 0x58, 0x98, 0x88,
    0x48, 0x49, 0x89, 0x4B, 0x8B, 0x8A, 0x4A, 0x4E, 0x8E, 0x8F, 0x4F, 0x8D, 0x4D, 0x4C, 0x8C,
    0x44, 0x84, 0x85, 0x45, 0x87, 0x47, 0x46, 0x86, 0x82, 0x42, 0x43, 0x83, 0x41, 0x81, 0x80,
    0x40
};
```
