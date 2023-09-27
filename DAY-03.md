# 【鐵人賽】DAY-03 初探 protobuf、gRPC

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-03-初探-protobuf、gRPC-01.png)

本文同步發佈於[毛毛的踩坑人生](https://tocandraw.com/2023-ironman/782/)

## 前言

昨天，我們有提到預計會將整個框架拆分成兩個後端、一個前端
並且後端跟後端之間是採用 gRPC 的方式溝通
應該是目前跨平台所使用的大多數微服務採用的方案
要使用這個技術
那就得先提到 [Protobuf](https://protobuf.dev)、[gRPC](https://grpc.io)
但我們會從動機開始談

## 動機

說到 API 的溝通媒介
我想大家第一時間還是會想起 `JSON`
作為一個跨 process 的溝通格式，而且也是人類可讀
也被廣泛使用了很長一段時間
優缺點也是很明顯，`Protobuf` 也強力問世
下面我們來比較一下兩者

### JSON

#### 優點

1. **易於讀寫和理解：** JSON 使用人類可讀的格式，易於閱讀和手動編輯。

3. **簡單性：** JSON 是一種輕量級的數據交換格式，不需要額外的工具或庫就可以解析和生成。

5. **廣泛支援：** JSON 被幾乎所有編程語言支援，可以輕鬆地在不同環境中使用。

#### 缺點

1. **效能較差：** JSON 效能較 Protobuf 差，因為它的文本格式需要解析和序列化的時間較長。

3. **佔用空間大：** JSON 使用文本格式，佔用的空間比二進制格式（如 Protobuf）多，導致較大的數據大小。

5. **沒有類型信息：** JSON 缺乏類型信息，需要額外的邏輯來處理數據的類型和校驗。

### Protobuf

#### 優點

1. **高效的二進制格式：** Protobuf 使用二進制編碼，比 JSON 更節省空間，並且解析/序列化速度更快。

3. **強大的類型系統：** Protobuf 定義了嚴格的數據類型和結構，可以提供更好的數據校驗和保護。

5. **自動生成程式碼：** Protobuf 可以從定義文件自動生成程式碼，減少了手動編碼的工作量。

7. **跨平台和語言支援：** Protobuf 支援多種編程語言，可以在不同平台上使用，並提供了良好的互操作性。

#### 缺點

1. **不易讀寫：** Protobuf 使用二進制編碼，對於人來說不易讀寫，需要工具來解析和編輯。

3. **不易調試：** 由於二進制格式，難以手動檢查和調試數據，特別是在測試和除錯時可能較為困難。

5. **版本兼容性：** 在數據結構變更時，可能需要複雜的版本處理和升級策略，以確保舊版本和新版本的兼容性。

### 小小總結一下

- 效能方面，在程式交易的範疇下，高效的 `Protobuf` 勝🥇

- 強類型部分，可以在開發階段少掉很多冤枉路 `Protobuf` 勝🥇

- 人類可讀性的部分，以我自己的觀點，只要錯誤處理得當，不可讀並不能算缺點

## 編譯器

決定使用 Protobuf 後，第一件事就是安裝編譯器
這個編譯器將會負責將同一份 protobuf，編譯成不同的程式語言
這次框架使用到的 `Python, Golang, Flutter` 全部有被支持

### 安裝

可以到這個 [Github](https://github.com/protocolbuffers/protobuf/releases) 找到自己平台的 Release 來安裝
這邊提供一份，我常用的安裝腳本

```bash
version="24.3"
curl -fSL https://github.com/protocolbuffers/protobuf/releases/download/v$version/protoc-$version-osx-aarch_64.zip --output protoc.zip
rm -rf ~/sdk_tools/protoc
mkdir -p ~/sdk_tools/protoc
echo "Unzipping protoc.zip..."
unzip -q protoc.zip -d ~/sdk_tools/protoc
chmod +x ~/sdk_tools/protoc/bin/protoc
rm -rf protoc.zip
```

然後更新一下環境變數
將以下加入 `~/.bashrc or ~/.zashrc`

`export PATH="$HOME/sdk_tools/protoc/bin:$PATH"`

```bash
$protoc --version
libprotoc 24.3
```

能夠看到版本，就沒有問題

### 定義 Proto

這邊將會使用最新的 [`proto3`](https://protobuf.dev/programming-guides/proto3/)
我先舉一個最基本的例子，在未來的篇幅會依據當時的進度加入新的 proto file

下面用一個期貨單的明細當作範例

```c
syntax            = "proto3";
option go_package = "./pb";
package toc_python_forwarder;

message FutureOrderDetail {
    string code    = 1;
    double price   = 2;
    int64 quantity = 3;
}
```

來說明一下，上面這段

1. `syntax = "proto3";`：這一行指定了使用 Proto3 語法，表明這個文件使用 Proto3 版本的 Protocol Buffers

3. `option go_package = "./pb";`：這一行指定了生成的 Golang package 名稱

5. `message FutureOrderDetail`：在 Protobuf 中會使用 message 當作宣告一個結構的開頭

7. `string code = 1;`、`double price = 2;`、`int64 quantity = 3;`：這些行定義了 `FutureOrderDetail` 消息的字段。每個字段都有一個名稱、類型和唯一的編號。在這個例子中，我們定義了三個：
    - `code` 是一個 string，編號為 1。

    - `price` 是一個 float，編號為 2。

    - `quantity` 是一個 int64，編號為 3。

特別要注意編號在一個結構當中，必須是個唯一值

### 資料結構

這邊分享一下比較常見的結構，如下圖

```bash
.
├── README.md
└── protos
    └── v3
        ├── app
        │   └── app.proto
        └── forwarder
            ├── basic.proto
            ├── entity.proto
            ├── history.proto
            ├── mq.proto
            ├── realtime.proto
            ├── subscribe.proto
            └── trade.proto
```

### 編譯

我們把剛剛那個專案資料夾先命名為 `trade-protobuf`
並且執行位置在這個資料夾的外面一層

#### Python

剛剛忘記了 Python 這邊，我會使用 Python 的 `grpcio-tools` 來取代 `protoc`
原因是到 gRPC 的編譯時，protoc 會缺少一些依賴，所幸我就用 `python` 原生的編譯器

```bash
$pip3 install -U \
  --no-warn-script-location \
  --no-cache-dir \
  grpcio \
  grpcio-tools

$outpath=./src/pb
$python3 -m grpc_tools.protoc \
  --python_out=$outpath \
  --grpc_python_out=$outpath \
  --mypy_out=$outpath \
  --proto_path=./trade-protobuf/protos/v3/app \
  --proto_path=./trade-protobuf/protos/v3/forwarder \
  ./trade-protobuf/protos/v3/*/*.proto
```

#### Golang

```bash
protoc \
    --go_out=. \
    --go-grpc_out=. \
    --proto_path=./trade-protobuf/protos/v3/app \
    --proto_path=./trade-protobuf/protos/v3/forwarder \
    ./trade-protobuf/protos/v3/*/*.proto
```

#### Flutter

```bash
protoc \
    --dart_out=./lib/pb \
    --proto_path=./trade-protobuf/protos/v3/app  \
    ./trade-protobuf/protos/v3/app/app.proto
```

## 總結

今天大略講述了我在 Protobuf 中的了解
以及如何編譯，接下來會繼續來到 gRPC
一步一步進入到核心的邏輯
