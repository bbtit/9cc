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

# Parse
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
  - 比較サイズの分だけ第1引数と第2引数を比較する。
  - 一致していれば0を、一致していなければ1または-1を返す
- `strtol`
  - 数値を読み込んだ後、第2引数のポインタをアップデートして、読み込んだ最後の文字の次の文字を指すように値を更新
  - 数値を1つ読み込んだ後、その次の文字が+や-ならば、pはその文字を指す

# アセンブリ

# レジスタ