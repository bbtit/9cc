# コンパイルの大まかな流れ
1. Tokenize → 文字列（トークン列）を単語（トークン）に区切り連結リストを作成
1. Parse → 連結リストから生成規則をもとに構文木を作成
1. Codegen → 構文木からアセンブリを作成


# main.c
argc（引数の個数）とargv（文字列配列）をコマンドライン引数から受け取る。

`user_input`と`token`はヘッダファイル`chibicc.h`でグローバル変数の宣言がされている。
```c
extern char *user_input;
extern Token *token;
```


# tokenize.c
`Token *tokenize()`が主な関数。グローバル変数`user_input`をtokenizeし、構造体Tokenで構成される連結リストの先頭を返す。


## tokenize.cの関数
- `Token *new_token(TokenKind kind, Token *cur, char *str, int len)`
  - 引数をもとに新たなトークン`tok`を作成。既存のトークン`cur`に連結し、`tok`を返す
- `bool consume(char *op)`
  - 現在の`token`が引数`*op`と一致していれば連結リストを進め(`token=token->next;`)て`true`を返す
- `void expect(char *op)`
  - 現在の`token`が引数`*op`と一致していれば連結リストを進め(`token=token->next;`)る
- `int expect_number()`
  - 現在の`token`が数字ならば、その値を返し、連結リストを進め(`token=token->next;`)る
- `bool startswith(char *p, char *q)`
  - pとqが一致しているかどうか(`memcmp(p, q, strlen(q)) == 0;`)を返す

## 構造体Token
```c
typedef enum {
  TK_RESERVED, // Keywords or punctuators 記号
  TK_IDENT,    // Identifiers 変数
  TK_NUM,      // Integer literals 整数
  TK_EOF,      // End-of-file markers 終端
} TokenKind;

// Token type
typedef struct Token Token;
struct Token {
  TokenKind kind; // Token kind 種類
  Token *next;    // Next token 次のトークンのポインタ
  int val;        // If kind is TK_NUM, its value 数字の値
  char *str;      // Token string 
  int len;        // Token length 
};
```

# parse.c
連結リストから生成規則をもとに構文木を作成する。生成規則をもとに再帰的な関数を書く。

## 生成規則
EBNFの書き方についてはEBNFの項を参照

生成規則の例
```
program    = stmt*
stmt       = expr ";"
expr       = assign
assign     = equality ("=" assign)?
equality   = relational ("==" relational | "!=" relational)*
relational = add ("<" add | "<=" add | ">" add | ">=" add)*
add        = mul ("+" mul | "-" mul)*
mul        = unary ("*" unary | "/" unary)*
unary      = ("+" | "-")? primary
primary    = num | ident | "(" expr ")"
```



## 構造体Node
```c
typedef enum {
  ND_ADD,    // +
  ND_SUB,    // -
  ND_MUL,    // *
  ND_DIV,    // /
  ND_EQ,     // ==
  ND_NE,     // !=
  ND_LT,     // <
  ND_LE,     // <=
  ND_ASSIGN, // =
  ND_RETURN, // "return"
  ND_EXPR_STMT, // Expression statement
  ND_LVAR,   // Local variable
  ND_NUM,    // Integer
} NodeKind;

// AST node type
typedef struct Node Node;
struct Node {
  NodeKind kind; // Node kind
  Node *next;    // Next node
  Node *lhs;     // Left-hand side
  Node *rhs;     // Right-hand side
  char name;     // Used if king == ND_LVAR
  int val;       // Used if kind == ND_NUM
};
```

# Codegen

# C言語の関数
- `malloc`
  - メモリ領域を割り当て、そのメモリアドレスを返す
- `calloc`
  - メモリ領域を割り当て、全ビットを0に初期化してから、そのメモリアドレスを返す
- `memcmp`
  - 第1引数、第2引数は比較対象メモリのポインタ、第3引数は比較サイズ
  - 比較サイズの分だけ第1引数と第2引数を比較
  - 一致していれば0を、一致していなければ1または-1を返す
- `strtol`
  - 数値を読み込んだ後、第2引数のポインタをアップデートして、読み込んだ最後の文字の次の文字を指すように値を更新
  - 数値を1つ読み込んだ後、その次の文字が+や-ならば、pはその文字を指す

# アセンブリ(オペコード)
- mov
  - 第1オペランドに第2オペランドの値をコピー
- jmp
  - プログラムの制御が指定したアドレスに移動
- push
  - スタックに値をプッシュ
  - スタックポインタ（RSPレジスタ）が減算
  - 指定した値が減算された位置にストアされる
- pop
  - この命令はスタックから値をポップ
  - ポップした値を指定したレジスタに保存
  - スタックポインタは増加する
- ret
  - サブルーチン（関数）からのリターンに使用
  - リターンアドレスをスタックからポップ、そのアドレスにジャンプ
  - サブルーチンを呼び出した元のコードに制御が戻る
- add
  - 2つのオペランドを加算し、結果を第1オペランドに格納
- sub
  - 2つのオペランドを減算し、結果を第1オペランドに格納
- imul
  - 2つのオペランドを符号付き整数として乗算し、結果を第1オペランドに格納
- cqo
  - RAXの符号拡張を行い、その結果をRDX:RAX（128ビットレジスタペア）に格納
  - 64ビット除算の前に実行されることが多い
- idiv
  - RDX:RAX（128ビットレジスタペア）を指定されたオペランドで符号付き除算
  - 商をRAXに、剰余をRDXに格納
- cmp
  - 第二のオペランドを第一のオペランドから減算し、その結果に基づいてフラグ（ゼロフラグ、符号フラグ、キャリーフラグなど）を設定
  - 結果自体は格納しない
  - ジャンプ命令と共に条件分岐に使用される
- sete
  - 直前のcmp命令で調べた2つのレジスタの値が同じだった場合に、指定されたレジスタに1をセット
  - ゼロフラグがセット（1）の場合、対象のオペランドに1をセット
  - ゼロフラグが1以外の場合、対象のオペランドに0をセット
  - 等しいかどうか（Equal）のテストに使用
- setle
  - ゼロフラグがセットまたは符号フラグとオーバーフローフラグが異なる場合、対象のオペランドに1をセット
  - それ以外の場合、対象のオペランドに0をセット
  - 小さいか等しい（Less or Equal）のテストに使用
- movzb
  - Bのビット数を除いたAの残りの上位ビットを0で埋める
  - 指定したソースオペランドの下位バイトを取り出し、その値をゼロ拡張して目的のオペランドにセット
  - 小さいデータ（8ビット）を大きなレジスタ（32ビットまたは64ビット）に安全に移すために使用される


# レジスタ
- RAX
  - 算術とロジック演算の結果を保持
  - 関数の返り値を保持
  - システムコールの引数および結果を保持
  - EAX(32ビット), AX(16ビット), AL(8ビット)といった小さいレジスタの拡張版
- RSP
  - スタックの開始位置を表す
- RBP
  - 関数フレーム開始位置
  - ベースレジスタとも呼ばれ、入っている値はベースポインタと呼ばれる
- RDI
  - 第一引数用レジスタ
- RSI
  - 第二引数用レジスタ
- AL
  - RAXの下位8ビットを指す別名レジスタ
- RFLAGS(フラグレジスタ)
  - 各ビットは特定の状態フラグ
  - 整数演算や比較演算命令が実行されるたびに更新
  - キャリーフラグ（CF）：これは最も右のビットで、算術演算におけるキャリーオーバー（加算時）またはボロー（減算時）を示す
  - パリティフラグ（PF）：最後に実行された命令の結果の最下位バイト内の1の数が偶数であればセットされる
  - オーバーフローフラグ（OF）：符号付き算術演算の結果が表現できる範囲を超えた場合にセットされる
  - ゼロフラグ（ZF）：最後に実行された命令の結果がゼロである場合にセットされる
  - 符号フラグ（SF）：最後に実行された命令の結果の最上位ビット（符号ビット）が1である場合にセットされる
  - 割り込み許可フラグ（IF）：セットされている場合、外部からの割り込みが許可される

## レジスタに格納されたアドレスを使うには
メモリから値をロードするときは、`mov dst, [src]`という構文を使う。「srcレジスタの値をアドレスとみなしてそこから値をロードしdstに保存する」という意味。pushやpopは暗黙のうちにRSPをアドレスとみなしてメモリアクセスをしている。

# EBNF(Extended Backus–Naur form)
|書き方  |意味|
|-----  |----|
|A*     |Aの0回以上の繰り返し|
|A?     |Aまたはε(何もなし)|
|A\|B   |AまたはB|
|(・・・)|グループ化|